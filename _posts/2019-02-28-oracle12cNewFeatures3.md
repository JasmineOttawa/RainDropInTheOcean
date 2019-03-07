---
layout: post
title: Oracle 12C New Features (3)
category: Oracle 
tags: [Oracle]
---

Features not used that frequently while show up in the certification exam:     
  
1, Invisible indexes are maintained like any other index, but they are ignored by the optimizer unless the OPTIMIZER_USE_INVISIBLE_INDEXES parameter is set to TRUE
2, Oracle 12c now provides the ability to index a subset of partitions and to exclude the others.
     Local and global indexes can now be created on a subset of the partitions of a table.
3, segment advisor, identifies segments that have space available fo reclamation. it performs its analysis by examing usage and growth statistics in the AWR, by samping the data in segment. 
  . determines if an object has a significant amount of free space. it recommends online segmen shrink. 
    if object is a table without automatic segment space management, it recommends online table redefinition 
  . determines if a table could benefit from compression with OLTP compression method. 
  . report a table with row chaining above a certain threshold    
4, RMAN multi-section backup 
   For backup very large data files, RMAN provides multisection backups as a way to parallelize the backup operation within the file itself, such that sections of a file are backed up in parallel,
   rather than backing up on a per-file basis. Wherever applicable, RMAN also uses unused block compression and block change tracking while creating multisection incremental backups. 
   When backup sets are used, you can create multisection full or incremental backups.
  
   If you specify a section size that is larger than the size of the file, then RMAN does not use multisection backups for that file. 
   If you specify a small section size that would produce more than 256 sections, then RMAN increases the section size to a value that results in exactly 256 sections.
   
5, DBMS_SQL_MONITOR.BEGIN_OPERATION
The SQL monitoring feature is enabled by default when the STATISTICS_LEVEL initialization parameter is either set to TYPICAL (the default value) or ALL. SQL monitoring starts automatically for all long-running queries.
because SQL monitoring is a feature of the Oracle Database Tuning Pack, the CONTROL_MANAGEMENT_PACK_ACCESS initialization parameter must be set to DIAGNOSTIC+TUNING (the default value).
VAR eid NUMBER
EXEC :eid := DBMS_SQL_MONITOR.BEGIN_OPERATION('sh_count',FORCED_TRACKING => ‘Y’);
SELECT count(*) FROM sh.sales;
...
EXEC DBMS_SQL_MONITOR.END_OPERATION('sh_count', :eid);
SELECT SUBSTR(DBOP_NAME, 1, 10), DBOP_EXEC_ID,  SUBSTR(STATUS, 1, 10) FROM  V$SQL_MONITOR  WHERE DBOP_NAME IS NOT NULL ORDER BY EXEC_ID;
NO_FORCE_TRACKING - the operation will be tracked only when it has consumed at least 5 seconds of CPU or I/O time.

6, tracing at the service level 
   DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE  , Enables SQL tracing for a given combination of Service Name, MODULE and ACTION globally unless an instance_name is specified
   Tracing information is present in multiple trace files and you must use the trcsess tool to collect it into a single file.
   
7, SQL plan directives
   https://oracle-base.com/articles/12c/sql-plan-directives-12cr1
SQL plan directives are like "extra notes" for the optimizer, to remind it that it previously selected a suboptimal plan, typically because of incorrect cardinality estimates. 
Incorrect cardinality estimates are often caused by issues like missing/stale statistics, complex predicates/operators. unlike SQL profiles, which are statement specific, SQL plan directives are linked 
to query expressions, so they can be used by several statements containining matching query expressions. situations like missing histograms or missing extended statistics may result in SQL plan directives generated. 
The database manages SQL plan directives internally, situations like automatic reoptimization may result in SQL plan directives being written to the SGA and later persisted to SYSAUX, at which point they can 
be displayed by : dba_sql_plan_directives, dba_sql_plan_dir_objects. they can also be persistened by dbms_SPD 
  
   
8, bigfile tablespace 
https://docs.oracle.com/cd/B28359_01/server.111/b28310/tspaces002.htm#i1010733
A bigfile tablespace is a tablespace with a single, but very large (up to 4G blocks) datafile.  
A bigfile tablespace with 8K blocks can contain a 32 terabyte datafile. A bigfile tablespace with 32K blocks can contain a 128 terabyte datafile.
The maximum number of datafiles in an Oracle Database is limited (usually to 64K files). 
Bigfile tablespaces are supported only for locally managed tablespaces with automatic segment space management, with three exceptions: locally managed undo tablespaces, temporary tablespaces, and the SYSTEM tablespace.
If the default tablespace type was set to BIGFILE at database creation, you need not specify the keyword BIGFILE in the CREATE TABLESPACE statement. A bigfile tablespace is created by default.
If the default tablespace type was set to BIGFILE at database creation, but you want to create a traditional (smallfile) tablespace, then specify a CREATE SMALLFILE TABLESPACE statement to override the default tablespace type for the tablespace that you are creating.
CREATE BIGFILE TABLESPACE bigtbs DATAFILE '/u02/oracle/data/bigtbs01.dbf' SIZE 50G

9, change a default tablespace, the user default tablespace are changed accordingly. 
SELECT PROPERTY_VALUE FROM DATABASE_PROPERTIES WHERE PROPERTY_NAME = 'DEFAULT_PERMANENT_TABLESPACE'; 
PROPERTY_VALUE
--------------------------------------------------------------------------------
USERS

col username format a30 
select username, default_tablespace from dba_users; 
CREATE BIGFILE TABLESPACE user2 DATAFILE '/oracle/app/oracle/oradata/orcl/orclpdb/user2.dbf' SIZE 64M;
alter database default tablespace user2; 

10, password file 
https://docs.oracle.com/cd/B28359_01/server.111/b28310/dba007.htm#ADMIN10241

Entries- corresponds to the number of distinct users allowed to connect to the database as SYSDBA or SYSOPER. The actual number of allowable entries can be higher than the number of users, 
because the ORAPWD utility continues to assign password entries until an operating system block is filled. For example, if your operating system block size is 512 bytes, it holds four password entries. 
The number of password entries allocated is always a multiple of four.

In addition to creating the password file, you must also set the initialization parameter REMOTE_LOGIN_PASSWORDFILE to the appropriate value: 
NONE: Setting this parameter to NONE causes Oracle Database to behave as if the password file does not exist. That is, no privileged connections are allowed over nonsecure connections.
EXCLUSIVE: (The default) An EXCLUSIVE password file can be used with only one instance of one database. Only an EXCLUSIVE file can be modified. Using an EXCLUSIVE password file enables you 
      to add, modify, and delete users. It also enables you to change the SYS password with the ALTER USER command.
SHARED: A SHARED password file can be used by multiple databases running on the same server, or multiple instances of an Oracle Real Application Clusters (RAC) database. A SHARED password file cannot be modified. 
      All users needing SYSDBA or SYSOPER system privileges must be added to the password file when REMOTE_LOGIN_PASSWORDFILE is set to EXCLUSIVE. After all users are added, you can change REMOTE_LOGIN_PASSWORDFILE to SHARED, and then share the file.

Note that this file does not actually contain password. 

11, SYSKM administrative privilege has ability to perform transparent data encryption wallet operations
SYSKM will be responsible for all TDE (Transparent Data Encryption) and Data Vault related administrative operations.

12. ILM ADO - Automatic Data Optimization
For the values of the TBS_PERCENT* parameters, ADO makes a best effort, but not a guarantee. When the percentage of the tablespace quota reaches the value of TBS_PERCENT_USED, 
ADO begins to move data so that percent free of the tablespace quota approaches the value of TBS_PERCENT_FREE. As an example, assume that TBS_PERCENT_USED is set to 85 and 
TBS_PERCENT_FREE is set to 25, and that a tablespace becomes 90 percent full. ADO then initiates actions to move data so that the tablespace quota has at least 25 percent free, 
which can also be interpreted as less than 75 percent used of the tablespace quota.

13, multiprocess, multithreaded architecture of Oracle Database 12c
https://docs.oracle.com/database/121/CNCPT/process.htm#CNCPT008
When Oracle Database 12c is installed, the database runs in process mode. You must set the THREADED_EXECUTION initialization parameter to TRUE to run the database in threaded mode. In threaded mode, some background processes on UNIX and Linux run as processes (with each process containing one thread), whereas the remaining Oracle processes run as threads within processes.
When the THREADED_EXECUTION initialization parameter is set to TRUE, operating system authentication is not supported.  You need a connection which is authenticated trough the password file.

https://petesdbablog.wordpress.com/2013/07/09/12c-new-feature-multi-process-multi-threaded-oracle/
CPU usage reduction
Memory usage reduction
Better performance for parallel executions / operations
Better system reliability

14， updating indexes 
https://docs.oracle.com/database/121/VLDBG/GUID-1D59BD49-CD86-4BFE-9099-D3B8D7FD932A.htm#VLDBG1122
By default, many table maintenance operations on partitioned tables invalidate (mark UNUSABLE) the corresponding indexes or index partitions. You must then rebuild the entire index or, for a global index, each of its partitions. The database lets you override this default behavior if you specify UPDATE INDEXES in your ALTER TABLE statement for the maintenance operation. Specifying this clause tells the database to update the indexes at the time it executes the maintenance operation DDL statement. This provides the following benefits:
The indexes are updated with the base table operation. You are not required to update later and independently rebuild the indexes.
The global indexes are more highly available, because they are not marked UNUSABLE. These indexes remain available even while the partition DDL is executing and can access unaffected partitions in the table.
You need not look up the names of all invalid indexes to rebuild them.

15, large pool 
https://docs.oracle.com/en/database/oracle/oracle-database/18/tgdba/tuning-shared-pool-and-large-pool.html#GUID-096F2CDD-B3EE-4124-BBE3-F60400600972
unlike the shared pool, the large pool does not have an LRU list, oracle database does not attempt to age objects out of the large pool. 
consider configuring a large pool if the database instance uses any of the following features: 
. shared server: session memory for each client process
. parallel query:  cache parallel execution message buffers
. recovery manager : cache I/O buffers during backup and restore operations. 

The large pool can provide large memory allocations for the following:
/(B)UGA(User Global Area)for the shared server and the Oracle XA interface (used where transactions interact with multiple databases)
/Message buffers used in the parallel execution of statements
/(A)Buffers for Recovery Manager (RMAN) I/O slaves

16, Hybrid Columnar Compression, initially only available on Exadata, has been extended to supportPillar Axiom and Sun ZFS Storage Appliance (ZFSSA) storage when used with Oracle Database Enterprise Edition 11.2.0.3 and above
https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/tables-and-table-clusters.html#GUID-901A054B-B47F-4ADF-A57B-2074EBB85341

17,  


