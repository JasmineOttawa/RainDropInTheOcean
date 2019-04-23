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
Commit log is a short-time storage that helps ensure that data is not lost if a node crashes or is shut down before memtables can be flushed to disk.   
because then a node is restarted, the commit log gets replayed. In fact, that's the only time the commit log is read.   
commitlog_segment_size_in_mb , default 32MB,   
commitlog_total_​space_in_mb, total space allocated to commit log. larger value means Cassandra will need to flush tables to disk less frequently   
commitlog_compression, supported algorithms include LZ4, Snappy, Deflate. compression comes at the cost of additional CPU time   
commitlog_sync, periodic - commitlog_sync_period_in_ms, defaults to 10000ms=10 seconds, you can potentially lose data that has not yet been synced to disk from write-behind cache.    
                batch - it will block until the write is synced to disk, needs another parameter commit_log_sync_batch_window_in_ms to set time between each sync effort.  

### SSTables    
Cassandra writes SSTable files to disk asynchronously.  
file_cache_size_in_mb, when reading SSTable files from disk, use buffer cache/pool to help reduce I/O. it uses off-heap memory   
buffer_pool_use_heap_if_exhausted=true, to use heap memory once the off-heap cache is full   

index_summary_capacity_in_mb, 5% of heap to store index summary   
index_​summary_​resize_interval_in_minutes=60.  read rates may change over time, Cassandra also resamples indexes from files stored on disk at this frequency  

min_index_​interval, per table setting, influence the relative amount of space allocated to indexes for different tables.   
max_index_interval  
 
### hinted handoff    
A coordinate node can keep a copy of data on behalf of a node that is down for some amount of time.   
hinted_handoff_throttle_in_kb | nodetool sethintedhandoffthrottlekb, default 1024, that 1MB/s   
in a cluster of three nodes, each of the two nodes delivering hints to a third node would throttle its hint delivery to half of this value, or 512KB/second.   

max_hints_file_size_in_mb, the amount of disk for hints,  /data/hints    
hints_directory, /data/hints     
max_hint_window_in_ms, hints expires after this time    
nodetool truncatehints IP1,IP2...     

### compaction 
SizeTieredCompactionStrategy
LeveledCompactionStrategy
DateTieredCompactionStrategy

### concurrency and threading 
concurrent_reads , how many simultaneous read requests the node can service, defaults to 32. should be set to the number of drives used for data storage × 16
concurrent_writes, should correlate to the number of clients that will write concurrently to the server. If Cassandra is backing a web application server, you can tune this setting from its default of 32 to match the number of threads the application server has available to connect to Cassandra.  
concurrent_counter_writes,  counter and materialized view writes both involve a read before write, it is best to set this to the lower of concurrent_reads and concurrent_writes.  
concurrent_materialized_view_writes, 

max_hints_delivery_threads, Maximum number of threads devoted to hint delivery  
memtable_flush_writers, Number of threads devoted to flushing memtables to disk  
concurrent_compactors,  Number of threads devoted to running compaction  
native_transport_max_threads, Maximum number of threads devoted to processing incoming CQL requests   
     (you may also notice the similar properties rpc_min_threads and rpc_​max_​threads, which are for the deprecated Thrift interface)  

### Networking and Timeouts
```
Property Name               Default Value       escription
read_request_timeout_in_ms  5000 (5 seconds)	How long the coordinator waits for read operations to complete
range_request_timeout_in_ms 10000 (10 seconds)	How long the coordinator should wait for range reads to complete
write_request_timeout_in_ms 2000 (2 seconds)	How long the coordinator should wait for writes to complete
counter_write_request_timeout_in_ms 5000 (5 seconds)	How long the coordinator should wait for counter writes to complete
cas_contention_timeout_in_ms 1000 (1 second)	How long a coordinator should continue to retry a lightweight transaction
truncate_request_timeout_in_ms 60000 (1 minute)	How long the coordinator should wait for truncates to complete (including snapshot)
streaming_socket_timeout_in_ms 3600000 (1 hour)	How long a node waits for streaming to complete
request_timeout_in_ms      10000 (10 seconds)	The default timeout for other, miscellaneous operations
```

cross_node_timeout, which defaults to false. If you have NTP configured in your environment, consider enabling this so that nodes can more accurately estimate when the coordinator has timed out on long-running requests and release resources more quickly.  
stream_throughput_outbound_megabits_per_sec, 
inter_dc_stream_throughput_outbound_megabits_per_sec, specify a per-thread throttle on streaming file transfers to other nodes in the local and remote data centers, respectively.
hinted_handoff_throttle_in_kb 
batchlog_​replay_​throttle_in_kb 
native_transport_max_frame_size_in_mb=256, Frame requests larger than this will be rejected by the node, limit traffic to the native CQL port on each node. 
native_transport_max_concurrent_connections_per_ip, To limit the number of simultaneous connections from a single source IP address, defaults to -1 (unlimited).

### JVM Settings
The Cassandra 3.0 release introduced another settings file in the conf directory called jvm.options. The purpose of this file is to move JVM settings related to heap size and garbage collection out of cassandra.in.sh file to a separate file, as these are the attributes that are tuned most frequently. The jvm.options and cassandra.in.sh are included (sourced) by the cassandra-env.sh file.  

HEAP: 
     if the machine has less than 1 GB of RAM, the heap is set to 50% of RAM.   
      If the machine has more than 4 GB of RAM, the heap is set to 25% of RAM, with a cap of 8 GB.   
	  To tune the minimum and maximum heap size, use the -Xms and -Xmx flags.   
	  These should be set to the same value to allow the entire heap to be locked in memory and not swapped out by the OS.   
	  It is not recommended to set the heap larger than 8GB if you are using the Concurrent Mark Sweep (CMS) garbage collector,as heap sizes larger than this value tend to lead to longer GC pauses.  
      In general, you’ll probably want to make sure that you’ve instructed the heap to dump its state if it hits an out-of-memory error, which is the default in cassandra-env.sh, set by the -XX:+HeapDumpOnOutOfMemory option.  
Garbage Collection:   
   The Java heap is broadly divided into two object spaces: young and old. The young space is subdivided into one for new object allocation (called “eden space”) and another for new objects that are still in use (the “survivor space”).  
   Cassandra uses the parallel copying collector in the young generation; this is set via the -XX:+UseParNewGC option. Older objects still have some reference, and have therefore survived a few garbage collections, so the Survivor Ratio is the ratio of eden space to survivor space in the young object part of the heap. Increasing the ratio makes sense for applications with lots of new object creation and low object preservation; decreasing it makes sense for applications with longer-living objects. Cassandra sets this value to 8 via the -XX:SurvivorRatio option, meaning that the ratio of eden to survivor space is 1:8 (each survivor space will be 1/8 the size of eden). This is fairly low, because the objects are living longer in the memtables.  
   Every Java object has an age field in its header, indicating how many times it has been copied within the young generation space. Objects are copied into a new space when they survive a young generation garbage collection, and this copying has a cost. Because long-living objects may be copied many times, tuning this value can improve performance. By default, Cassandra has this value set at 1 via the -XX:MaxTenuringThreshold option. Set it to 0 to immediately move an object that survives a young generation collection to the old generation. Tune the survivor ratio together along with the tenuring threshold.  
   By default, modern Cassandra releases use the Concurrent Mark Sweep (CMS) garbage collector for the old generation; this is set via the -XX:+UseConcMarkSweepGC option. This setting uses more RAM and CPU power to do frequent garbage collections while the application is running, in order to keep the GC pause time to a minimum. When using this strategy, it’s important to set the heap min and max values to the same value, in order to prevent the JVM from having to spend a lot of time growing the heap initially.  
   
G1GC, The Garbage First garbage collector, introduced in Java7， replacement for CMS GC, especially on multiprocessor machines with more memory.  
G1GC divides the heap into multiple equal size regions and allocates these to eden, survivor, and old generations dynamically, so that each generation is a logical collection of regions that need not be consecutive regions in memory. This approach enables garbage collection to run continually and require fewer of the “stop the world” pauses that characterize traditional garbage collectors.   
G1GC generally requires fewer tuning decisions; the intended usage is that you need only define the min and max heap size and a pause time goal. A lower pause time will cause GC to occur more frequently.  
G1GC was on course to become the default for the Cassandra 3.0 release, but was backed out as part of the JIRA issue CASSANDRA-10403, due to the fact that it did not perform as well as the CMS for heap sizes smaller than 8 GB.  
If you’d like to experiment with using G1GC and a larger heap size on a Cassandra 2.2 or later release, the settings are readily available in the cassandra-env.sh file (or jvm.options) file.  
Look for G1GC to become the default in a future Cassandra release, most likely in conjunction with support for Java 9, in which G1GC will become the Java default.  

gc_warn_threshold_in_ms, in cassandra.yaml file determines the pause length that will cause Cassandra  to generate log messages at the WARN logging level. This value defaults to 1000 ms (1 second).













