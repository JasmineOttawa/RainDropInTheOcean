---
layout: post
title: setup a Dev mongoDB cluster 
category: MongoDB 
tags: [database, mongo]
---

A mongodb sharded cluster horizontally scales the data/read/write to multiple nodes. It consists of:    
**Config servers**: store metadata and configuration settings for the cluster. It must be deployed as a replica set   
**Mongos**: a query router, providing an interface between client applications and the sharded cluster    
**Shard**: each shard contains a subset of the sharded data, shards must be deployed as a replica set.    
  
Shard and config server could be configured as one-node replica set for dev.    
Here we will setup a dev cluster on 3 nodes:    
node0, IP0, configServer + mongos    
node1, IP1, shard1    
node2, IP2, shard2    

## step1 , on RHEL7, install mongod,mongos,mongo shell
```vi /etc/yum.repos.d/mongodb-org-4.0.repo  
[mongodb-org-4.0]  
name=MongoDB Repository  
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/  
gpgcheck=1  
enabled=1  
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc  
  
yum install -y mongodb-org-server-4.0.4 mongodb-org-shell-4.0.4 mongodb-org-tools-4.0.4  
yum install -y mongodb-org-mongos-4.0.4 # on node0 only   
```
  
## step2, setup configserver on node0  
```vi /etc/mongod.conf  
sharding:  
  clusterRole: configsvr   
replication:  
  replSetName: rsConfig  
net:  
  bindIp: localhost,IP0  
```  
service mongod start    
mongo  
```rs.initiate(  
  {  
    _id: "rsConfig",  
    configsvr: true,  
    members: [  
      { _id : 0, host : "IP0:27017" }  
    ]  
  }  
)  
```  
## step3, setup shards on node1/2  
```sharding:  
   clusterRole: shardsvr  
replication:  
   replSetName: rs1  
net:  
   bindIp: localhost,IP1  
```  
service mongod start   
```rs.initiate(  
  {  
    _id : "rs1",  
    members: [  
      { _id : 0, host : "IP1:27017" }  
    ]  
  }  
)  
```  
## step4, connect to mongos, add shards to cluster  
nohup mongos --configdb rsConfig/localhost:27017 --port 27030 &   
mongo --host localhost --port 27030  
```sh.addShard( "rs1/IP1:27017")  
sh.addShard( "rs2/IP2:27017")  
db.adminCommand( { listShards : 1 } )  
```  
## Q & A   
Q1 - first I tried on ubuntu 16.04, when editing mongod.conf in Mobaxterm, it changes first character to 'g'  
A - change env variable TERM from xterm to linux   
  
Q2 - when starting mongod on 2nd shard, got message  
Dec 12 20:10:08 Node2 systemd[1]: mongod.service: control process exited, code=exited status=62  
Dec 12 20:10:08 Node2 systemd[1]: Failed to start MongoDB Database Server.  
per https://github.com/mongodb/mongo/blob/mastear/src/mongo/util/exit_code.h  
EXIT_NEED_DOWNGRADE = 62, // The current binary version is not appropriate to run on the existing datafiles.  
  
I took the workaround to recreate DB files since it's new:   
ls -ld /var/lib/mongo  
cd /var/lib  
mv mongo mongo.backup  
mkdir mongo  
chown mongod.mongod mongo  
   
If there are existing data, how to fix the issue while keeping data?   
A -   
  
Q3 - How to connect to mongos remotely, mongo --host IP0 --port 27030 returns message:   
     failed: SocketException: Error connecting to IP0:27030 :: caused by :: Connection refused :  
connect@src/mongo/shell/mongo.js:257:13  
A -   
