---
layout: post
title: Cassandra - Performance (2)
category: Cassandra
tags: [Cassandra]
---

Take a look at Cassandra memory structure, following topics included:  
+ caching 
+ Memtables 
+ Commit Logs 
+ SSTables 
+ Hinted Handoff 
+ Compaction 
+ Concurrency and Threading 
+ Networking and Timeouts 
+ JVM Settings 
+ Cassandra-stress 

### caching  
+ key cache, stores a map of partition keys to row index entries, facilitating faster read access into SSTables stored on disk   
  It can be configured per-table : ALTER TABLE test_table WITH caching = { 'keys' : 'ALL', 'rows_per_partition' : 'NONE' };  
  by default it is enabled. key_cache_size_in_mb, default value is the small one of (5% of JVM heap, 100MB)  
+ row cache, stores entire rows, easily lead to more performance issues than it solves.  
  ALTER TABLE test_table WITH caching = { 'keys' : 'NONE', 'rows_per_partition' : '200' };  
  by default it is disabled.   
+ counter cache, improves counter performance by reducing lock contention for the most frequently accessed counters.   
  counter_cache_size_in_mb, default value is the small one of (2.5% of JVM heap, 50MB)  
  
caches are saved to disk periodically, so that they can be read on startup as a way to quickly warm the cache.   
+ saved_caches # directory   
+ key_cache_save_period # 14000, 4 hours   
+ row_cache_save_period # 0, disabled   
+ counter_cache_save_period # 7200, 2 hours   
  
Other cache properties/operations: 
+ key|row|counter_cache_keys_to_save #caches are indexed by key values  
+ invalidatekey|row|countercache     #clear cache   
+ setcachecapacity   
+ setcachekeystosave     
  
### Memtables  
Memtables could be stored either in heap, or off-heap memory  
memtable_heap_space_in_mb, default to 1/4 of heap size   
memtable_offheap_space_in_mb  
  
  memtable_allocation_type influence how cassandra allocates and manages memTable memory  
+ heap_buffers : default, allocate memtables on the heap using Jave New I/O  
+ offheap_buffers : allocate a portion of each memtable both on and off heap  
+ offheap_objects : use native memory directly, making cassandra entirely responsible for memory management and garbage collection.  
 
memtable_flush_writers, by defaul it is 2.   
If your data directories on SSD, increase this to the number of cores, without exceding max value of 8.   
  
memtable_flush_period_in_ms, sets the interval at which the memtable will be flushed to disk.  
This will result in more predictable write I/O, while more SSTables and more frequent compactions. by it is 0, disabled by default,   
and flushes will only occur based on the commit log threshold or memtable threshold being reached.   
  
### Commit Logs 

+ SSTables 


### check cassandra heap, cache size 







