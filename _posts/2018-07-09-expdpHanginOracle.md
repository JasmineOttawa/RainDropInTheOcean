---
layout: post
title: Oracle - SHMALL 
category: Oracle
tags: [Oracle]
---

# Question 
expdp hung on “wait for unread message on broadcast channel”    

# Investigation 
expdp $user/$pass directory=aaa parfile=aaa.par parallel=10 logfile=aaa.log exclude=CONSTRAINT,INDEX,TRIGGER dumpfile=aaa.dmp FLASHBACK_TIME=$flashback_time job_name=AAA reuse_dumpfiles=Y content=ALL network_link=$DBNAME    

Add TRACE=480300 in above command line,
trace was generated on dm process, it stucks at a system call:   
```
select sql_text from v$sqlarea where sql_id = ’65sh6sucvcnc8’;
SQL_TEXT
BEGIN SYS.KUPM$MCP.MAIN(‘STRMINST’, ‘R6’, 0, 4719360); END;
```

Note that parallel, FLASHBACK_TIME are the 2 features involved in some bugs,   
tried to remove parallel, same error   
remove flashback_time, expdp works fine  

Here expdp is used to copy data from a master site, setup a base for streams replication.   
flashback_time is to perform a time consistent export, without it, what will be the side affect?   
– no side effect, provided we setting streams instantiation SCN earlier than the starting time of export.  

