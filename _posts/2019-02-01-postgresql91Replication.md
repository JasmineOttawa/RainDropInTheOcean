---
layout: post
title: Postgresql hot standby failover 
category: Postgres
tags: [Postgres]
---

There are 2 nodes running postgres9.1 on RHEL6.5. 
2 tickets were opened recently, one if the role of master and slave exchanges, the other one is that both nodes become writable standalone. 

Theoratically: 


Actually, I build a lab to test the failover scenario 
### step1, build 2 standalone database   
```
yum install libedit-20090923-3.0_1.el6.x86_64.rpm --nogpgcheck   
yum install postgres-9.1.1-1.x86_64.openscg.rpm --nogpgcheck  
export PGHOME="/opt/postgres/9.1"
[root@standby stage]# su - postgres -c "LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH $PGHOME/bin/initdb -D /opt/postgres/9.1/data -U postgres"
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale en_US.UTF-8.
The default database encoding has accordingly been set to UTF8.
The default text search configuration will be set to "english".

fixing permissions on existing directory /opt/postgres/9.1/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 32MB
creating configuration files ... ok
creating template1 database in /opt/postgres/9.1/data/base/1 ... ok
initializing pg_authid ... ok
initializing dependencies ... ok
creating system views ... ok
loading system objects' descriptions ... ok
creating collations ... ok
creating conversions ... ok
creating dictionaries ... ok
setting privileges on built-in objects ... ok
creating information schema ... ok
loading PL/pgSQL server-side language ... ok
vacuuming database template1 ... ok
copying template1 to template0 ... ok
copying template1 to postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the -A option the
next time you run initdb.

Success. You can now start the database server using:

    /opt/postgres/9.1/bin/postgres -D /opt/postgres/9.1/data
or
    /opt/postgres/9.1/bin/pg_ctl -D /opt/postgres/9.1/data -l logfile start
```

setup config file 
```
[postgres@standby]$ cat pg91.env
#!/bin/bash
export PGHOME="/opt/postgres/9.1"
export PGDATA=$PGHOME/data
export PATH=$PGHOME/bin:$PATH
export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
export PGUSER=postgres
export PGDATABASE=postgres
export PGPORT=5432
```

### step2, setup replication 
2.1) on master node 
edit pg_hba.conf, add a rule that will allow the database user from the standby to access the primary.    
```
host    replication     postgres         0.0.0.0/0    trust
host    replication     postgres         ::/0         trust
```

edit postgresql.conf 
```
Section	                        Default setting	                Modified setting
CONNECTIONS AND AUTHENTICATION	#listen_addresses = 'localhost'	listen_addresses = '*'
WRITE AHEAD LOG	                #wal_level = minimal	          wal_level = 'hot_standby'
WRITE AHEAD LOG	                #archive_mode = off	            archive_mode = on
WRITE AHEAD LOG	                #archive_command = ''	          archive_command = 'true'
REPLICATION	                    #max_wal_senders = 0	          max_wal_senders = 3
REPLICATION	                    #hot_standby = off	            hot_standby = on
#grep -E 'listen|wal|archive|standby' postgresql.conf 
archive_command = 'cp %p /opt/postgres/9.1/archive/%f'  # command to use to archive a logfile segment
```

2.2) on standby node 
```
#remove everything including config files 
rm -rf /opt/postgres/9.1/data   
#make a consistent copy of the primary to standby
[postgres@standby]$ /opt/postgres/9.1/bin/pg_basebackup -h $MASTER_IP -U postgres -D  /opt/postgres/9.1/data
WARNING:  skipping special file "/opt/postgres/9.1/data/pg_tblspc/16385"
pg_basebackup: directory "/opt/postgres/9.1/data" exists but is not empty

#edit pg_hba.conf, change the IP address for the repuser to connect from need to be the master server's address in 

#edit postgresql.conf,  same as master. This way both systems can take the role of master or standby regarding these configuration files.

#vi recovery.conf. If this file is present, the server will enter recovery mode on startup
#trigger file can be created on the event of primary crash, it will trigger failover on the standby, meaning the database starts to accept write operations as well.
standby_mode = 'on'
primary_conninfo = 'host=$MASTER_IP port=5432 user=postgres password=POSTGRES_PASSWORD'
trigger_file= '/opt/postgres/9.1/data/trigger_file'

#restart DB 
/opt/postgres/9.1/bin/pg_ctl -D /opt/postgres/9.1/data -l /opt/postgres/9.1/logs/postgres91.log stop 
/opt/postgres/9.1/bin/pg_ctl -D /opt/postgres/9.1/data -l /opt/postgres/9.1/logs/postgres91.log start
```
when query status on standby, it shows:  
```
psql postgres postgres <<_MAC
SELECT pg_is_in_recovery();
_MAC
psql.bin: FATAL:  the database system is starting up
```

edit postgresql.conf on standby 
```
hot_standby = on
commented out the archive_mode=on
commented out archive command; and
# restart standby 
/opt/postgres/9.1/bin/pg_ctl -D /opt/postgres/9.1/data -l /opt/postgres/9.1/logs/postgres91.log stop 
/opt/postgres/9.1/bin/pg_ctl -D /opt/postgres/9.1/data -l /opt/postgres/9.1/logs/postgres91.log start
# now it works 
[postgres@kll2023 data]$ psql postgres postgres <<_MAC
> SELECT pg_is_in_recovery();
> _MAC
Password for user postgres:
 pg_is_in_recovery
-------------------
 t
(1 row)
```

2.3) test replication 
create a table on master, it shows up on slave 
```
postgres=# \dt
No relations found.
postgres=# CREATE TABLE cities (
postgres(#     name            varchar(80),
postgres(#     location        point
postgres(# );
CREATE TABLE
postgres=# \dt
         List of relations
 Schema |  Name  | Type  |  Owner
--------+--------+-------+----------
 public | cities | table | postgres
(1 row)
```
### step3, test failover 
3.1) restart standby , slave remains slave, master remains master.  
3.2) restart master, slave remains slave, master remains master.  
3.3) create trigger file, 2 nodes become writable standalone DBs 
```
[postgres@standby]$ cat recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=192.168.153.47 port=5432 user=postgres password=postgres'
trigger_file= '/opt/postgres/9.1/data/trigger_file'
[postgres@standby]$ touch /opt/postgres/9.1/data/trigger_file
[postgres@standby]$

sender process disappears on master, and receiver process disppears on slave. 
following query returns f on both nodes, means both nodes are not in recovery mode, they can both accept write traffic. 
psql postgres postgres <<_MAC
 SELECT pg_is_in_recovery();
_MAC
```

### Q & A 
#### troubleshooting replication - out-of sync 
https://info.crunchydata.com/blog/wheres-my-replica-troubleshooting-streaming-replication-synchronization-in-postgresql  
replication slots since 9.4 

#### postgres9.1 document 
https://www.postgresql.org/docs/9.1/runtime-config-logging.html#RUNTIME-CONFIG-LOGGING-WHEN  

#### there is following error message in slave log, does not affect replication though 
could not open temporary statistics file "pg_stat_tmp/pgstat.tmp": No such file or directory  

#### In which situation will the roll of master & slave changes?   

#### when putty session freeze
ctrl + Q 
#### print out lines that do not start with # or ; 
grep "^[^#;]" postgresql.conf   
The first ^ refers to the beginning of the line, so lines with comments starting after the first character will not be excluded.  [^#;] means any character which is not # or ;.
















