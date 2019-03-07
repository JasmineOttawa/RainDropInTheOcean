---
layout: post
title: All About PDBs  
category: Oracle 
tags: [Oracle]
---

# 1, create a PDB 
There are multiple ways to create a PDB. 
copy:  copy from seed; clone locally from another PDB; clone remotely from a PDB/non-CDB 
plug in: plug in an unplugged PDB; plug in a non-CDB as a PDB 

I will try 3 ways of the 6 above: 
1.1) copy from seed   
alter pluggable database salespdb close;
drop pluggable database salespdb including datafiles;
CREATE PLUGGABLE DATABASE salespdb ADMIN USER salesadm IDENTIFIED BY salesadm
  STORAGE (MAXSIZE 5G)
  DEFAULT TABLESPACE sales 
    DATAFILE '/oracle/app/oracle/oradata/orcl/salespdb/sales01.dbf' SIZE 250M AUTOEXTEND ON
  PATH_PREFIX = '/oracle/app/oracle/oradata/orcl/salespdb'
  FILE_NAME_CONVERT = ('/oracle/app/oracle/oradata/orcl/pdbseed', '/oracle/app/oracle/oradata/orcl/salespdb');
  
alter pluggable database salespdb open;   # it is in mounted status after created; after open read-write, the default tablespace datafile is shown up 

1.2) clone from a local PDB   
CREATE PLUGGABLE DATABASE salespdb2 FROM salespdb 
  PATH_PREFIX = '/oracle/app/oracle/oradata/orcl/salespdb2'
  FILE_NAME_CONVERT = ('/oracle/app/oracle/oradata/orcl/salespdb', '/oracle/app/oracle/oradata/orcl/salespdb2')
  SERVICE_NAME_CONVERT = ('salespdb','salespdb2svc')
  NOLOGGING;
SQL> select name, pdb from v$services;         # note that service name is created by default, not as per service_name_convert, which is intended to convert user-created services only. 
NAME                                                         PDB
------------------------------------------------------------ ------------------------------
salespdb                                                     SALESPDB
salespdb2                                                    SALESPDB2

1.3) plug in an unplugged PDB   
```
# generate a description file, check compatibility 
execute DBMS_PDB.DESCRIBE(pdb_descr_file => '/disk1/oracle/salespdb.xml', pdb_name       => 'SALESPDB@remote');
SET SERVEROUTPUT ON
DECLARE
  compatible CONSTANT VARCHAR2(3) := 
    CASE DBMS_PDB.CHECK_PLUG_COMPATIBILITY(
           pdb_descr_file => '/oracle/data/salespdb2.xml',
           pdb_name       => 'SALESPDB')
    WHEN TRUE THEN 'YES'
    ELSE 'NO'
END;
BEGIN
  DBMS_OUTPUT.PUT_LINE(compatible);
END;
/
#if it's compatible, then unplug db 
ALTER PLUGGABLE DATABASE salespdb2 UNPLUG INTO '/oracle/data/salespdb2.xml';
#plugin salespdb2 
CREATE PLUGGABLE DATABASE salespdb3 AS CLONE USING '/oracle/data/salespdb2.xml' 
  COPY
  FILE_NAME_CONVERT = ('/oracle/app/oracle/oradata/orcl/salespdb2', '/oracle/app/oracle/oradata/orcl/salespdb3');  
```
# 2, connect to PDB 
I choose to create CDB during installation, named it ORCL. There are 3 containers initially, they look like:  
```
set linesize 120 
col name format a60
select con_id, name, open_mode from v$containers;

    CON_ID NAME                           OPEN_MODE
---------- ------------------------------ ----------
         1 CDB$ROOT                       READ WRITE
         2 PDB$SEED                       READ ONLY
         3 ORCLPDB                        READ WRITE
SQL> select con_id, name from v$datafile order by con_id, name;

    CON_ID NAME
---------- ------------------------------------------------------------
         1 /oracle/app/oracle/oradata/orcl/sysaux01.dbf
         1 /oracle/app/oracle/oradata/orcl/system01.dbf
         1 /oracle/app/oracle/oradata/orcl/undotbs01.dbf
         1 /oracle/app/oracle/oradata/orcl/users01.dbf
         2 /oracle/app/oracle/oradata/orcl/pdbseed/sysaux01.dbf
         2 /oracle/app/oracle/oradata/orcl/pdbseed/system01.dbf
         2 /oracle/app/oracle/oradata/orcl/pdbseed/undotbs01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/sysaux01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/system01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/undotbs01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/users01.dbf

11 rows selected.

col name format a100
set linesize 120 
select con_id, name from v$tempfile order by con_id, name;
    CON_ID NAME
---------- ----------------------------------------------------------------------------------------------------
         1 /oracle/app/oracle/oradata/orcl/temp01.dbf
         2 /oracle/app/oracle/oradata/orcl/pdbseed/temp012019-02-04_18-58-29-382-PM.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/temp01.dbf

```
Note that each PDB, including root, seed, has its own sysaux, system, undotbs, and temp tablespace.   
a PDB can either have its own temporary tablespace, or if it is created without a temporary tablespace, it can share the temporary tablespace with CDB. 
Temp files cannot be backed up and because no redo is ever generated for them, RMAN never restores or recovers temp files.   
RMAN does track the names of temp files, but only so that it can automatically re-create them when needed.    

# in orclpdb, the missing temp files will be recreated automaitcally on startup 
```
DROP TABLESPACE temp INCLUDING CONTENTS AND DATAFILES;   # ORA-12906: cannot drop default temporary tablespace
connect as sys @ CDB 
shutdown immediate;
cd /oracle/app/oracle/oradata/orcl/orclpdb; rm temp01.dbf
startup; 
col name format a30
select con_id, name, open_mode from v$containers;
    CON_ID NAME                           OPEN_MODE
---------- ------------------------------ ----------
         1 CDB$ROOT                       READ WRITE
         2 PDB$SEED                       READ ONLY
         3 ORCLPDB                        MOUNTED
         4 SALESPDB2                      MOUNTED
alter pluggable database all open; 
SQL> select con_id, name, open_mode from v$containers;

    CON_ID NAME                                                         OPEN_MODE
---------- ------------------------------------------------------------ ----------
         1 CDB$ROOT                                                     READ WRITE
         2 PDB$SEED                                                     READ ONLY
         3 ORCLPDB                                                      READ WRITE
         4 SALESPDB3                                                    READ WRITE
         5 SALESPDB                                                     READ WRITE
         6 SALESPDB2                                                    MOUNTED
SQL> col pdb_name format a30
SQL> select pdb_name, status from dba_pdbs;

PDB_NAME                       STATUS
------------------------------ ----------
ORCLPDB                        NORMAL
PDB$SEED                       NORMAL
SALESPDB                       NORMAL
SALESPDB3                      NORMAL
SALESPDB2                      UNPLUGGED
SQL> host ls -ltr
........
-rw-rw---- 1 oracle oinstall  67117056 Feb 11 19:34 temp01.dbf
```
## 2.1) connect to PDB : alter session set container=
```
SQL> alter session set container=orclpdb;
SQL> select name from v$database;
NAME
---------
ORCL
SQL> show con_name;
CON_NAME
------------------------------
ORCLPDB

SQL> col username format a30
SQL> select username, account_status from dba_users;
SQL> alter user HR account unlock identified by HR;
```

# 2.2) connect to PDB: service 
Direct connections to pluggable database must be made using a service. 
Each pluggable database automatically registers a service with the listener. 
```
col name format a30
col pdb format a20
select name, pdb from dba_services;    -- this one only shows the service inside root container 
NAME                           PDB
------------------------------ --------------------
SYS$BACKGROUND                 CDB$ROOT
SYS$USERS                      CDB$ROOT
orclXDB                        CDB$ROOT
orcl                           CDB$ROOT
SQL> select name, pdb from v$services;  -- shows the service inside whole CDB 
NAME                           PDB
------------------------------ --------------------
orclXDB                        CDB$ROOT
ORCLPDB_SVC                    ORCLPDB
salespdb2                      SALESPDB2
orcl                           CDB$ROOT
SYS$BACKGROUND                 CDB$ROOT
SYS$USERS                      CDB$ROOT
orclpdb                        ORCLPDB

#To create another service for orclpdb manually 
SQL> alter session set container=ORCLPDB;
SQL> BEGIN
  DBMS_SERVICE.create_service('ORCLPDB_SVC','ORCLPDB_NETWORK');
  DBMS_SERVICE.start_service('ORCLPDB_SVC');
END;#
/
col #
col #
select name, pdb from dba_services;
NAME                           PDB
------------------------------ --------------------
orclpdb                        ORCLPDB
ORCLPDB_SVC                    ORCLPDB

# service name vs. network name 
A service name is a feature in which a database can register itself with the listener. it can be used as a parameter in tnsnames.ora. otherwise a SID can be used in tnsnames.ora. 
a Net service name resolves to the connect descriptor, the Db server host, IP, Db service name 
dbms_service.create_service:  network_name - Network name of the service as used in SQLNet connect descriptors for client connections; same as net service name 
```
with service, you can connec to PDB via EZCONNECT 
SQL> CONN HR/HR@//localhost:1521/ORCLPDB
Connected.
SQL> select name from v$database;           # in PDB, no privileges on system view like v$database 
select name from v$database
                 *
ERROR at line 1:
ORA-00942: table or view does not exist
SQL> show con_name;
CON_NAME
------------------------------
ORCLPDB

## 2.3) connect to PDB: TNS entry   
add following into tnsnames.ora: 
```
ORCLPDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = kanlinux54106)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLPDB)
    )
  )
```
then we can connect to PDB by:  sqlplus HR/HR@orclpdb

#3, restore corrupted datafiles in PDB 
SQL> connect system/oracle@orclpdb
Connected.
SQL> col name format a60
SQL> select con_id, name from v$datafile order by con_id, name;

    CON_ID NAME
---------- ------------------------------------------------------------
         3 /oracle/app/oracle/oradata/orcl/orclpdb/sysaux01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/system01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/undotbs01.dbf
         3 /oracle/app/oracle/oradata/orcl/orclpdb/users01.dbf

--enable archivelog mode 
shutdown immediate; 
startup mount; 
alter database archivelog;
alter database open; 
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /oracle/app/oracle/product/12.2.0/dbhome_1/dbs/arch
Oldest online log sequence     21
Next log sequence to archive   23
Current log sequence           23
SQL> alter system switch logfile;
System altered.
SQL> host ls -ltr /oracle/app/oracle/product/12.2.0/dbhome_1/dbs/arch
ls: cannot access /oracle/app/oracle/product/12.2.0/dbhome_1/dbs/arch: No such file or directory
-- note that if archivelog destination does not exist, alter system switch logfile does not report error    
   until the destination path being created, the logfile will not be actually archived and switched. 
SQL> host mkdir -p /oracle/app/oracle/product/12.2.0/dbhome_1/dbs/arch
SQL> alter system switch logfile;
System altered.
SQL>  host ls -ltr /oracle/app/oracle/product/12.2.0/dbhome_1/dbs/arch
total 24
-rw-rw---- 1 oracle oinstall 20992 Feb 12 14:46 1_25_999370667.dbf

-- RMAN backup CDB 
rman target / 
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/oracle/data/backup/ora_df%t_s%s_s%p';
BACKUP DATABASE PLUS ARCHIVELOG; 

-- RMAN restore PDB files 
   after CDB startup, all PDBs are in mount status 
rm /oracle/app/oracle/oradata/orcl/salespdb/sales01.dbf
SQL> alter pluggable database all open;
alter pluggable database all open
*
ERROR at line 1:
ORA-01157: cannot identify/lock data file 36 - see DBWR trace file
ORA-01110: data file 36: '/oracle/app/oracle/oradata/orcl/salespdb/sales01.dbf'

alter session set container=salespdb; 
alter database datafile 36 offline;
SQL> alter pluggable database salespdb open;
Pluggable database altered.
SQL> col error format a40
SQL> select file#, online_status, error from v$recover_file;

     FILE# ONLINE_ ERROR
---------- ------- ----------------------------------------
        36 OFFLINE OFFLINE NORMAL

rman target
restore datafile 36;
recover datafile 36; 

alter session set container=salespdb;
alter database datafile 36 online;
SQL> select file#, online_status, error from v$recover_file;
no rows selected

-- RMAN backup PDB 
rman target hrbkup@hrpdb

#4, restore temp files in PDB or CDB
delete it, it will be recreated after DB restart 

#5, restore sysaux tablespace in root container 
shutdown immediate; 
cd  /oracle/app/oracle/oradata/orcl
mv sysaux01.dbf sysaux01.dbf.bak
SQL> startup
......
Database mounted.
ORA-01157: cannot identify/lock data file 3 - see DBWR trace file
ORA-01110: data file 3: '/oracle/app/oracle/oradata/orcl/sysaux01.dbf'
restore datafile 3;
recover datafile 3; 
alter database open |resetlogs; 
alter pluggable database all open |resetlogs; 

#6, restore control file for a CDB 

#7, ADDM in CDB 
AWR - Automatic Workload Repository, 
ADDM - Automatic Database Diagnostic Monitor, analyzes AWR data on a regular basis, provides performance disgnostic report every hour by default. 

# 8, UNDO, 
in oracle 12.2, there are 2 modes of undo 
https://oracle-base.com/articles/12c/multitenant-local-undo-mode-12cr2   

shared undo   
local undo   





