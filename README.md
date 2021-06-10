# Redis6 on TLS/SSL

**The article describes how to configure TLS/SSL for Redis and test using Redisson client.**
  
A.	Install Redis

B.	Generate keypairs:
  * Install openssl on your machine
  * Generate CA key pair
  * Generate Server private key and csr (certificate signing request)
  * Generate Server certificate (by signing with CA keypair)
  * Create Client private key and csr
  * Generate Client certificate (by signing with the same CA keypair)
  * Create KeyStore and import Client keypair 
  * Create TrustStore and import CA certificate
  
C.	Configure redis.conf and start redis-server

D.	Start redis-cli in tls mode

E. 	Access Server using Redisson

## Detailed Steps:


### A.	Install Redis:
```
wget http://download.redis.io/releases/redis-6.0.3.tar.gz

tar xzf redis-6.0.3.tar.gz

cd redis-6.0.3
```
Necessary commands to build Redis with TLS support:
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
&nbsp;&nbsp;&nbsp;&nbsp;rsa: an asymmetric cryptography algo uses a keypair to encryption and decryption 
  
&nbsp;&nbsp;&nbsp;&nbsp;2048: length of certificate, other is 4096 which is slower and increases CPU consumption
  
&nbsp;&nbsp;&nbsp;&nbsp;aes256: a secure symmetric encryption which uses a cipher. So after creating the private key, we are encrypting it with a cipher.

![](https://github.com/jain-abhishek/images/blob/main/1.JPG)
    
2.	*Generate CA certificate*
```
openssl req -x509 -new -key rootCA.key -sha256 -days 365 -out rootCA.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;req: to generate PKCS10  format certificate

&nbsp;&nbsp;&nbsp;&nbsp;x509: to create a self signed certificate rather than a certificate which requires signing

![](https://github.com/jain-abhishek/images/blob/main/2.JPG)

&nbsp;&nbsp;&nbsp;&nbsp;Here CN (Common Name) is important, it must be address/FQDN of the server you want to access, otherwise you will see such errors: java.security.cert.CertificateException: No name matching <host> found.  

3.	*Check if certificate is having all the necessary information which is provided in the previous step*
```
openssl x509 -in rootCA.crt -noout –text
```
![](https://github.com/jain-abhishek/images/blob/main/3.JPG)
  
4.	*Generate server private key*
```  
openssl genrsa -out redisServer.key -aes256 2048
```
  
5.	*Create Server CSR (certificate signing request)*
```  
openssl req -new -sha256 -key redisServer.key -subj "/C=IN/O=OldIndianStreets/OU=RedisServer/CN=${DNS_ADDRESS}" -out redisServer.csr
```
&nbsp;&nbsp;&nbsp;&nbsp;Here subject is important to provide
  
&nbsp;&nbsp;&nbsp;&nbsp;Other important point is that CN of CA and Server must be different (which can bot be same in prod obviously), otherwise you may see such errors:
java.security.cert.CertificateException: No name matching <hostname> found
javax.net.ssl.SSLHandshakeException: No name matching <hostname> found

Important: set Server CN as hostname/FQDN as the certificate is valid only if the request hostname matches the certificate common name.
	
```
openssl req -in redisServer.csr -noout -subject -verify
```  
![](https://github.com/jain-abhishek/images/blob/main/4.JPG)
 
6.	*Sign server certificate using CA keypair*
```  
openssl x509 -req -in redisServer.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out redisServer.crt -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}\nextendedKeyUsage=serverAuth") -days 365 -sha256 -ext SAN -extensions SAN
```
&nbsp;&nbsp;&nbsp;&nbsp;CAcreateserial: while signing, CA generates a unique serial number for each certificate
                                                                                                                              
&nbsp;&nbsp;&nbsp;&nbsp;SAN: subject alternative name tells that all subdomains are secured by the same SSL certificate

Here SAN are important to provide, otherwise you may see errors like: No subject alternative names matching IP address. Or: javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names present.                                                            

&nbsp;&nbsp;&nbsp;&nbsp;extension: to include SAN

&nbsp;&nbsp;&nbsp;&nbsp;extendedKeyUsage: to specify the usage of certificate ie: for server authentication or client authentication or both. Its mandatory parameter to set in order to generate separate certificates for Server and Client. Here the value is set as serverAuth, so the certificate is created for server authentication only.


7.	*Verify that the signed Server certificate has information of issuer, subject and san*
```
openssl x509 -in redisServer.crt -noout -text -purpose                                                                                                                           openssl verify -CAfile rootCA.crt redisServer.crt
```
&nbsp;&nbsp;&nbsp;&nbsp;purpose: tells about extended key usage                                                                                                       
                                                                                                                              
![](https://github.com/jain-abhishek/images/blob/main/5.JPG)
![](https://github.com/jain-abhishek/images/blob/main/6.JPG)
![](https://github.com/jain-abhishek/images/blob/main/7.JPG)
    
8.	*Create Client keypair*

Important: The Client CSR needs to be signed by the same CA which is used to sign Server CSR. (I do'nt find a way to upload a different CA certificate (which is used to sign Client certificate) at Redis Server side. Else you may see such error: Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target.
															      
```                                                                                                                              
openssl genrsa -out redisClient.key -aes256 2048
openssl req -new -sha256 -key redisClient.key -subj "/C=IN/O=OldIndianStreets/OU=Client/CN=${DNS_ADDRESS}" -out redisClient.csr
openssl x509 -req -in redisClient.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out redisClient.crt -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:${DNS_ADDRESS}\nextendedKeyUsage=clientAuth") -days 365 -sha256 -ext SAN -extensions SAN
```

&nbsp;&nbsp;&nbsp;&nbsp;Here extendedKeyUsage is set as clientAuth, so the certificate can be used for client authentication only.

9. *Verify that the signed Client certificate has information of issuer, subject and san*
```
openssl x509 -in redisClient.crt -noout -text -purpose
``` 
![](https://github.com/jain-abhishek/images/blob/main/8.JPG)
![](https://github.com/jain-abhishek/images/blob/main/9.JPG)
![](https://github.com/jain-abhishek/images/blob/main/10.JPG)
 
10.	*Create keystore by including Client keypair*

We will use the keystore in Redission Client code in order to get authenticated by the Server 
 
```                                                                                                                              
openssl pkcs12 -export -in redisClient.crt -inkey redisClient.key -out redisClientKeystore.p12 -CAfile rootCA.crt
```
 
&nbsp;&nbsp;&nbsp;&nbsp;pkcs12: standard of keystore

 
11.	*Check if keypair exist in keystore*
```
openssl pkcs12 -info -in redisClientKeystore.p12
```
![](https://github.com/jain-abhishek/images/blob/main/11.JPG)
![](https://github.com/jain-abhishek/images/blob/main/12.JPG) 
 
12.	*Create truststore* 

The truststore includes CA certificate by which Client certificate is signed. We will use it in Redisson client code.
 
```
keytool -import -alias redisCA -trustcacerts -file rootCA.crt -keystore cacerts
```
 
![](https://github.com/jain-abhishek/images/blob/main/13.JPG)                                                                                                                              
13.	*List all the certificates of truststore*
```                                                                                                                              
keytool -list -keystore cacerts
```
![](https://github.com/jain-abhishek/images/blob/main/14.JPG)

14. To verify if the private key matches the certificate by checking their modulus
```                                                                                                                              
openssl x509 -noout -modulus -in redisServer.key | openssl sha256
output: (stdin)= b2525f5a95e24dc348774a1be910fe93aa9298957cfb1253aa91c7bc2e319fda
openssl rsa  -noout -modulus -in redisServer.crt | openssl sha256
output: (stdin)= b2525f5a95e24dc348774a1be910fe93aa9298957cfb1253aa91c7bc2e319fda

openssl x509 -noout -modulus -in redisClient.key | openssl sha256
openssl x509 -noout -modulus -in redisClient.crt | openssl sha256
```	
If both the values are same, keypair are related to each other.
	
### C.	Configure redis.conf and start redis server
 
*Update redis configuration file with TLS properties*
```
tls-cert-file /path/to/redisServer.crt
tls-key-file /path/to/redisServer.key
tls-ca-cert-file /path/to/rootCA.crt
port 0
tls-port 6379
tls-auth-clients yes
```

Setting port 0 stops all non-tls communications. 
 
*Start redis server*
```
./redis-server redis.conf
```
 
![](https://github.com/jain-abhishek/images/blob/main/15.JPG)
 
### D.	Start redis-cli with certificates by passing CA certificate and Client keypair
```
./redis-cli --tls --cert redisClient.crt --key redisClient.key  --cacert rootCA.crt 
```
 
![](https://github.com/jain-abhishek/images/blob/main/16.JPG)
 

### E. Access Server using Redisson
	
1. Copy keystore and truststore at Client location
	
2. Add maven dependency
``` 
<dependencies>
	<dependency>
  		<groupId>org.redisson</groupId>
  		<artifactId>redisson</artifactId>
  		<version>3.13.1</version>
 	</dependency>
</dependencies>
``` 
	
3. 
```
import java.net.MalformedURLException;
import java.net.URI;
import org.redisson.Redisson;
import org.redisson.api.RBucket;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.redisson.config.SingleServerConfig;
import org.redisson.config.TransportMode;

public class RedissonOnTLS {
 	public static void main(String[] args) throws MalformedURLException {
  		Config config = new Config();
		config.setTransportMode(TransportMode.NIO);
		
		SingleServerConfig serverConfig = config.useSingleServer();
		serverConfig.setAddress("rediss://ec2-3-143-214-20.us-east-2.compute.amazonaws.com:6379");
		serverConfig.setSslKeystore(URI.create("file:/C:/workspace/Redis/redisClientKeystore.p12").toURL());
		serverConfig.setSslKeystorePassword("passwd");  // Used passwd as keystore, truststore and user password
		serverConfig.setSslTruststore(URI.create("file:/C:/workspace/Redis/cacerts").toURL());
		serverConfig.setSslTruststorePassword("passwd");
		serverConfig.setPassword("passwd");
		
		RedissonClient client = Redisson.create(config);
		RBucket<String> bucket = client.getBucket("key");
		bucket.set("value");
		
		bucket = client.getBucket("key");
		System.out.println(bucket.get());
		
		client.shutdown();
	}
}
```
 
![](https://github.com/jain-abhishek/images/blob/main/17.JPG)
 
 
Please refer following URLs for more information:
                                                                                                                              
https://en.wikipedia.org/wiki/PKCS
                                                                                                                              
https://wiki.openssl.org/index.php/Command_Line_Utilities
                                                                                                                              
https://redis.io/topics/encryption

https://www.digicert.com/faq/subject-alternative-name.htm 
 
https://redisson.org/redis-java-client-with-code-example.html
                                                 
