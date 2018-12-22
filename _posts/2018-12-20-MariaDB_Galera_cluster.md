---
layout: post
title: setup a mariadb galera cluster 
category: mysql 
tags: [mariadb]
---

MariaDB Galera Cluster is MariaDB plus the MySQL-wsrep.   
In MariaDB 5.5 and MariaDB 10.0, MariaDB Galera Server is a separate package installed instead of the standard server Package. Since MariaDB 10.1, the MariaDB and MariaDB Galera Server packages have been combined, Galera packages and their dependencies get installed automatically when installing MariaDB. The Galera parts remain dormant until configured.     
  
Following are the steps to install MariaDB Galera 10.1 on ubuntu 16.04      

## step1, on 3 hosts, install MariaDB Galera   
As of Ubuntu 16.04 "Xenial", The id of the new signing key is 0xF1656F24C74CD1D8 and the full key fingerprint is: 177F 4010 FE56 CA33 3630  0305 F165 6F24 C74C D1D8. Import the key: 
```
apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
`````
Add MariaDB to your sources.list   
```
add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://ftp.utexas.edu/mariadb/repo/10.1/ubuntu xenial main'
apt-get update -y
apt-get install mariadb-server rsync -y
```  
  
## step2, set mariadb cluster on node1/2/3   
vi /etc/mysql/conf.d/galera.cnf
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="galera_cluster"
wsrep_cluster_address="gcomm://IP1,IP2,IP3"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="IP1"
wsrep_node_name="Node1"  
```  

stop mariadb service on 3 nodes : systemctl stop mariadb    
On node1: 
```
galera_new_cluster
```

on node2/3 
```
systemctl start mysql
```

## step3, verify replication status  
```
mysql -u root -p -e "show status like 'wsrep_cluster_size'"
Enter password:
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
show status like 'wsrep_cluster_status';
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+
1 row in set (0.00 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

MariaDB [(none)]> create database test_db;   
```

## Q & A   
### Q1 - the configuration file is galera.cnf, when I accidently named it as galera.conf, I got following message: 
```
[Note] InnoDB:  Percona XtraDB (http://www.percona.com) 5.6.41-84.1 started; log sequence numb
[Note] InnoDB: Dumping buffer pool(s) not yet started
[Note] Plugin 'FEEDBACK' is disabled.
[Note] Server socket created on IP: '127.0.0.1'.
[Note] Reading of all Master_info entries succeded
[Note] Added new Master_info '' to hash table
[Note] /usr/sbin/mysqld: ready for connections.
mysqld[25633]: Version: '10.1.37-MariaDB-1~xenial'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  mariadb.org binary distribution
mysqld[25664]: Checking for corrupt, not cleanly closed and upgrade needing tables.
```
  
### Q2 - Start mariadb galera cluster 5.5.24 on RHEL7
5.5 is a rather old version, following command is not available 
```
[node1]# galera_new_cluster
-bash: galera_new_cluster: command not found
```
systemd doesn't provide a way to pass command-line arguments to the unit file 
```
[node1]# systemctl start mariadb --wsrep-new-cluster
systemctl: unrecognized option '--wsrep-new-cluster'
```

An alternative way to bootstrap 5.5 MariaDB galera cluster:   
Let following command run in front 
```
[node1]# /usr/bin/mysqld_safe --wsrep-new-cluster
181220 21:09:11 mysqld_safe Logging to '/var/log/mariadb/mariadb.log'.
/usr/bin/mysqld_safe: line 728: ulimit: -1: invalid option
ulimit: usage: ulimit [-SHacdefilmnpqrstuvx] [limit]
181220 21:09:11 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
181220 21:09:11 mysqld_safe WSREP: Running position recovery with --log_error='/var/lib/mysql/wsrep_recovery.zyHnbr' --pid-file='/var/lib/mysql/env1rhosp10-controller-0.localdomain-recover.pid'
181220 21:09:13 mysqld_safe WSREP: Recovered position 2d2f74fc-37f0-11e8-a76e-23af4b6dc863:293636335
```
On other nodes   
```
systemctl start mariadb
```

After the cluster has been fully formed, Go back to first node, stop the mariadb on the first node by sending it a SIGQUIT (press CTRL + \ on the console). Start it again by 'systemctl start mariadb'

### Q3 - when adding node2 to a 5.5 cluster, got following message: 
```
[Node2]# systemctl start mariadb
Job for mariadb.service failed because the control process exited with error code. See "systemctl status mariadb.service" and "journalctl -xe" for details.
[Node2]# journalctl -u mariadb
mysqld_safe[887836]: rm: cannot remove â/var/lib/mysql/mysql.sockâ: Permission denied
mysqld_safe[887836]: mktemp: failed to create file via template â/var/lib/mysql/wsrep_recovery.XXXXXXâ: Permission denied
mysqld_safe[887836]: 181220 17:35:39 mysqld_safe WSREP: mktemp failed
```

After removing /var/lib/mysql/mysql.sock, /var/log/mariadb/mariadb.log shows: 
```
181220 21:57:34 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
181220 21:57:34 mysqld_safe WSREP: mktemp failed
```
### Q4 - when adding node3 to a 5.5 cluster, got following message: 
```
181220 21:37:38 [ERROR] mysqld got signal 6 ;
This could be because you hit a bug. It is also possible that this binary
or one of the libraries it was linked against is corrupt, improperly built,
or misconfigured. This error can also be caused by malfunctioning hardware.
```

A workaround for Q3/Q4 is to re-initialize the other nodes with empty datafile, then refresh from node0. 

