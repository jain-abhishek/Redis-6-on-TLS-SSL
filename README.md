# Redis6 on TLS/SSL

A.	Install Redis

B.	Generate keys and certificates
  1. Install openssl on your machine. 
  2. Generate private key and certificate of CA (Certificate Authority)
  3. Generate private key and a certificate signing request of redis server
  4. Create keystore and import redis server certificate 
  5. Create truststore and import CA certificate
  
C.	Configure redis.conf and start redis server

D.	Start redis-cli with certificates 

## Detailed Steps:


### A.	Install Redis:
```
wget http://download.redis.io/releases/redis-6.0.3.tar.gz

tar xzf redis-6.0.3.tar.gz

cd redis-6.0.3
```
Necessary commands to build Redis with SSL support:
```
make BUILD_TLS=yes

make install BUILD_TLS=yes

export IP_ADDRESS=<Your IP>
  
export DNS_ADDRESS=ip-<Your IP replacing . with - >
```
  
### B.	Generate keys and certificates

1.	*Generate CA private key* 
```  
openssl genrsa -out rootCA.key 2048
```
&nbsp;&nbsp;&nbsp;&nbsp;rsa: an asymmetric cryptography algo uses a private and a public key to encryption and decryption 
  
&nbsp;&nbsp;&nbsp;&nbsp;2048: length of certificate, other is 4096 which is slower and increases CPU consumption
  
&nbsp;&nbsp;&nbsp;&nbsp;We can use aes256 which is a secure symmetric encryption and uses a cipher. 
```
openssl genrsa -out rootCA.key –aes256
```  
  
![](https://github.com/jain-abhishek/images/blob/main/1.jpg)

2.	*Generate CA certificate*
```
openssl req -x509 -new -key rootCA.key -sha256 -days 365 -out rootCA.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;req: to generate PKCS10  format certificate

&nbsp;&nbsp;&nbsp;&nbsp;x509: to create a self signed certificate rather than a certificate which requires signing

![](https://github.com/jain-abhishek/images/blob/main/2.JPG)

&nbsp;&nbsp;&nbsp;&nbsp;Filling some dummy values here. 
&nbsp;&nbsp;&nbsp;&nbsp;Here CN is important, it must be address of the server you want to access, otherwise you will see errors: java.security.cert.CertificateException: No name matching <host> found. For us, CN is important for server certificate. 

3.	*Check if certificate is having all the necessary information which is provided in the previous step*
```
openssl x509 -in rootCA.crt -noout –text
```
![](https://github.com/jain-abhishek/images/blob/main/3.JPG) 
  
4.	*Generate server private key*
```  
openssl genrsa -out redisServer.key -aes256
```
  
5.	*Create certificate signing request*
```  
openssl req -new -sha256 -key redisServer.key -subj "/C=IN/ST=UP/L=Noida/O=OldIndianStreets/OU=IT/CN=${DNS_ADDRESS}" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}")) -out redisServer.csr
```  
&nbsp;&nbsp;&nbsp;&nbsp;Here subject and subject alt name are important to provide, otherwise you may see errors like: No subject alternative names matching IP address. Or: javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names present.
  
&nbsp;&nbsp;&nbsp;&nbsp;Other important point is that CN of CA and redis server must be different, otherwise you may see related errors.
  
6.	*Sign server’s certificate using CA private key and certificate*
```  
openssl x509 -req -in redisServer.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out redisServer.crt -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}") -days 365 -sha256 -extensions SAN
```
&nbsp;&nbsp;&nbsp;&nbsp;CAcreateserial: while signing CA generates a unique serial number for each certificate
                                                                                                                              
&nbsp;&nbsp;&nbsp;&nbsp;extension: to include subject alternative name

7.	*Check signed certificate has information about issuer, subject and san*
```
openssl x509 -in redisServer.crt -noout -text
```
![](https://github.com/jain-abhishek/images/blob/main/4.JPG)
                                                                                                                              
8.	*Create keystore by including redis server private key and certificate*
```                                                                                                                              
openssl pkcs12 -export -in redisServer.crt -inkey redisServer.key -out redisServerKeystore.p12 -CAfile rootCA.crt 
```
&nbsp;&nbsp;&nbsp;&nbsp;pkcs12: standard of keystore

9.	*Check if certificates exist in keystore*
```
openssl pkcs12 -info -in redisServerKeystore.p12
```
![](https://github.com/jain-abhishek/images/blob/main/5.JPG)
                                                                                                                              
10.	*Create truststore* 
```
keytool -import -alias redisServerCA -trustcacerts -file rootCA.crt -keystore cacerts
```
                                                                                                                              
11.	*List all the certificates of truststore*
```                                                                                                                              
keytool -list -keystore cacerts
```
![](https://github.com/jain-abhishek/images/blob/main/6.JPG)
                                                                                                                              
### C.	Configure redis.conf and start redis server

*Update redis configuration file with TLS properties*
```
tls-cert-file /path/to/redisServer.crt
tls-key-file /path/to/redisServer.key
tls-ca-cert-file /path/to/rootCA.crt
port 0
tls-port 6379
```
                                                                                                                              
*Start redis server*
```
./redis-server redis.conf
```
                                                                                                                              
### D.	Start redis-cli with certificates by passing CA and server’s private key, certificate
```
./redis-cli --tls --cert redisServer.crt --key redisServer.key  --cacert rootCA.crt -a
```

Please refer following URLs for more information:
                                                                                                                              
https://en.wikipedia.org/wiki/PKCS
                                                                                                                              
https://wiki.openssl.org/index.php/Command_Line_Utilities
                                                                                                                              
https://redis.io/topics/encryption
                                                                                                                              


