---
layout: post
title: When cluster member gets corrupted datafiles  
category: mysql 
tags: [mariadb]
---

The issue we got in Mariadb Galera 5.5.24 : run out of space, DB log shows datafile corruption.  
```
[root@node1]# pwd
/var/lib/mysql
[root@node1]# ls -ltr
total 78698168
........
-rw-rw----. 1 mysql mysql **80406904832** Dec 21 15:32 ibdata1
```
what are this 80G for ?  they’re keystone.token,  over 5 million tokens stored in this table, most of them are expired. 
```
MariaDB [information_schema]> SELECT table_schema "DB Name",
    -> ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) "DB Size in MB"
    -> FROM information_schema.tables
    -> GROUP BY table_schema;
+--------------------+---------------+
| DB Name            | DB Size in MB |
+--------------------+---------------+
| aodh               |           0.1 |
| cinder             |           1.1 |
| glance             |           1.1 |
| gnocchi            |         359.3 |
| heat               |           1.1 |
| information_schema |           0.1 |
| keystone           |       63143.1 |
| mysql              |           0.6 |
| nova               |          46.2 |
| nova_api           |           5.6 |
| ovs_neutron        |           6.8 |
| performance_schema |           0.0 |
+--------------------+---------------+
12 rows in set (0.56 sec)

SELECT table_name AS `Table`, round(((data_length + index_length) / 1024 / 1024), 2) `Size in MB` 
FROM information_schema.TABLES 
where table_schema='keystone'
ORDER BY (data_length + index_length) DESC LIMIT 0,10; 
+----------------+------------+
| Table          | Size in MB |
+----------------+------------+
| token          |   63142.00 |
| trust          |       0.14 |
| federated_user |       0.06 |
| local_user     |       0.05 |
| endpoint       |       0.05 |
| project        |       0.05 |
| trust_role     |       0.05 |
| access_token   |       0.05 |
| password       |       0.03 |
| nonlocal_user  |       0.03 |
+----------------+------------+
10 rows in set (0.01 sec)

MariaDB [keystone]> select count(*) from token;
+----------+
| count(*) |
+----------+
|  5242581 |
+----------+
1 row in set (1.83 sec)

MariaDB [keystone]> select count(*) from token where expires < CURRENT_DATE;
+----------+
| count(*) |
+----------+
|  5236970 |
+----------+
1 row in set (2.65 sec)
```

Unfortunately, this ibdata1 cannot be shrinked, the only way is to delete this file, rebuild database and import data.  There is a parameter innodb_file_per_table=1, which tells MySQL to store each table, including its indexes, is stored as a separate file, it is be default enabled since 5.6.6. Here we have a rather old version at 5.5.24. 

Following steps are tested on 10.1 
# on node1 
Take backup1 of db   
Delete all expired tokens  
Do a mysqldump of all databases, procedures, triggers etc except the mysql and performance_schema databases  
Drop all databases except the above 2 databases  
Stop mariadb  
Delete ibdata1 and ib_log files  
Edit /etc/my.cnf.d/server.cnf to include innodb_file_per_table=1  
Start mariadb  
Restore from dump  
```
#wsrep_on = ON 

mysqldump -u root -p --databases test_db testdb2 ... > testdb.sql
delete from token where expires < CURRENT_DATE;
mysqldump -u root -p --databases test_db testdb2 ...> testdb2.sql
drop database test_db; 
drop database testdb2; 
......
rm -rf ibdata1, logfiles  
mysql -u root -p  < ./testdb2.sql
uncomment #wsrep_on=ON 
galera_new_cluster
```

# on node2/3 
Delete all databases except the mysql and performance_schema, add it into cluster, refresh data from node1
```
systemctl stop mariadb 
cd /var/lib/mysql; rm -rf * 
mysql_install_db 
systemctl start mariadb
```
DB password will be reset. If the config file galera.cnf wrong, like [mysqld] was accidently changed to gmysqld], some old values like password be used in new DB, and it won't refesh data from node1 






