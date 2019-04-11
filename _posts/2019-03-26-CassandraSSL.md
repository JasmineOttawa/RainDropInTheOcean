---
layout: post
title: Cassandra - enable SSL  
category: Cassandra
tags: [Cassandra]
---
# setup a single node cassandra cluster 
# extend to a 3-node cluster 
# enable Cassandra authentication 
# enable SSL on cassandra 
# a SSL error troubleshooting 

# setup a 1-node cassandra cluster 
step1, make sure java, python are there
```
[root@centos-1 ~]# java -version
openjdk version "1.8.0_201"
OpenJDK Runtime Environment (build 1.8.0_201-b09)
OpenJDK 64-Bit Server VM (build 25.201-b09, mixed mode)
[root@centos-1 ~]# python --version
Python 2.7.5

yum -y install java  # install java if not installed 
yum install libaio
```

step2, install cassandra    
option1 - community version   
  http://cassandra.apache.org/download/  
```
cat <<EOF >/etc/yum.repos.d/datastax.repo 
[cassandra]
name=Apache Cassandra
baseurl=https://www.apache.org/dist/cassandra/redhat/311x/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://www.apache.org/dist/cassandra/KEYS
EOF

yum install cassandra
# configuration 
systemctl enable cassandra  
systemctl start cassandra 
systemctl status cassandra 
```
option2 - DataStax Version   
  https://docs.datastax.com/en/install/6.7/install/installRHELdse.html    
  as of March 2019, the latest version of DataStax Enterprise 6.7 is 6.7.2.  
  This procedure installs DSE 6.7 and the DataStax Agent. It does not install OpsCenter, DataStax Studio, Graph Loader, or DataStax Bulk Loader.  
```
cat <<EOF >/etc/yum.repos.d/datastax.repo 
[datastax]
name = DataStax Repo for Apache Cassandra
baseurl = http://rpm.datastax.com/community
enabled = 1
gpgcheck = 0
EOF

yum install dse-full          # install 6.7.2 
yum install dse-full-6.7.0-1  # install an earlier 6.7 version 
systemctl dse start
nodetool status 
```
step3, configuration   
```
vi /etc/cassandra/conf/cassandra.yaml
In a settings seed_provider change - seeds: "127.0.0.1" to "seeds: "<Your VM ip address>"
Change "listen_address: localhost"  settings to "listen_address: <Your VM ip address>"
Change "start_rpc: false" setting to "start_rpc: true"
Change "rpc_address: localhost" to "rpc_address: <Your VM ip address>"
vi /etc/cassandra/default.conf/cassandra-env.sh
Uncomment following setting: # JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=<public name>" and change <pubic name> to your VM ip address.
```
step4, verify 
``` 
[root@centos-1 conf]# cqlsh 10.12.5.68
Connected to Test Cluster at 10.12.5.68:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

# extend to a 3-node cluster 
on each node:  
Add IPs of other cassandra VMs to seed_provider/ -seeds settings: -seeds: "current_VM_IP, other_VM_IP1,other_VM_IP2 ..."   
systemctl restart cassandra   
```
[root@centos-1 conf]# nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.12.5.143  189.3 KiB  256          70.8%             2f94f361-b742-420a-bd56-bea03988e57f  rack1
UN  10.12.5.68   288.25 KiB  256          67.7%             69dc231d-447e-4d33-97c1-9e9d5f9b0ee3  rack1
UN  10.12.6.39   204.15 KiB  256          61.4%             6d367cd0-5537-4d55-876f-b53b0fbf4f65  rack1
```

# enable Cassandra authentication 
```
vi /etc/cassandra/conf/cassandra.yaml 
authenticator: PasswordAuthenticator

systemctl restart cassandra 
cqlsh <node_ip> -u cassandra -p cassandra
CREATE ROLE raindrop WITH PASSWORD = 'raindrop' AND SUPERUSER = true AND LOGIN = true;
cassandra@cqlsh> ALTER ROLE cassandra WITH PASSWORD='cassandra' AND SUPERUSER=false;
Unauthorized: Error from server: code=2100 [Unauthorized] message="You aren't allowed to alter your own superuser status or that of a role granted to you"
cassandra@cqlsh> login raindrop 'raindrop';
raindrop@cqlsh>  ALTER ROLE cassandra WITH PASSWORD='cassandra' AND SUPERUSER=false;
raindrop@cqlsh> list users;

 name      | super
-----------+-------
 cassandra | False
  raindrop |  True

#connect in Java driver 
cluster = Cluster.builder()
        .addContactPoint(properties.getProperty("contactPoint"))
        .withCredentials("raindrop","raindrop")
        .build();
```

# enable SSL on cassandra 
step1, Generate a private and public key pair, all server nodes use same key 
```
[root@centos-1 cert]# keytool -genkey -keyalg RSA -alias node0 -validity 36500 -dname "cn=cassandra_cluster, ou=Kanata, o=Ottawa, c=ON" -keypass cassandra -storepass cassandra -keystore /stage/cert/keystore.node1
Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore /stage/cert/keystore.node1 -destkeystore /stage/cert/keystore.node1 -deststoretype pkcs12".
[root@centos-1 cert]# keytool -genkey -keyalg RSA -alias node0 -validity 36500 -dname "cn=cassandra_cluster, ou=Kanata, o=Ottawa, c=ON" -keypass cassandra -storepass cassandra -keystore /stage/cert/keystore.node1 -deststoretype pkcs12
keytool error: java.io.IOException: DerInputStream.getLength(): lengthTag=109, too big.
[root@centos-1 cert]# rm -rf keystore.node1
[root@centos-1 cert]# keytool -genkey -keyalg RSA -alias servernode -validity 36500 -dname "cn=cluster1, ou=cassandra, o=Ottawa, c=ON" -keypass cassandra -storepass cassandra -keystore /stage/cert2/keystore.p12 -deststoretype pkcs12
```
step2, Export the public part of the certificate to a separate file.
```
[root@centos-1 cert]# keytool -export -alias servernode -file /stage/cert2/server.cer -keystore /stage/cert2/keystore.p12 -keypass cassandra -storepass cassandra
[root@centos-1 cert]# ls -ltr
total 8
-rw-r--r-- 1 root root 2501 Apr  5 19:54 keystore.p12
-rw-r--r-- 1 root root  809 Apr  5 19:56 server.cer
```
step3, Add the node1.cer certificate to the node1 truststore of the node using the keytool -import command.
```
[root@centos-1 cert]# keytool -import -noprompt -v -trustcacerts -alias servernode  -file /stage/cert2/server.cer -keystore /stage/cert2/truststore.p12 -keypass cassandra -storepass cassandra
Certificate was added to keystore
[Storing /stage/cert/truststore.node1]
[root@centos-1 cert]# ls -ltr
total 12
-rw-r--r-- 1 root root 2501 Apr  5 19:54 keystore.p12
-rw-r--r-- 1 root root  809 Apr  5 19:56 server.cer
-rw-r--r-- 1 root root  871 Apr  5 19:57 truststore.p12
```
step4,cqlsh does not work with the certificate in the format generated. openssl is used to generate a PEM file of the certificate with no keys, node1.cer.pem, and a PEM file of the key with no certificate, node1.key.pem. 
```
# if keystore is in JKS format, convert it to PKCS12 format first 
keytool -importkeystore -srckeystore /stage/cert/keystore.node1 -destkeystore /stage/cert/node1.p12 -deststoretype PKCS12 -srcstorepass cassandra -deststorepass cassandra
# generate PEm file for cqlsh 
openssl pkcs12 -in /stage/cert2/keystore.p12 -nokeys -out /stage/cert2/server.cer.pem -passin pass:cassandra
openssl pkcs12 -in /stage/cert2/keystore.p12 -nodes -nocerts -out /stage/cert2/server.key.pem -passin pass:cassandra
```
step5. Copy generated certificates to different nodes to /stage/cert2/ folder.
``` 
scp * root@10.12.5.143:/stage/cert2
scp * root@10.12.6.39:/stage/cert2
```
step6. For node to node enryption modify server encyption options in /etc/cassandra/conf/cassandra.yaml
```
server_encryption_options:
    internode_encryption: all
    keystore: /stage/cert2/keystore.p12
    keystore_password: cassandra
    truststore: /stage/cert2/truststore.p12
    truststore_password: cassandra
   # More advanced defaults below:
   protocol: TLS
   algorithm: SunX509
   store_type: PKCS12
   cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
   require_client_auth: true
```
   
step7. For client to node enryption modify client encyption options in /etc/cassandra/conf/cassandra.yaml
```
client_encryption_options:
    enabled: true
    # If enabled and optional is set to true encrypted and unencrypted connections are handled.
    optional: false
    keystore: /opt/nfvo/mft/keystore.p12
    keystore_password: cassandra
    require_client_auth: false
    # Set trustore and truststore_password if require_client_auth is true
    truststore: /opt/nfvo/mft/truststore.p12
    truststore_password: cassandra
    # More advanced defaults below:
    # protocol: TLSv1
    # algorithm: SunX509
    # store_type: JKS
    # cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
```
step8. comment out line $JVM_OPTS -Djava.rmi.server.hostname=... inside /etc/cassandra/default.conf/cassandra-env.sh        
note that /etc/cassandra/conf, /etc/alternatives/cassandra, /etc/cassandra/default.conf/ are same: 
```
[root@centos-1 cassandra]# ls -ltr
total 4
lrwxrwxrwx. 1 root root   27 Mar 21 20:01 conf -> /etc/alternatives/cassandra
drwxr-xr-x. 3 root root 4096 Apr  8 15:53 default.conf
[root@centos-1 cassandra]# ls -ltr /etc/alternatives/cassandra
lrwxrwxrwx. 1 root root 29 Mar 21 20:01 /etc/alternatives/cassandra -> //etc/cassandra/default.conf/
```
step9. Restart cassandra service by "service cassandra restart"

step10. Check that everything is ok by command: grep SSL var/log/cassandra/logs/system.log
```
[root@centos-3 conf]# grep SSL /var/log/cassandra/system.log
INFO  [main] 2019-04-08 15:58:35,433 MessagingService.java:704 - Starting Encrypted Messaging Service on SSL port 7001

# got following message on node1 
ERROR [main] 2019-04-08 15:58:19,709 CassandraDaemon.java:749 - Fatal configuration error
......
Caused by: java.io.IOException: DerInputStream.getLength(): lengthTag=109, too big.
......
        ... 10 common frames omitted

compared config on 3 nodes, note that on centos3, it's: store_type: JKS
while on centos1|2, it's: store_type: PKCS12, change it to JKS 
```
	
step11. To run java client with ssl security copy certificates from node to your laptop folder and use these parameters:
-ea -Djavax.net.ssl.keyStore=c:\...\keystore.p12 -Djavax.net.ssl.keyStorePassword=cassandra -Djavax.net.ssl.trustStore=c:\...\truststore.p12 -Djavax.net.ssl.trustStorePassword=cassandra

step12. Your builder shoud be used with "withSSL()" parameter.
Cluster cluster = Cluster.builder()
  .addContactPoint("node_address")
  .withSSL()
  .build();
step13. To use Create cqlshrc file inside ~/.cassandra folder wit this content:
[ssl]
certfile = /stage/cert2/server.cer.pem
validate = false ;; Optional, true by default. See the paragraph below. 

cqlsh IP --ssl -u raindrop -p raindrop 
 
 
# an issue troubleshooting , keystore mismatch cause 1 node not communicating with the other two. 
```
IP1    node1
IP2    node2
IP3    node3
[root@node1 conf]# nodetool status
Datacenter: DC1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  IP2  11.56 GiB  256          100.0%            c03d2ab9-0abd-4b7e-b2a9-f9a4dc3acb3d  RAC1
UN  IP1  11.6 GiB   256          100.0%            2f548ef9-abd4-4640-8f1a-e5c448df8a91  RAC1
DN  IP3  ?          256          100.0%            c15b097a-a1f4-48f7-bd94-1ac3e65a97c8  r1
[root@node2 conf]# nodetool status
Datacenter: DC1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  IP2  11.56 GiB  256          100.0%            c03d2ab9-0abd-4b7e-b2a9-f9a4dc3acb3d  RAC1
UN  IP1  11.6 GiB   256          100.0%            2f548ef9-abd4-4640-8f1a-e5c448df8a91  RAC1
DN  IP3  ?          256          100.0%            c15b097a-a1f4-48f7-bd94-1ac3e65a97c8  r1
[root@node3 conf]# nodetool status
Datacenter: DC1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
DN  IP2  ?          256          100.0%            c03d2ab9-0abd-4b7e-b2a9-f9a4dc3acb3d  r1
DN  IP1  ?          256          100.0%            2f548ef9-abd4-4640-8f1a-e5c448df8a91  r1
UN  IP3  10.5 GiB   256          100.0%            c15b097a-a1f4-48f7-bd94-1ac3e65a97c8  RAC1


### node3 cassandra:/var/log/cassandra/system.log 
ERROR [ACCEPT-/IP3] 2019-03-30 18:41:43,095 MessagingService.java:1337 - SSL handshake error for inbound connection from 2a982ce7[SSL_NULL_WITH_NULL_NULL: Socket[addr=/IP1,port=38674,localport=7001]]
javax.net.ssl.SSLHandshakeException: Received fatal alert: certificate_unknown
        at sun.security.ssl.Alerts.getSSLException(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.Alerts.getSSLException(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.SSLSocketImpl.recvAlert(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.SSLSocketImpl.readRecord(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.SSLSocketImpl.performInitialHandshake(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.SSLSocketImpl.readDataRecord(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.AppInputStream.read(Unknown Source) ~[na:1.8.0_181]
        at sun.security.ssl.AppInputStream.read(Unknown Source) ~[na:1.8.0_181]
        at java.io.DataInputStream.readInt(Unknown Source) ~[na:1.8.0_181]
        at org.apache.cassandra.net.MessagingService$SocketThread.run(MessagingService.java:1311) ~[apache-cassandra-3.11.3.jar:3.11.3]
		
### check its SSL settings 
[root@node1 conf]# grep server_encryption_options /etc/cassandra/conf/cassandra.yaml -A20
......
server_encryption_options:
    internode_encryption: all
    keystore: /opt/ncso/var/lib/mft/keycert.p12
    keystore_password: nfvo
    truststore: /tmp/trustore.p12
    truststore_password: castore
    # More advanced defaults below:
    protocol: TLS
    algorithm: SunX509
    store_type: PKCS12
    cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
    require_client_auth: true

[root@node1 conf]# cksum /tmp/trustore.p12
2269275275 1018 /tmp/trustore.p12
[root@node2 ~]# cksum /tmp/trustore.p12
3709427013 1018 /tmp/trustore.p12
[root@node3 ~]# cksum /tmp/trustore.p12
1750882647 1018 /tmp/trustore.p12

[root@node1 conf]# cksum /opt/ncso/var/lib/mft/keycert.p12
439154860 3192 /opt/ncso/var/lib/mft/keycert.p12
[root@node2 ~]# cksum /opt/ncso/var/lib/mft/keycert.p12
439154860 3192 /opt/ncso/var/lib/mft/keycert.p12
[root@node3 ~]# cksum /opt/ncso/var/lib/mft/keycert.p12
1275902438 3192 /opt/ncso/var/lib/mft/keycert.p12

### so it is /opt/ncso/var/lib/mft/keycert.p12 mismatch on node1, replace this file and restart cassandra, perform node repair. 
### verify if SSL fine 
grep SSL /var/log/cassandra/system.log
```