---
layout: post
title: Oracle - Streams Propagation process stuck 
category: Oracle
tags: [Oracle]
---

We have a 2-node streams replication setup on oracle 12i, and RMAN cron job setup for daily backup and archived redo log purge.  
Today I got a case reporting on high usage of archive redo log destination.  

### step1, check RMAN backup job log, it failed with message:   
```
ORA-19625: error identifying file /....../1_365_978019206.dbf 
```
It turns out log 365 is missing on the disk while exists in DB control file.  The inconsistency cause the backup job confusion and above message. 
it could be fixed by: 
```
RMAN target / 
crosscheck archivelog all; 
delete expired archivelog all;  # delete the archivelog records, which does not physically exists, from controlfile, 
```

### step2, check if streams affected by this missing log file.   
Following query shows long apply latency, and change on one node cannot be replicated to another node: 
```
select apply_name, DEQUEUE_TIME, DEQUEUED_MESSAGE_CREATE_TIME from v$streams_apply_reader order by apply_name; 
```
Propagation process look normal, when I try to restart it, It stucks. 
There is a known Bug 16994952 : UNABLE TO UNSCHEDULE PROPAGATION DUE TO CJQ0 SELF-DEADLOCK

### step3, work around for Bug 16994952 - propagation stuck
kill the propagation OS process 
```
stop capture: DBMS_CAPTURE_ADM.STOP_CAPTURE
set job_queue_process to 0: alter system set job_queue_processes=0;
Kill propagation process by: 
determine the J process associated with propagation:  select schema, qname, destination, schedule_disabled, process_name from dba_queue_schedules;
kill -9 J000 process
alter system set job_queue_processes=$original_value;
start propagation
start capture
```

### step4, WAITING FOR DICTIONARY REDO: FIRST SCN 
Now propagation throws message: WAITING FOR DICTIONARY REDO: FIRST SCN 
Find the missing redo logs, as they're being cleared from controlfile, re-register by: 
catalog start with '/oracle/g01/arch01/arch'; 

### step5, RMAN-08137: WARNING: archived log not deleted, needed for standby or upstream capture process
Now that Streams catch up, we need to remove stale archive log to alleviate disk pressure 

RMAN - delete archivelog until time "sysdate -5" failed with message: 
```
RMAN-08137: WARNING: archived log not deleted, needed for standby or upstream capture process
archived log file name=/....../1_400_978019206.dbf thread=1 sequence=400
```
dba_capture.REQUIRED_CHECKPOINT_SCN was used to tell if an archived redo log still need for Streams replication. 
By default, Streams checkpoints every 6 hours, to manually decrease this parameter temporarily:
```
exec dbms_propagation_adm.stop_propagation('PROPA');
execute dbms_capture_adm.stop_capture('CAPA');
execute DBMS_CAPTURE_ADM.SET_PARAMETER('CAPA', '_CKPT_RETENTION_CHECK_FREQ', '10'); # wait over 10s 
execute DBMS_CAPTURE_ADM.SET_PARAMETER('CAPA', '_CKPT_RETENTION_CHECK_FREQ', '21600'); # restore to original value 
exec dbms_propagation_adm.start_propagation('PROPA');
execute dbms_capture_adm.start_capture('CAPA'); 
```
