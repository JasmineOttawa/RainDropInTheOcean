---
layout: post
title: Oracle - SHMALL 
category: Oracle
tags: [Oracle]
---

# Question 
Tuning oracle 12c on RHEL7 got error ORA-27104. same tuning process works fine on RHEL6.   

# Investigation 
SHMMAX and SHMALL are two key shared memory parameters that directly impact’s the way by which Oracle creates an SGA. Shared memory is nothing but part of Unix IPC System (Inter Process Communication) maintained by kernel where multiple processes share a single chunk of memory to communicate with each other.  

While trying to create an SGA during a database startup, Oracle chooses from one of the 3 memory management models   
a) one-segment or   
b) contiguous-multi segment or   
c) non-contiguous multi segment.   
Adoption of any of these models is dependent on the size of SGA and values defined for the shared memory parameters in the linux kernel, most importantly SHMMAX.

SHMMAX is the maximum size of a single shared memory segment set in “bytes”.  
SHMALL is the total size of Shared Memory Segments System wide set in “pages”.  

SHMALL is the total size of Shard Memory Segments System wide, it should always be less than the Physical Memory on the System and should be greater than sum of SGA’s of all the oracle databases on the server.   
when shmall is less than the total SGA, startup will throw error message :   
ORA-27102: out of memory # on 11g   
ORA-27104: system-defined limits for shared memory was misconfigured # on 12c  

When create database, you may get following message:  
INFO: Error Message:PRVG-1201 : OS kernel parameter “shmall” does not have expected configured value on node “…” [Expected = “13172980” ; Current = “2097152”; Configured = “2097152”]  

In RHEL6, default value of shmall is set big enough, almost no limit for shared memory.   
kernel.shmmax = 68719476736  
kernel.shmall = 4294967296 #means 16384GB on an node with pagesize = 4096  

While in RHEL7, default value of shmall is small, when tuning a node with large memory without setting this value, oracle throws ORA-27104
kernel.shmall = 2097152 #means 8G on an node with pagesize = 4096   
kernel.shmmax = 67499231232  

In RHEL7, we need to calculate of shmall : total_sga in bytes / pagesize , and set it up in /etc/sysctl.conf   
shmmni=getconf PAGE_SIZE  
shmall=echo ${oramem}*1024*1024/${shmmni} | bc  