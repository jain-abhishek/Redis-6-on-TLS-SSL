# Redis6 on TLS/SSL

A.	Install Redis

B.	Generate keypairs:
  * Install openssl on your machine
  * Generate CA key pair
  * Generate Server private key and csr (certificate signing request)
  * Generate Server certificate (by signing with CA keypair)
  * Create Client private key and csr
  * Generate Client certificate (by signing with CA keypair)
  * Create KeyStore and import Client keypair 
  * Create TrustStore and import CA certificate
  
C.	Configure redis.conf and start redis-server

D.	Start redis-cli in tls mode

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
openssl genrsa -out rootCA.key –aes256 2048
```
&nbsp;&nbsp;&nbsp;&nbsp;rsa: an asymmetric cryptography algo uses a private and a public key to encryption and decryption 
  
&nbsp;&nbsp;&nbsp;&nbsp;2048: length of certificate, other is 4096 which is slower and increases CPU consumption
  
&nbsp;&nbsp;&nbsp;&nbsp;aes256: a secure symmetric encryption which uses a cipher

![](https://github.com/jain-abhishek/images/blob/main/1.jpg)

2.	*Generate CA certificate*
```
openssl req -x509 -new -key rootCA.key -sha256 -days 365 -out rootCA.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;req: to generate PKCS10  format certificate

&nbsp;&nbsp;&nbsp;&nbsp;x509: to create a self signed certificate rather than a certificate which requires signing

![](https://github.com/jain-abhishek/images/blob/main/2.JPG)

&nbsp;&nbsp;&nbsp;&nbsp;Here CN is important, it must be address/FQDN of the server you want to access, otherwise you will see errors: java.security.cert.CertificateException: No name matching <host> found.  

3.	*Check if certificate is having all the necessary information which is provided in the previous step*
```
openssl x509 -in rootCA.crt -noout –text
```
![](https://github.com/jain-abhishek/images/blob/main/3.JPG) 
  
4.	*Generate server private key*
```  
openssl genrsa -out redisServer.key -aes256 2048
```
  
5.	*Create certificate signing request*
```  
openssl req -new -sha256 -key redisServer.key -subj "/C=IN/O=OldIndianStreets/OU=IT/CN=${DNS_ADDRESS}" -out redisServer.csr
openssl req -in redisServer.csr -noout -subject -verify
```  
&nbsp;&nbsp;&nbsp;&nbsp;Here subject is important to provide
  
&nbsp;&nbsp;&nbsp;&nbsp;Other important point is that CN of CA and Server must be different, otherwise you may see related errors.
  
6.	*Sign server certificate using CA private key and certificate*
```  
openssl x509 -req -in redisServer.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out redisServer.crt -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}\nextendedKeyUsage=serverAuth") -days 365 -sha256 -ext SAN -extensions SAN
```
&nbsp;&nbsp;&nbsp;&nbsp;CAcreateserial: while signing CA generates a unique serial number for each certificate
                                                                                                                              
&nbsp;&nbsp;&nbsp;&nbsp;extension: to include subject alternative name                                                        

&nbsp;&nbsp;&nbsp;&nbsp;extendedKeyUsage: to specify the usage of certificate ie: for server authentication or client authentication or both. Here the value is set as serverAuth, so the certificate is created for server authentication only.

Here subject alt name are important to provide, otherwise you may see errors like: No subject alternative names matching IP address. Or: javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names present.                                                            

7.	*Check signed certificate has information about issuer, subject and san*
```
openssl x509 -in redisServer.crt -noout -text -purpose                                                                                                                           openssl verify -CAfile rootCA.crt redisServer.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;purpose: tells about extended key usage                                                                                                       
                                                                                                                              
![](https://github.com/jain-abhishek/images/blob/main/4.JPG)
![](https://github.com/jain-abhishek/images/blob/main/7.JPG)
    
8.	*Create Client keypair*
```                                                                                                                              
openssl genrsa -out redisClient.key -aes256 2048
openssl req -new -sha256 -key redisClient.key -subj "/C=IN/O=OldIndianStreets/OU=Client/CN=${DNS_ADDRESS}" -out redisClient.csr
openssl x509 -req -in redisClient.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out redisClient.crt -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}\nextendedKeyUsage=clientAuth") -days 365 -sha256 -ext SAN -extensions SAN
```

&nbsp;&nbsp;&nbsp;&nbsp;Here extendedKeyUsage is set as clientAuth, so the certificate can be used for client authentication only.

9. 
```
openssl x509 -in redisClient.crt -noout -text -purpose
``` 
![](https://github.com/jain-abhishek/images/blob/main/8.JPG)
![](https://github.com/jain-abhishek/images/blob/main/9.JPG)
 
10.	*Create keystore by including Client keypair*
```                                                                                                                              
openssl pkcs12 -export -in redisClient.crt -inkey redisClient.key -out redisClientKeystore.p12 -CAfile rootCA.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;pkcs12: standard of keystore

11.	*Check if certificates exist in keystore*
```
openssl pkcs12 -info -in redisClientKeystore.p12
```
![](https://github.com/jain-abhishek/images/blob/main/5.JPG)
                                                                                                                              
12.	*Create truststore* 
```
keytool -import -alias redisCA -trustcacerts -file rootCA.crt -keystore cacerts
```
                                                                                                                              
13.	*List all the certificates of truststore*
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
                                                                                                                              


