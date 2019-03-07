---
layout: post
title: Oracle 12C New Features (2)
category: Oracle 
tags: [Oracle]
---

Features not used that frequently while show up in the certification exam:       
Temporal Validity, DMU, Oracle Restart, ROWNUM& TOP-N query, encrypted tablespace & SALT 

# 1, In-Database Archiving  
https://docs.oracle.com/database/121/VLDBG/GUID-5A76B6CE-C96D-49EE-9A89-0A2CB993A933.htm#VLDBG14154

In-Database archiving enables you to archive rowa within a table by marking them as inactive. These inactive rows are in the database and can be optimized using compression, but are not visible to an application. With In-Database Archiving you can store more data for a longer period of time within a single database, without compromising application performance. Archived data can be compressed to help improve backup performance, and updates to archived data can be deferred during application upgrades to improve the performance of upgrades.

How it works:  ROW ARCHIVAL clause in create/aler table creates a hidden column: ORA_ARCHIVE_STATE. the value of this columne and session parameter ROW ARCHIVE VISIBILITY determines if rows available.   
```
ALTER SESSION SET ROW ARCHIVAL VISIBILITY = ACTIVE;  #ACTIVE/ALL, it is ACTIVE by default 
CREATE TABLE employees_indbarch (employee_id NUMBER(6) NOT NULL) ROW ARCHIVAL;
INSERT INTO employees_indbarch(employee_id) VALUES (251);
INSERT INTO employees_indbarch(employee_id) VALUES (252);

SELECT SUBSTR(COLUMN_NAME,1,22) NAME, SUBSTR(DATA_TYPE,1,20) DATA_TYPE, COLUMN_ID AS COL_ID,
  SEGMENT_COLUMN_ID AS SEG_COL_ID, INTERNAL_COLUMN_ID AS INT_COL_ID, HIDDEN_COLUMN, CHAR_LENGTH 
  FROM USER_TAB_COLS WHERE TABLE_NAME='EMPLOYEES_INDBARCH';
NAME                   DATA_TYPE                COL_ID SEG_COL_ID INT_COL_ID HID CHAR_LENGTH
---------------------- -------------------- ---------- ---------- ---------- --- -----------
ORA_ARCHIVE_STATE      VARCHAR2                                 1          1 YES        4000
EMPLOYEE_ID            NUMBER                        1          2          2 NO            0
...

COLUMN ORA_ARCHIVE_STATE FORMAT a18;
SELECT employee_id, ORA_ARCHIVE_STATE FROM employees_indbarch;
EMPLOYEE_ID ORA_ARCHIVE_STATE
----------- ------------------
        251 0
        252 0

UPDATE employees_indbarch SET ORA_ARCHIVE_STATE = '20' WHERE employee_id = 252;
SELECT employee_id, ORA_ARCHIVE_STATE FROM employees_indbarch;   -- now only shows 251 
ALTER SESSION SET ROW ARCHIVAL VISIBILITY = ALL;
SELECT employee_id, ORA_ARCHIVE_STATE FROM employees_indbarch;   -- now show all rows 
```

#2, ILM, ADO 
Managing data in Oracle database with ILM:  https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/manage-data-db-ilm.html#GUID-AC2B567F-14EF-4E7A-9992-076A2A820305
Implementing an ILM strategy with Heat Map and ADO: https://docs.oracle.com/en/database/oracle/oracle-database/12.2/vldbg/ilm-strategy-heatmap-ado.html#GUID-1E510486-2ED8-467A-A5BA-045F4F3AC324
To implement an ILM - Information lifecycle Management strategy for data movement in your database, you can use Heat Map and Automatic Data Optimization features. 
a) using heat map 
   Heat Map provides data access tracking at segment level and data modification tracking at segment/row level. 
   -- enable Heat Map, all accesses are tracked by the in-memory tracking module, objects in SYSTEM/SYSAUX are not tracked
      it's also a prerequisite for ADO
   alter system set heat_map = ON: 
   -- displaying heap map tracking data 
   SELECT SUBSTR(OBJECT_NAME,1,20), SUBSTR(SUBOBJECT_NAME,1,20), TRACK_TIME, SEGMENT_WRITE, FULL_SCAN, LOOKUP_SCAN FROM V$HEAT_MAP_SEGMENT;
   SELECT SUBSTR(OBJECT_NAME,1,20), SUBSTR(SUBOBJECT_NAME,1,20), SEGMENT_WRITE_TIME, SEGMENT_READ_TIME, FULL_SCAN, LOOKUP_SCAN FROM USER_HEAT_MAP_SEGMENT;
   SELECT SUBSTR(OBJECT_NAME,1,20), SUBSTR(SUBOBJECT_NAME,1,20), TRACK_TIME, SEGMENT_WRITE, FULL_SCAN,  LOOKUP_SCAN FROM USER_HEAT_MAP_SEG_HISTOGRAM;
   SELECT SUBSTR(OWNER,1,20), SUBSTR(OBJECT_NAME,1,20), OBJECT_TYPE, SUBSTR(TABLESPACE_NAME,1,20),  SEGMENT_COUNT FROM DBA_HEATMAP_TOP_OBJECTS ORDER BY SEGMENT_COUNT DESC;
   SELECT SUBSTR(TABLESPACE_NAME,1,20), SEGMENT_COUNT FROM DBA_HEATMAP_TOP_TABLESPACES ORDER BY SEGMENT_COUNT DESC;
   -- managing Heat Map data with DBMS_HEAT_MAP subprograms 
   provides additional flexibility for displaying heat map data 
   SELECT SUBSTR(segment_name,1,10) Segment, min_writetime, min_ftstime FROM TABLE(DBMS_HEAT_MAP.OBJECT_HEAT_MAP('SH','SALES'));
   SELECT SUBSTR(owner,1,10) Owner, SUBSTR(segment_name,1,10) Segment, SUBSTR(partition_name,1,16) Partition, SUBSTR(tablespace_name,1,16) Tblspace, segment_type, segment_size FROM TABLE(DBMS_HEAT_MAP.OBJECT_HEAT_MAP('SH','SALES'));
   SELECT SUBSTR(tablespace_name,1,16) Tblspace, min_writetime, min_ftstime FROM  TABLE(DBMS_HEAT_MAP.TABLESPACE_HEAT_MAP('EXAMPLE'));
   SELECT relative_fno, block_id, blocks, TO_CHAR(min_writetime, 'mm-dd-yy hh-mi-ss') Mintime, TO_CHAR(max_writetime, 'mm-dd-yy hh-mi-ss') Maxtime,  TO_CHAR(avg_writetime, 'mm-dd-yy hh-mi-ss') Avgtime  FROM TABLE(DBMS_HEAT_MAP.EXTENT_HEAT_MAP('SH','SALES')) WHERE ROWNUM < 10;   
   
b) using ADO 
   Automate the compression and movement of data between different tires of storage within the database 
   Create policies that specify different compression levels for each tier, and control when the data movement takes place. 
   -- managing policies for ADO 
      You can specify policies for ADO at row, segment, tablespace granularity, ILM ADO policies are given a system-generated name, such P1, P2, and so on to Pn.
      A segment level policy executes only one time. After the policy executes successfully, it is disabled and is not evaluated again. 
      However, you can explicitly enable the policy again. A row level policy continues to execute and is not disabled after a successful execution.
      The scope of an ADO policy can be specified for a group of related objects or at the level of a segment or row, using the keywords GROUP, ROW, or SEGMENT.
      The default mappings for compression that can be applied to group policies are:
COMPRESS ADVANCED on a heap table maps to standard compression for indexes and LOW for LOB segments.
COMPRESS FOR QUERY LOW/QUERY HIGH on a heap table maps to standard compression for indexes and MEDIUM for LOB segments.
COMPRESS FOR ARCHIVE LOW/ARCHIVE HIGH on a heap table maps to standard compression for indexes and HIGH for LOB segments.
     In-Memory Column Store: DBA_ILMDATAMOVEMENTPOLICIES; V$HEAT_MAP_SEGMENT; V$IM_ADOTASKS ; V$IM_ADOTASKDETAILS
     CREATE OR REPLACE FUNCTION my_custom_ado_rules (objn IN NUMBER) RETURN BOOLEAN;
     ALTER TABLE sales_custom ILM ADD POLICY COMPRESS ADVANCED SEGMENT ON my_custom_ado_rules;
         
   -- creating a table with an ILM ADO policy 
CREATE TABLE sales_ado 
 (PROD_ID NUMBER NOT NULL,
  CUST_ID NUMBER NOT NULL, 
  TIME_ID DATE NOT NULL, 
  CHANNEL_ID NUMBER NOT NULL,
  PROMO_ID NUMBER NOT NULL,
  QUANTITY_SOLD NUMBER(10,2) NOT NULL,
  AMOUNT_SOLD NUMBER(10,2) NOT NULL )
 PARTITION BY RANGE (time_id)
 ( PARTITION sales_q1_2012 VALUES LESS THAN (TO_DATE('01-APR-2012','dd-MON-yyyy')),
   PARTITION sales_q2_2012 VALUES LESS THAN (TO_DATE('01-JUL-2012','dd-MON-yyyy')),
   PARTITION sales_q3_2012 VALUES LESS THAN (TO_DATE('01-OCT-2012','dd-MON-yyyy')),
   PARTITION sales_q4_2012 VALUES LESS THAN (TO_DATE('01-JAN-2013','dd-MON-yyyy')) )
  ILM ADD POLICY COMPRESS FOR ARCHIVE HIGH SEGMENT AFTER 12 MONTHS OF NO ACCESS;
SELECT SUBSTR(policy_name,1,24) POLICY_NAME, policy_type, enabled  FROM USER_ILMPOLICIES;
  
   -- adding ILM ADO policies 
      ... ILM ADD POLICY ROW STORE COMPRESS ADVANCED ROW AFTER 30 DAYS OF NO MODIFICATION;     -- row level compression 
          ILM ADD POLICY COMPRESS FOR ARCHIVE HIGH SEGMENT AFTER 6 MONTHS OF NO MODIFICATION;  -- segment level compression 
          ILM ADD POLICY COMPRESS FOR ARCHIVE HIGH SEGMENT AFTER 12 MONTHS OF NO ACCESS;
          ILM ADD POLICY TIER TO my_low_cost_sales_tablespace;   -- Add storage tier policy to move old data to a different tablespace          
  
   -- disabling and deleting ILM ADO policies 
      delete/disable policy p1/all; 
      
   -- specifying segment-level compression and storage tiering with ADO 
      ALTER TABLE sales_ado ILM ADD POLICY COMPRESS FOR ARCHIVE HIGH SEGMENT  AFTER 6 MONTHS OF NO MODIFICATION;
      ALTER TABLE sales_ado ILM ADD POLICY TIER TO my_low_cost_tablespace;
      SELECT SUBSTR(policy_name,1,24) POLICY_NAME, policy_type, enabled   FROM USER_ILMPOLICIES;
   -- specifying row-level compression tiering with ADO 
      ALTER TABLE employees_ilm ILM ADD POLICY COLUMN STORE COMPRESS FOR QUERY ROW AFTER 30 DAYS OF NO MODIFICATION;  -- HCC, Hybrid Columnar Compression 
      ALTER TABLE sales_ado ILM ADD POLICY ROW STORE COMPRESS ADVANCED ROW AFTER 60 DAYS OF NO MODIFICATION;
      SELECT policy_name, policy_type, enabled FROM USER_ILMPOLICIES;
      
   -- managing ILM ADO parameters 
      DBMS_ILM_ADMIN.CUSTOMIZE_ILM 
      ABSOLUTEJOB LIMIT ,  absolute number of concurrent ADO jobs 
      DEGREEOF PARALLELISM, parallelism in which the ADO policy jobs are run 
      ENABLED ,  control ADO background evaluation and execution, by default it is on 
          If the HEAT_MAP initialization parameter is set to ON and the ENABLED parameter is set to FALSE (0), then heat map statistics are collected, but ADO does not act on the statistics automatically.
          If the HEAT_MAP initialization parameter is set to OFF and the ENABLED parameter is set to TRUE (1), then heat map statistics are not collected and because ADO cannot rely on the heat map statistics, ADO does nothing. ADO behaves as if ENABLED is set to FALSE.
      EXECUTION MODE, controls whether ADO executes in online or offline mode. by default it is on. 
      EXECUTION INTERVAL, frequency that ADO initiates background evaluation, default to 15 minutes 
      JOB LIMIT, default value is 2, it is (number of instances)*(number of CPUs for each instance)
      POLICY TIME, determines if ADO policies are specified in seconds or days. values are 1 for seconds or 0 for days
      RETENTION TIME, the length of time that data of completed ADO tasks is kept before that data is purged. the default is 30 days. 
      TBS PERSENT USED, percentage of the tablespace quota when a tablespace is considered full. default is 85 percent 
      TBS PERCENT FREE, specifies the targeted free percentage for the tablespace, default is 25. 
      SELECT NAME, VALUE FROM DBA_ILMPARAMETERS;      
   -- using PL/SQL fucntions for policy management    
   -- using views to monitor policies for ADO 
      dba_ilmdatamovementpolicies 
      dba_ilmtasks 
      dba_ilmevaluationdetails 
      dba_ilmobjects 
      dba_ilmpolicies 
      dba_ilmresults 
      dba_ilmparameters   
c) limitations and restrictions 


#3, Database Smart Flash Cache
https://docs.oracle.com/cd/E18283_01/server.112/e17120/memory005.htm
-- when to configure the Flash Cache
   consider adding flash cache when all of the following is true: 
   your database is running on solaris or oracle enterprise linux OS, flash cache is supported on these OS only. 
   the buffer pool advisory section of your AWR report indicates that doubling the size of buffer cache would be beneficial 
   db file sequential read is a top wait event 
   your have spare CPU 
-- Sizing the Flash Cache
   As a general rule, size the flash cache to be between 2 times and 10 times the size of the buffer cache.  
-- Tuning memory for the Flash Cache 
-- Flash Cache Initialization parameter 
   db_flash_cache_file/size
-- Flash Cache in RAC 


