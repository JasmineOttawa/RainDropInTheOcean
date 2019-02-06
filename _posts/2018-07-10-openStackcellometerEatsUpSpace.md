---
layout: post
title: ceilometer eats up all the space
category: OpenStack
tags: [OpenStack]
---

# Question 
ceilometer eats up all the space

# Investigation 
how to reduce the size of ceilometer mongodb, why is the size of /var/lib/mongodb increasing gradually?  

## suppress the ceilometer data growth 
In /etc/ceilometer/ceilometer.conf, under [database] section: 
set time_to_live = 86400 (seconds, that’s 1 day. <=0 means forever)

## check ip address of ceilometer , check database size 
```
grep mongo /etc/ceilometer/ceilometer.conf
connection=mongodb://10.3.29.117:27017,10.3.29.100:27017,10.3.29.116:27017/ceilometer
replicaSet=tripleo

mongo 10.3.29.117:27017
MongoDB shell version: 2.6.11
connecting to: 10.3.29.117:27017/test
….
tripleo:PRIMARY>
tripleo:PRIMARY> use ceilometer;
switched to db ceilometer
tripleo:PRIMARY> db.meter.dataSize();
```
## remove individual records from the database 
```
tripleo:PRIMARY> db.meter.remove({“timestamp”:{“$lt”:ISODate(“2017-01-01T19:00:40”)}})
```
## compact database in mongodb 3.0 or later 
After enough meters have been deleted to lower the database size, the next step is to run compact to de-fragment the reclaim disk space   
```
tripleo:PRIMARY> db.runCommand({ compact : ‘meter’ })
```
## drop ceilometer database 
```
- stop ceilometer 
[root@ctrl0 heat-admin]# service=$(pcs status | grep ceilometer |awk ‘{print $3}’);for i in $service;do echo $i;done
openstack-ceilometer-alarm-notifier-clone
openstack-ceilometer-api-clone
openstack-ceilometer-collector-clone
openstack-ceilometer-notification-clone
openstack-ceilometer-central-clone
[root@ctrl0 heat-admin]service=$(pcs status | grep ceilometer |awk ‘{print $3}’);for i in $service;do pcs resource disable $i;done

– drop database 
tripleo:PRIMARY> db.dropDatabase()
{ “dropped” : “ceilometer”, “ok” : 1 }
tripleo:PRIMARY>

tripleo:PRIMARY> db.stats()
{
“db” : “ceilometer”,
“collections” : 0,
“objects” : 0,
“avgObjSize” : 0,
“dataSize” : 0,
“storageSize” : 0,
“numExtents” : 0,
“indexes” : 0,
“indexSize” : 0,
“fileSize” : 0,
“dataFileVersion” : {

    },  
    "ok" : 1
```  
Now that the database is empty check /var/lib/mongodb/ to ensure the database files are gone.  
[root@ctrl0 heat-admin]ls -lh /var/lib/mongodb/ | grep ceilometer  
  
Now restart ceilometer  
[root@ctrl0 heat-admin]service=$(pcs status | grep ceilometer |awk ‘{print $3}’);for i in $service;do pcs resource enable $i;done\  
  
Check Db stats, and datafiles, meter data will begin to flow back into the database.  
  
refer to: https://access.redhat.com/solutions/2215701 