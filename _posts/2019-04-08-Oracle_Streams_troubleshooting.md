---
layout: post
title: Oracle - 2 Streams troubleshooting cases
category: Oracle
tags: [Oracle]
---

# 1- DDL ping-pong 
In a 2-node streams replication, the archived redo log is huge despite of very little user activity.  
By mining archivelog we learn that a DDL (grant privilege) ping-pong between 2 nodes.  
We hit Oracle bug 14702322  

Talking with R&D, DDL replication is not in need. the workaround we take here is to exclude DDL from replication.  
Exclude DDL from replication  
```
declare 
  v_localDB varchar2(36) :='';
  cursor all_captures is 
  select capture_name, queue_name from dba_capture order by capture_name;
  cur_capture_row all_captures%ROWTYPE;
  
begin    
  select global_name into v_localDB from global_name;
  for cur_capture_row IN all_captures LOOP 
    DBMS_STREAMS_ADM.ADD_SCHEMA_RULES(
   schema_name =>'OWNER',
   streams_type =>'CAPTURE',
   streams_name=>cur_capture_row.capture_name,
   queue_name=>cur_capture_row.queue_name,
   include_dml=>false,
   include_ddl =>true,
   source_database =>v_localDB,
   inclusion_rule =>false);
   end loop;
end;
/
```
verify is DDL replication exists at schema level:   
```
select RULE_SET_TYPE,STREAMS_RULE_TYPE,RULE_TYPE,count(*) 
from dba_streams_rules 
group by RULE_SET_TYPE,STREAMS_RULE_TYPE,RULE_TYPE
order by RULE_SET_TYPE,STREAMS_RULE_TYPE,RULE_TYPE;
```

# 2- batch operation  
Here what we are trying to do is importing configuration into a set of staging tables on a node, which participates in 4-way replication.   
then activate this set of configuration.   
The activation will first check if changes being pushed to all nodes, make sure latency is less than 1 minutes before activativtion.   

This process sounds straightforward, while in the actual implementation, we got issues    
```
1. apply error occurred in other nodes, which cannot being fixed. dba_apply_error shows following message:  ORA-26753: Mismatched columns found in 'OWNER.TABLE_NAME'
   sometimes re-apply error pass it, sometime re-apply does not work. 
Streams indicates there is conflicting columns, but there is not.  We opened ticket with Oracle Support, after rounds of troubleshooting togethere, there is no results produced. 
the workaround we take is to remove nodes from replication, import and activate configuration on a standalone node, then add other nodes back to replication. 

2. latency, the date column oracle relies on for latency calculation was not updated in time. workaround is same as 1st issue.  
```
Side note here is: avoid batch operation when there is replication in the way   

