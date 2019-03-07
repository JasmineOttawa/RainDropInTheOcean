---
layout: post
title: Oracle 12C New Features 
category: Oracle 
tags: [Oracle]
---

Features not used that frequently while show up in the certification exam:       
Temporal Validity, DMU, Oracle Restart, ROWNUM& TOP-N query, encrypted tablespace & SALT 

# 1, Temporal Validity
https://docs.oracle.com/database/121/ADFNS/adfns_design.htm#ADFNS967  
Temporal Validity Support, associate one or more valid time dimensions with a table and have data be visible depending on its time-based validity.   
It is typically used with Oracle Flashback for queries that specify the valid period in AS OF and VERSIONS BETWEEN clauses.   
It is useful in ILM - Information Lifecycle Management   
```
CREATE TABLE my_emp(
  empno NUMBER,
  last_name VARCHAR2(30),
  start_time TIMESTAMP,
  end_time TIMESTAMP,
PERIOD FOR user_valid_time (start_time, end_time));

-- To add Temporal Validiy to an existing table by following SQL if there is no START_TIME and END_TIME columns, it will add two hidden columns to the table MY_EMP: USER_VALID_TIME_START and USER_VALID_TIME_END. You can insert rows that specify values for these columns, but the columns do not appear in the output of the SQL*Plus DESCRIBE statement, and SELECT statements show the data in those columns only if the SELECT list explicitly includes the column names.
ALTER TABLE my_emp ADD PERIOD FOR user_valid_time;   

INSERT INTO my_emp VALUES (100, 'Ames', '01-Jan-10', '30-Jun-11');
INSERT INTO my_emp VALUES (101, 'Burton', '01-Jan-11', '30-Jun-11');
INSERT INTO my_emp VALUES (102, 'Chen', '01-Jan-12', null);

SELECT * from my_emp AS OF PERIOD FOR user_valid_time TO_TIMESTAMP('01-Jun-10');  -- returns only Ames 
SELECT * from my_emp VERSIONS PERIOD FOR user_valid_time BETWEEN TO_TIMESTAMP('01-Jun-10') AND TO_TIMESTAMP('02-Jun-10'); -- returns only Ames 

-- support session level visibility control for temporal table queries 
https://docs.oracle.com/database/121/VLDBG/GUID-AF78C832-516A-4686-9DDF-CE12597F7723.htm#VLDBG14127  
EXECUTE DBMS_FLASHBACK_ARCHIVE.enable_at_valid_time('ASOF', '31-DEC-12 12.00.01 PM');  -- sets the valid time visibility as of the given time.
EXECUTE DBMS_FLASHBACK_ARCHIVE.enable_at_valid_time('CURRENT');
EXECUTE DBMS_FLASHBACK_ARCHIVE.enable_at_valid_time('ALL');   --  sets the visibility of temporal data to the full table, which is the default temporal table visibility.
```

# 2, DMU - Database Migration Assistant for Unicode 
https://docs.oracle.com/database/121/DUMAG/toc.htm  
character set A is a superset of character set B if A contains or defines all characters of B plus some other characters.   character set A is a strict or binary superset of character set B if A contains or defines all characters of B and each such character has identical byte representation in both A and B.   By definition, if data is converted from a character set to its superset, no replacement characters are used because all characters are available in the superset; if data is converted from a character set to its binary (strict) superset, no character codes actually change.   

The replacement character of a given source character may be defined explicitly in the target character set. For example, ä (a with an umlaut) is replaced by a. when converted to the character set US7ASCII. If no explicit replacement character is defined, the default replacement character of the target character set is used. This character is usually the question mark ? or the inverted question mark ¿. The default replacement character in Unicode character sets is the character U+FFFD.

changeless - If character data in a database to be converted can be classified into data that needs no conversion
convertible - data that requires conversion,
character set migration - The process of identifying the suitable superset, classifying character data as changeless or convertible, checking for issues and conversion requirements, fixing the issues, changing the declaration in the data dictionary and converting data as needed 

A-to-B migration - when data is copied from a source database into a new database defined with the target character set   
inline migation - data is converted inside a database and the character set information of the database is changed   

Oracle supports two encodings of Unicode as the database character set: UTF-8 through the AL32UTF8 character set and CESU-8 through the UTF8 character set. (Note the use of the hyphen in the name of Unicode encoding and the lack of a hyphen in the name of Oracle character set. the later one is deprecated.    
UTF-8 is a multibyte varying width Unicode encoding using 1 to 4 bytes per character. AL32UTF8 encodes ASCII characters in 1 byte, characters from European, and Middle East languages in 2 bytes, characters from South and East Asian languages in 3 bytes. 

steps for character set migration tools: 
## 2.1, CSSCAN - identify migration issues with Database Character Set Scanner 
      most os its functions are included in DMU. it's not available in 12c 
## 2.2, changing character set metadata using CSALTER script 
      CSALTER script is part of CSSCAN, not available in 12c either. 
## 2.3, converting character set using export/import data pump utilities 
## 2.4, migrating database using DMU  
      support database 10.2.0.4 or later 
      
# 3, Oracle Restart 
https://docs.oracle.com/database/121/ADMIN/restart.htm#ADMIN12708  
Oracle restart enhances the availability of Oracle databases in a single-instance environment. It automatically restarts various components after hardware/software failuer. 
Oracle Restarted manages following components :    
database instance, including database services:  Oracle Restart can accomodate multiple instance on a single host   
oracle net listener    
ASM instance   
ASM disk groups: restart a disk group mean mounting it.   
ONS   

Oracle Restart ensures that Oracle components are started in the proper order, in accordance with component dependencies. For example, if database files are stored in Oracle ASM disk groups, then before starting the database instance, Oracle Restart ensures that the Oracle ASM instance is started and the required disk groups are mounted. Likewise, if a component must be shut down, Oracle Restart ensures that dependent components are cleanly shut down first; Oracle Restart also manages the weak dependency between database instances and the Oracle Net listener (the listener): When a database instance is started, Oracle Restart attempts to start the listener. If the listener startup fails, then the database is still started. If the listener later fails, Oracle Restart does not shut down and restart any database instances.

An important difference between starting a component with SRVCTL and starting it with SQL*Plus (or another utility) is the following:  
When you start a component with SRVCTL, any components on which this component depends are automatically started first, and in the proper order.  
When you start a component with SQL*Plus (or another utility), other components in the dependency chain are not automatically started; you must ensure that any components on which this component depends are started.

# 4, ROWNUM and Row Limiting for TOP-N queries in Oracle 12c 
https://blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results

ROWNUM is a pseudocolumn that is available in a query.  A ROWNUM value is assigned to a row after it passes the predicate phase of the query but before the query does any sorting or aggregation. Also, a ROWNUM value is incremented only after it is assigned, so the following query will never returns a row. 
select * from t where rownum>1; 

Here is the order of a query 
```
select ..., ROWNUM
  from t
 where <where clause>
 group by <columns>
having <having clause>
 order by <columns>;

The FROM/WHERE clause goes first.
ROWNUM is assigned and incremented to each output row from the FROM/WHERE clause.
SELECT is applied.
GROUP BY is applied.
HAVING is applied.
ORDER BY is applied.
```

so, query select * from emp  where ROWNUM <= 5  order by sal desc;  won't return the top 5 highest salaries. 
the correct version is to : sort EMP by salary descending and then return the first five records it encounters (the top-five records)   
```
select * from  ( select *  from emp order by sal desc ) where ROWNUM <= 5;
```

https://oracle-base.com/articles/12c/row-limiting-clause-for-top-n-queries-12cr1
```
SELECT val FROM   rownum_order_test ORDER BY val DESC FETCH FIRST 5 ROWS ONLY;
SELECT val FROM   rownum_order_test ORDER BY val DESC FETCH FIRST 5 ROWS WITH TIES;
SELECT val FROM   rownum_order_test ORDER BY val FETCH FIRST 20 PERCENT ROWS ONLY;
```
Paging 
```
SELECT val
FROM   (SELECT val, rownum AS rnum
        FROM   (SELECT val
                FROM   rownum_order_test
                ORDER BY val)
        WHERE rownum <= 8)
WHERE  rnum >= 5;
SELECT val FROM   rownum_order_test ORDER BY val OFFSET 4 ROWS FETCH NEXT 4 ROWS ONLY;
SELECT val FROM   rownum_order_test ORDER BY val OFFSET 4 ROWS FETCH NEXT 20 PERCENT ROWS ONLY;
```

The keywords ROW and ROWS can be used interchangeably, as can the FIRST and NEXT keywords. Pick the ones that scan best when reading the SQL like a sentence.
If the offset is not specified it is assumed to be 0.
Negative values for the offset, rowcount or percent are treated as 0.
Null values for offset, rowcount or percent result in no rows being returned.
Fractional portions of offset, rowcount or percent are truncated.
If the offset is greater than or equal to the total number of rows in the set, no rows are returned.
If the rowcount or percent are greater than the total number of rows after the offset, all rows are returned.
The row limiting clause can not be used with the FOR UPDATE clause, CURRVAL and NEXTVAL sequence pseudocolumns or in an fast refresh materialized view.

# 5, encrypted tablespace and SALT 
https://docs.oracle.com/cd/B28359_01/network.111/b28530/asotrans.htm#BABDJEJJ  
https://docs.oracle.com/cd/B28359_01/network.111/b28530/asotrans.htm#g1011122
 
Transparent data encryption helps protect data stored on media in the event that the storage media or data file gets stolen. 
It is a key-based access control system. even if the encrypted data is retrieved, it cannot be understtod until authorized decryption occurs. When a table contains encrypted columns, a single key is used regardless of the number of encrypted columns. this key is called the column encryption key. the column encryption keys for all tables, containing encrypted columns, are encrypted with the database server master encryption key and stored in a dictionary table in the database. The master encryption key is stored in an external security module that is outside the database, here is Oracle Wallet. 
You cannot use transparent data encryption to encrypt columns used in foreign key constraints. This is because every table has a unique column encryption key.
Transparent data encryption encrypts and decrypts data at the SQL layer. Oracle Database utilities and features that bypass the SQL layer cannot leverage the services provided by transparent data encryption. 
```
ALTER SYSTEM SET ENCRYPTION KEY IDENTIFIED BY "mypassword";  #if a wallet does not exits, then a new one is created. 
ORA-28368: cannot auto-create wallet

mkdir -p /$ORACLE_BASE/admin/<database_name>/wallet    # create default directory for wallet, now it works 
ALTER SYSTEM SET ENCRYPTION KEY IDENTIFIED BY "Or@cle123";   # the master encryption key remains accessible until instance down. 
System altered.

ALTER SYSTEM SET ENCRYPTION WALLET OPEN IDENTIFIED BY "Or@cle123";   # to load master encryption key after the database restart 
CREATE TABLE table_name ( column_name column_type ENCRYPT,....);  # ENCRYPT keyword against a column specifies that the column should be encrypted 
ALTER TABLE table_name MODIFY ( column_name column_type ENCRYPT,...); 
ALTER SYSTEM SET ENCRYPTION WALLET CLOSE;    # disable access to all encrypted columnd in the database 
```

Tablespace encryption encrypts all data that is stored in an encrypted tablespace and its corresponding redo data. Available since oracle 11.1 
```
step1,  set the tablespace master encryption key   
Tablespace encryption uses the same software wallet that is used by column-based transparent data encryption to store tha master encryption key.   
Check to ensure that the ENCRYPTION_WALLET_LOCATION (or WALLET_LOCATION) parameter in the sqlnet.ora file points to the correct software wallet location 
ENCRYPTION_WALLET_LOCATION=
  (SOURCE=(METHOD=FILE)(METHOD_DATA=
   (DIRECTORY=/oracle/app/oracle/admin/orcl/wallet/)))

step2, open the oracle wallet. The wallet must also be open before you can access data in an encrypted tablespace.

step3, create an encrypted tablespace. For security reasons, a tablespace cannot be encrypted with the NO SALT option.
CREATE TABLESPACE securespace
DATAFILE '/oracle/app/oracle/oradata/orcl/secureTBS01.dbf' SIZE 64M
ENCRYPTION USING '3DES168'
DEFAULT STORAGE(ENCRYPT);

SQL> select tablespace_name, encrypted from dba_tablespaces;
TABLESPACE_NAME                ENC
------------------------------ ---
......
SECURESPACE                    YES

create table secT1(col1 integer, col2 varchar2(16)) tablespace securespace; 
                   
                   
```
Salt is a way to strength the security of encrypted data. It is a random string added to the data before it is encrypted, causing repetition of text in the clear to appear different when encrypted. Salt removes the one common method attackers use to steal data, namely, matching patterns of encrypted text.
```
# test create a table in encrypted tablespace, 
create table secT2(col1 integer, col2 varchar2(16) ENCRYPT SALT) tablespace securespace; 
ORA-28336: cannot encrypt SYS owned objects

create user user1 identified by user1 default tablespace users temporary tablespace temp;
ORA-65096: invalid common user or role name

create user c##user1 identified by user1 default tablespace users temporary tablespace temp;
ORA-65048: error encountered when processing the current DDL statement in pluggable database SALESPDB3
ORA-00959: tablespace 'USERS' does not exist

SQL> create user c##user1 identified by user1;
User created.
grant dba to c##user1; 
connect c##user1/user1@orcl
create table secT2(col1 integer, col2 varchar2(16) ENCRYPT SALT) tablespace securespace; 
Table created.

# after closing wallet, you won't be able to create a table in encrypted tablespace, with or without SALT option 
ALTER SYSTEM SET ENCRYPTION WALLET CLOSE; 
ORA-28390: auto login wallet not open but encryption wallet may be open
ALTER SYSTEM SET ENCRYPTION WALLET CLOSE IDENTIFIED BY "Or@cle123"; 
System altered.
create table secT3(col1 integer, col2 varchar2(16) ENCRYPT SALT) tablespace securespace; 
ORA-28365: wallet is not open
create table secT3(col1 integer, col2 varchar2(16)) tablespace securespace; 
ORA-28365: wallet is not open

# try wallet in PDB, 
connect system/oracle@orclpdb
CREATE TABLESPACE securespace
DATAFILE '/oracle/app/oracle/oradata/orclpdb/secureTBS01.dbf' SIZE 64M
ENCRYPTION USING '3DES168'
DEFAULT STORAGE(ENCRYPT);
ORA-28365: wallet is not open
SQL> select wallet_type, status from v$encryption_wallet;

WALLET_TYPE          STATUS
-------------------- ------------------------------
UNKNOWN              CLOSED

connect sys@orclpdb
ALTER SYSTEM SET ENCRYPTION WALLET OPEN IDENTIFIED BY "Or@cle123";
ORA-65040: operation not allowed from within a pluggable database

```





































 

