---
layout: post
title: Cassandra - Performance 
category: Cassandra
tags: [Cassandra]
---

Let's take a look at:  
+ performance goals  
+ monitor performance - nodetool subcommands  
+ Analyze performance - trace

### performance goals  
When planning a cluster, it's important to understand how the cluster will be used, like: 
```
number of clients, 
intended usage patterns, 
expected peak hours 
```

Before beginning any performance tuning effort, it is important to have clear goals in mind, whether you are just getting started on deploying an application on Cassandra, or maintaining an existing application. The performance goals includs:  
```
throughput (number of queries serviced per unit time)
latency(the time to complete a given query)
```
An example: The cluster must support 30,000 read operations per second from the available_​rooms_by_hotel_date table with a 95th percentile read latency of 3 ms.  

Regardless of your performance goals, It's important to remember that performance tuning is all about trade-offs, having well-defined performance goals help you articulate what trade-offs are acceptable for your application.  
+ Enabling SSTable compression in order to conserve disk space, at the cost of additional CPU processing 
+ throttling network usage and threads, which can be used to keep network and CPU utilization under control, at the cost of reduced throughput and increased latency 
+ Adjusting number of threads allocated to specific tasks such as reads, writes, or compaction in order to affect the priority relative to other tasks or support additional clients. 
+ Increase heap size in order to decrease query time. 

### monitor performance 
A good practice it to take frequent baselines to measure the performance of your cluster against its goals
+ nodetool tpstats
+ nodetool tablestats 
+ nodetool proxyhistograms
+ nodetool tablehistograms 

#### tpstats 
tpstats provides information on the thread pools that Cassandra maintains. Cassandra employs a SEDA - Staged Event Driven Architecture internally.  

```
[root@centos-1 ~]# nodetool tpstats
Pool Name                         Active   Pending      Completed   Blocked  All time blocked
ReadStage                              0         0        3662688         0                 0
MiscStage                              0         0              0         0                 0
CompactionExecutor                     0         0         928525         0                 0
MutationStage                          0         0         559726         0                 0
MemtableReclaimMemory                  0         0            467         0                 0
PendingRangeCalculator                 0         0              3         0                 0
GossipStage                            0         0        1948081         0                 0
SecondaryIndexManagement               0         0              0         0                 0
HintsDispatcher                        0         0             19         0                 0
RequestResponseStage                   0         0        2273195         0                 0
Native-Transport-Requests              0         0        6263312         0                 0
ReadRepairStage                        0         0         263840         0                 0
CounterMutationStage                   0         0              0         0                 0
MigrationStage                         0         0              4         0                 0
MemtablePostFlush                      0         0            865         0                 0
PerDiskMemtableFlushWriter_0           0         0            467         0                 0
ValidationExecutor                     0         0            192         0                 0
Sampler                                0         0              0         0                 0
MemtableFlushWriter                    0         0            467         0                 0
InternalResponseStage                  0         0          41903         0                 0
ViewMutationStage                      0         0              0         0                 0
AntiEntropyStage                       0         0            650         0                 0
CacheCleanupExecutor                   0         0              0         0                 0

Message type           Dropped
READ                         1
RANGE_SLICE                  0
_TRACE                       0
HINT                        82
MUTATION                     0
COUNTER_MUTATION             0
BATCH_STORE                  0
BATCH_REMOVE                 0
REQUEST_RESPONSE             0
PAGED_RANGE                  0
READ_REPAIR                  0
```
The top portion of the output presents data on tasks in each of Cassandra’s thread pools. You can see directly how many operations are in what stage, and whether they are active, pending, or completed. If this command is run during a write operation, there will be an active task in the MutationStage.  
The bottom portion of the output indicates the number of dropped messages for the node. Dropped messages are an indicator of Cassandra’s load shedding implementation, which each node uses to defend itself when it receives more requests than it can handle. For example, internode messages that are received by a node but not processed within the rpc_timeout are dropped, rather than processed, as the coordinator node will no longer be waiting for a response.  
Seeing lots of zeros in the output for blocked tasks and dropped messages means that you either have very little activity on the server or that Cassandra is doing an exceptional job of keeping up with the load. Lots of non-zero values is indicative of situations where Cassandra is having a hard time keeping up.   

#### tablestats  
It was prior known as cfstats. It shows overview statistics for keyspace and tables.  
It shows read and write latency and total number of reads and writes at the keyspace and table level. It also shows detailed information about Cassandra’s internal structures for each table, including memtables, Bloom filters and SSTables.
```
[root@centos-1 ~]# nodetool tablestats testkeyspace.test_table
Total number of tables: 106
----------------
Keyspace : testkeyspace
        Read Count: 4046045
        Read Latency: 0.06688233867888271 ms
        Write Count: 9974
        Write Latency: 0.046226288349709245 ms
        Pending Flushes: 0
                Table: test_table
                SSTable count: 3
                Space used (live): 59795026
                Space used (total): 59795026
                Space used by snapshots (total): 0
                Off heap memory used (total): 105672
                SSTable Compression Ratio: 0.8016660848572847
                Number of partitions (estimate): 31549
                Memtable cell count: 0
                Memtable data size: 0
                Memtable off heap memory used: 0
                Memtable switch count: 1
                Local read count: 0
                Local read latency: NaN ms
                Local write count: 231
                Local write latency: NaN ms
                Pending flushes: 0
                Percent repaired: 55.34
                Bloom filter false positives: 0
                Bloom filter false ratio: 0.00000
                Bloom filter space used: 62728
                Bloom filter off heap memory used: 62704
                Index summary off heap memory used: 34496
                Compression metadata off heap memory used: 8472
                Compacted partition minimum bytes: 643
                Compacted partition maximum bytes: 1597
                Compacted partition mean bytes: 1585
                Average live cells per slice (last five minutes): NaN
                Maximum live cells per slice (last five minutes): 0
                Average tombstones per slice (last five minutes): NaN
                Maximum tombstones per slice (last five minutes): 0
                Dropped Mutations: 0
```

#### proxyhistograms  
shows the latency of reads, writes, and range requests for which the requested node has served as the coordinator.  
The output is expressed in terms of percentile rank as well as minimum and maximum values in microseconds. Running this command on multiple nodes can help identify the presence of slow nodes in the cluster. A large range latency (in the hundreds of milliseconds or more) can be an indicator of clients using inefficient range queries, such as those requiring the ALLOW FILTERING clause or index lookups.  
```
[root@centos-1 ~]# nodetool proxyhistograms
proxy histograms
Percentile       Read Latency      Write Latency      Range Latency   CAS Read Latency  CAS Write Latency View Write Latency
                     (micros)           (micros)           (micros)           (micros)           (micros)           (micros)
50%                   1955.67               0.00            5839.59               0.00               0.00               0.00
75%                   2816.16               0.00           10090.81               0.00               0.00               0.00
95%                  12108.97               0.00           36157.19               0.00               0.00               0.00
98%                  14530.76               0.00           36157.19               0.00               0.00               0.00
99%                  20924.30               0.00           36157.19               0.00               0.00               0.00
Min                    545.79               0.00             545.79               0.00               0.00               0.00
Max                  20924.30               0.00           36157.19               0.00               0.00               0.00
```
#### tablehistograms 
proxyhistograms is useful for identifying general performance issues, tablehistograms focus on performance on specific tables.  
the output omits the range latency and instead provide counts of SSTables read per query, partition size, cell count.   
```
[root@centos-1 ~]# nodetool tablehistograms testkeyspace test_table 
testkeyspace/test_table histograms
Percentile  SSTables     Write Latency      Read Latency    Partition Size        Cell Count
                              (micros)          (micros)           (bytes)
50%             0.00              0.00              0.00              1109                 4
75%             0.00              0.00              0.00              3311                 4
95%             0.00              0.00              0.00              3311                 4
98%             0.00              0.00              0.00              3311                 4
99%             0.00              0.00              0.00              3311                 4
Min             0.00              0.00              0.00               643                 4
Max             0.00              0.00              0.00             20501                 4
```

Note the metrics reported are lifetime metrics since the node was started. to reset the metrics on a node, you have to restart it.  
Once you’ve gained familiarity with these metrics and what they tell you about your cluster, you should identify key metrics to monitor and even implement automated alerts that indicate your performance goals are not being met. You can accomplish this via DataStax OpsCenter or any JMX-based metrics framework.  

### analyze performance 
When there is a performance issue, to narrow down to specific tables and qven queries, to find the root cause. 
The quality of data model is usually the most influential factor in the performance of Cassandra cluster. 

Trace will show activities such as preparing statements, read repair, key cache searches, data lookups in memtables and SSTables, interactions between nodes, and the time associated with each step in microseconds.
```
cqlsh:testkeyspace> TRACING ON
cqlsh:testkeyspace> SELECT count(*) from test_table;
cassandra@cqlsh:testkeyspace> select count(*) from test_table;

 count
-------
  1768

(1 rows)

Warnings :
Aggregation query used without partition key


Tracing session: 508daac0-5c88-11e9-91f1-eb932b10e229

 activity                                                                                                                                          | timestamp                  | source       | source_elapsed | client
---------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+--------------+----------------+--------------
                                                                                                                                Execute CQL3 query | 2019-04-11 21:33:34.828000 | 10.247.99.28 |              0 | 10.247.99.28
                                                                READ message received from /10.247.99.28 [MessagingService-Incoming-/10.247.99.28] | 2019-04-11 21:05:32.136000 | 10.247.99.31 |             58 | 10.247.99.28
                                                                                           Executing single-partition query on roles [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |            486 | 10.247.99.28
                                                                                                        Acquiring sstable references [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |            569 | 10.247.99.28
                                                           Skipped 0/2 non-slice-intersecting sstables, included 0 due to tombstones [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |            689 | 10.247.99.28
                                                                                                         Key cache hit for sstable 1 [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |            823 | 10.247.99.28
                                                                                                         Key cache hit for sstable 2 [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |            897 | 10.247.99.28
                                                                                           Merged data from memtables and 2 sstables [ReadStage-1] | 2019-04-11 21:05:32.137000 | 10.247.99.31 |           1039 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-1] | 2019-04-11 21:05:32.137001 | 10.247.99.31 |           1117 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-1] | 2019-04-11 21:05:32.137001 | 10.247.99.31 |           1162 | 10.247.99.28
                                                                                                 Enqueuing response to /10.247.99.28 [ReadStage-1] | 2019-04-11 21:05:32.137001 | 10.247.99.31 |           1195 | 10.247.99.28
                                                 Sending REQUEST_RESPONSE message to /10.247.99.28 [MessagingService-Outgoing-/10.247.99.28-Small] | 2019-04-11 21:05:32.138000 | 10.247.99.31 |           1553 | 10.247.99.28
                                                                READ message received from /10.247.99.28 [MessagingService-Incoming-/10.247.99.28] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |              9 | 10.247.99.28
                                                                                           Executing single-partition query on roles [ReadStage-1] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |             92 | 10.247.99.28
                                                                                                        Acquiring sstable references [ReadStage-1] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |            127 | 10.247.99.28
                                                           Skipped 0/2 non-slice-intersecting sstables, included 0 due to tombstones [ReadStage-1] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |            168 | 10.247.99.28
                                                                                                         Key cache hit for sstable 1 [ReadStage-1] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |            215 | 10.247.99.28
                                                                                                         Key cache hit for sstable 2 [ReadStage-1] | 2019-04-11 21:05:32.143000 | 10.247.99.31 |            263 | 10.247.99.28
                                                                                           Merged data from memtables and 2 sstables [ReadStage-1] | 2019-04-11 21:05:32.143001 | 10.247.99.31 |            339 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-1] | 2019-04-11 21:05:32.143001 | 10.247.99.31 |            384 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-1] | 2019-04-11 21:05:32.143001 | 10.247.99.31 |            419 | 10.247.99.28
                                                                                                 Enqueuing response to /10.247.99.28 [ReadStage-1] | 2019-04-11 21:05:32.144000 | 10.247.99.31 |            460 | 10.247.99.28
                                                 Sending REQUEST_RESPONSE message to /10.247.99.28 [MessagingService-Outgoing-/10.247.99.28-Small] | 2019-04-11 21:05:32.144000 | 10.247.99.31 |            776 | 10.247.99.28
                                                                           Parsing select count(*) from ncso_config; [Native-Transport-Requests-1] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            197 | 10.247.99.28
                                                                                                 Preparing statement [Native-Transport-Requests-1] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            344 | 10.247.99.28
                                                                                   reading digest from /10.247.99.31 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            565 | 10.247.99.28
                                                                                           Executing single-partition query on roles [ReadStage-3] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            808 | 10.247.99.28
                                                                                                        Acquiring sstable references [ReadStage-3] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            887 | 10.247.99.28
                                                             Sending READ message to /10.247.99.31 [MessagingService-Outgoing-/10.247.99.31-Small] | 2019-04-11 21:33:34.829000 | 10.247.99.28 |            883 | 10.247.99.28
                                                           Skipped 0/1 non-slice-intersecting sstables, included 0 due to tombstones [ReadStage-3] | 2019-04-11 21:33:34.829001 | 10.247.99.28 |           1068 | 10.247.99.28
                                                                                                         Key cache hit for sstable 1 [ReadStage-3] | 2019-04-11 21:33:34.830000 | 10.247.99.28 |           1164 | 10.247.99.28
                                                                                           Merged data from memtables and 1 sstables [ReadStage-3] | 2019-04-11 21:33:34.830000 | 10.247.99.28 |           1343 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-3] | 2019-04-11 21:33:34.830000 | 10.247.99.28 |           1443 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.31 [MessagingService-Incoming-/10.247.99.31] | 2019-04-11 21:33:34.834000 | 10.247.99.28 |           5375 | 10.247.99.28
                                                                                   Processing response from /10.247.99.31 [RequestResponseStage-2] | 2019-04-11 21:33:34.834000 | 10.247.99.28 |           5890 | 10.247.99.28
                                                                                   reading digest from /10.247.99.31 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.835000 | 10.247.99.28 |           6430 | 10.247.99.28
                                                             Sending READ message to /10.247.99.31 [MessagingService-Outgoing-/10.247.99.31-Small] | 2019-04-11 21:33:34.835000 | 10.247.99.28 |           6804 | 10.247.99.28
                                                                                           Executing single-partition query on roles [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7167 | 10.247.99.28
                                                                                                        Acquiring sstable references [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7245 | 10.247.99.28
                                                           Skipped 0/1 non-slice-intersecting sstables, included 0 due to tombstones [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7350 | 10.247.99.28
                                                                                                         Key cache hit for sstable 1 [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7430 | 10.247.99.28
                                                                                           Merged data from memtables and 1 sstables [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7573 | 10.247.99.28
                                                                                              Read 1 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:34.836000 | 10.247.99.28 |           7657 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.31 [MessagingService-Incoming-/10.247.99.31] | 2019-04-11 21:33:34.840000 | 10.247.99.28 |          11856 | 10.247.99.28
                                                                                   Processing response from /10.247.99.31 [RequestResponseStage-2] | 2019-04-11 21:33:34.841000 | 10.247.99.28 |          12211 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.842000 | 10.247.99.28 |          13279 | 10.247.99.28
                      Submitting range requests on 769 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.842000 | 10.247.99.28 |          13683 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.844000 | 10.247.99.28 |          15612 | 10.247.99.28
                                     Executing seq scan across 4 sstables for (min(-9223372036854775808), min(-9223372036854775808)] [ReadStage-2] | 2019-04-11 21:33:34.844000 | 10.247.99.28 |          16125 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:34.847000 | 10.247.99.28 |          18125 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.849000 | 10.247.99.28 |          20639 | 10.247.99.28
                      Submitting range requests on 714 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.849000 | 10.247.99.28 |          20888 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.851000 | 10.247.99.28 |          22178 | 10.247.99.28
 Executing seq scan across 4 sstables for (tnt_ZreOdbfucyXkiBgwTvNDFSlHIt:SDC_CredentialsForDistribution, min(-9223372036854775808)] [ReadStage-2] | 2019-04-11 21:33:34.852000 | 10.247.99.28 |          23133 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:34.854000 | 10.247.99.28 |          25915 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.856000 | 10.247.99.28 |          27192 | 10.247.99.28
                      Submitting range requests on 649 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.856000 | 10.247.99.28 |          27442 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.861000 | 10.247.99.28 |          32716 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_ZHNRlEezTjxJArkXUDScpdBYsI:SDC_Configuration, min(-9223372036854775808)] [ReadStage-4] | 2019-04-11 21:33:34.862000 | 10.247.99.28 |          33524 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-4] | 2019-04-11 21:33:34.864000 | 10.247.99.28 |          35433 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.865000 | 10.247.99.28 |          36902 | 10.247.99.28
                      Submitting range requests on 592 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.865000 | 10.247.99.28 |          37102 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.867000 | 10.247.99.28 |          38411 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_WalEUdJZMPTSiRkfGbuIDOKAys:SDC_Configuration, min(-9223372036854775808)] [ReadStage-6] | 2019-04-11 21:33:34.867000 | 10.247.99.28 |          38548 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-6] | 2019-04-11 21:33:34.869000 | 10.247.99.28 |          40918 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.871000 | 10.247.99.28 |          42515 | 10.247.99.28
                      Submitting range requests on 555 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.871000 | 10.247.99.28 |          42836 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.872000 | 10.247.99.28 |          43952 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_VrRFQbuvHlhdoxnOtMipYfPzBg:SDC_Configuration, min(-9223372036854775808)] [ReadStage-4] | 2019-04-11 21:33:34.872000 | 10.247.99.28 |          44098 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-4] | 2019-04-11 21:33:34.876000 | 10.247.99.28 |          47195 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.881000 | 10.247.99.28 |          52660 | 10.247.99.28
                      Submitting range requests on 505 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.881000 | 10.247.99.28 |          52956 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.883000 | 10.247.99.28 |          54390 | 10.247.99.28
                     Executing seq scan across 4 sstables for (TNT_sOfGRaICWYwVdUTwXPOXqswj11:MockConfig, min(-9223372036854775808)] [ReadStage-5] | 2019-04-11 21:33:34.883000 | 10.247.99.28 |          54546 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-5] | 2019-04-11 21:33:34.886000 | 10.247.99.28 |          57887 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.889000 | 10.247.99.28 |          60281 | 10.247.99.28
                      Submitting range requests on 459 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.889000 | 10.247.99.28 |          60567 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.890000 | 10.247.99.28 |          61886 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_FYfvbMnZLmlAHgGeBKNCJukXEP:SDC_Configuration, min(-9223372036854775808)] [ReadStage-4] | 2019-04-11 21:33:34.890000 | 10.247.99.28 |          62025 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-4] | 2019-04-11 21:33:34.893000 | 10.247.99.28 |          64898 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.894000 | 10.247.99.28 |          66105 | 10.247.99.28
                      Submitting range requests on 421 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.895000 | 10.247.99.28 |          66387 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.896000 | 10.247.99.28 |          67599 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_tkvQlDAORXMUygGdLueYWNisqC:SDC_Configuration, min(-9223372036854775808)] [ReadStage-2] | 2019-04-11 21:33:34.896000 | 10.247.99.28 |          68056 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:34.903000 | 10.247.99.28 |          74191 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.904000 | 10.247.99.28 |          75570 | 10.247.99.28
                      Submitting range requests on 380 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.904000 | 10.247.99.28 |          75819 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.905000 | 10.247.99.28 |          76771 | 10.247.99.28
 Executing seq scan across 4 sstables for (tnt_hMVSuzmIKXjFGYxHbtATlRJdaW:SDC_CredentialsForDistribution, min(-9223372036854775808)] [ReadStage-4] | 2019-04-11 21:33:34.905000 | 10.247.99.28 |          76925 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-4] | 2019-04-11 21:33:34.907000 | 10.247.99.28 |          79049 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.909000 | 10.247.99.28 |          80576 | 10.247.99.28
                      Submitting range requests on 331 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.909000 | 10.247.99.28 |          80798 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.911000 | 10.247.99.28 |          82139 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.911000 | 10.247.99.28 |          82722 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:34.911000 | 10.247.99.28 |          83021 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:34.928000 | 10.247.99.28 |          99574 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-3] | 2019-04-11 21:33:34.940000 | 10.247.99.28 |         112025 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.942000 | 10.247.99.28 |         113607 | 10.247.99.28
                      Submitting range requests on 293 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.942000 | 10.247.99.28 |         113849 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.943000 | 10.247.99.28 |         114865 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.944000 | 10.247.99.28 |         115136 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:34.944000 | 10.247.99.28 |         115236 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:34.952000 | 10.247.99.28 |         123755 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-3] | 2019-04-11 21:33:34.953000 | 10.247.99.28 |         124366 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.954000 | 10.247.99.28 |         125540 | 10.247.99.28
                      Submitting range requests on 254 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.954000 | 10.247.99.28 |         125700 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.955000 | 10.247.99.28 |         126341 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.955000 | 10.247.99.28 |         126422 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:34.955000 | 10.247.99.28 |         126586 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:34.965000 | 10.247.99.28 |         136873 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-5] | 2019-04-11 21:33:34.966000 | 10.247.99.28 |         137417 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.967000 | 10.247.99.28 |         138863 | 10.247.99.28
                      Submitting range requests on 219 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.967000 | 10.247.99.28 |         139013 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.968000 | 10.247.99.28 |         139608 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.968000 | 10.247.99.28 |         139689 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:34.968000 | 10.247.99.28 |         139857 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:34.983000 | 10.247.99.28 |         154892 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-5] | 2019-04-11 21:33:34.984000 | 10.247.99.28 |         155487 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:34.985000 | 10.247.99.28 |         157066 | 10.247.99.28
                      Submitting range requests on 179 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:34.986000 | 10.247.99.28 |         157258 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:34.986000 | 10.247.99.28 |         157784 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:34.986000 | 10.247.99.28 |         157888 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:34.986000 | 10.247.99.28 |         158071 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:35.005000 | 10.247.99.28 |         176824 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-2] | 2019-04-11 21:33:35.006000 | 10.247.99.28 |         177138 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:35.007000 | 10.247.99.28 |         178664 | 10.247.99.28
                      Submitting range requests on 135 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:35.007000 | 10.247.99.28 |         178798 | 10.247.99.28
                                                                                  Enqueuing request to /10.247.99.26 [Native-Transport-Requests-1] | 2019-04-11 21:33:35.008000 | 10.247.99.28 |         179182 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:35.008000 | 10.247.99.28 |         179285 | 10.247.99.28
                                                      Sending RANGE_SLICE message to /10.247.99.26 [MessagingService-Outgoing-/10.247.99.26-Small] | 2019-04-11 21:33:35.008000 | 10.247.99.28 |         179488 | 10.247.99.28
                                                    REQUEST_RESPONSE message received from /10.247.99.26 [MessagingService-Incoming-/10.247.99.26] | 2019-04-11 21:33:35.020000 | 10.247.99.28 |         191774 | 10.247.99.28
                                                                                   Processing response from /10.247.99.26 [RequestResponseStage-2] | 2019-04-11 21:33:35.021000 | 10.247.99.28 |         192250 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:35.022000 | 10.247.99.28 |         193395 | 10.247.99.28
                      Submitting range requests on 100 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:35.022000 | 10.247.99.28 |         193532 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:35.022000 | 10.247.99.28 |         193791 | 10.247.99.28
              Executing seq scan across 4 sstables for (tnt_JKpPqrawAsxmkWvBdylDIfgeZX:SDC_Configuration, min(-9223372036854775808)] [ReadStage-2] | 2019-04-11 21:33:35.022000 | 10.247.99.28 |         193903 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:35.025000 | 10.247.99.28 |         196139 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:35.026000 | 10.247.99.28 |         198106 | 10.247.99.28
                       Submitting range requests on 53 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:35.027000 | 10.247.99.28 |         198204 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:35.027000 | 10.247.99.28 |         198450 | 10.247.99.28
                     Executing seq scan across 4 sstables for (tnt_bEUleovQyhANTwWCOjRkGfBgrq:MockConfig, min(-9223372036854775808)] [ReadStage-5] | 2019-04-11 21:33:35.027000 | 10.247.99.28 |         198552 | 10.247.99.28
                                                                                            Read 100 live rows and 0 tombstone cells [ReadStage-5] | 2019-04-11 21:33:35.029000 | 10.247.99.28 |         200543 | 10.247.99.28
                                                                                           Computing ranges to query [Native-Transport-Requests-1] | 2019-04-11 21:33:35.031000 | 10.247.99.28 |         202539 | 10.247.99.28
                       Submitting range requests on 16 ranges with a concurrency of 16 (6.3 rows per range expected) [Native-Transport-Requests-1] | 2019-04-11 21:33:35.031000 | 10.247.99.28 |         202645 | 10.247.99.28
                                                                               Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2019-04-11 21:33:35.031000 | 10.247.99.28 |         202732 | 10.247.99.28
                     Executing seq scan across 4 sstables for (tnt_CGzUajYHxfXBlrAqknVgWhNEsO:MockConfig, min(-9223372036854775808)] [ReadStage-2] | 2019-04-11 21:33:35.031000 | 10.247.99.28 |         202947 | 10.247.99.28
                                                                                             Read 68 live rows and 0 tombstone cells [ReadStage-2] | 2019-04-11 21:33:35.033000 | 10.247.99.28 |         204485 | 10.247.99.28
                                                                                                                                  Request complete | 2019-04-11 21:33:35.041371 | 10.247.99.28 |         213371 | 10.247.99.28
```
Cassandra stores query trace results in the system_traces keyspace. Cassandra limits the TTL on these tables to prevent them from filling up your disk over time. tracetype_query_ttl and tracetype_repair_ttl in the cassandra.yaml file.  
```
[root@centos1 conf]# grep trace *
cassandra.yaml:# TTL for different trace types used during logging of the repair process.
cassandra.yaml:tracetype_query_ttl: 86400   # 86400 seconds = 24 hours
cassandra.yaml:tracetype_repair_ttl: 604800

```

Configure individual nodes to trace some or all of their queries. This command  takes a number between 0.0 (the default) and 1.0, where 0.0 disables tracing and 1.0 traces every query. A typical trace session involves 10 or more writes. Setting a trace level of 1.0 could easily overload a busy cluster, so a value such as 0.01 or 0.001 is typically appropriate  
```
[root@centos-1 data]# nodetool gettraceprobability
Current trace probability: 0.0
[root@centos-1 data]# nodetool settraceprobability 0.01
```

### cheat sheet  
+ get all the table size in a keyspace   
nodetool tablestats -- testkeyspace | grep Table:  -A3  
+ check keyspace size   
```
[root@centos-1 data]# pwd
/data/cassandra/data/data
[root@centos-1 data]# du -sh *
85M     keyspace1
50M     keyspace2
97M     keyspace3
1.7M    system
124K    system_auth
1.1M    system_distributed
172K    system_schema
0       system_traces
14G     testkeyspace
```
+ when I trace a CQL query manually, the query string appears in system_traces.sessions.parameters  
cassandra@cqlsh:system_traces> select parameters from sessions;  
+ get table performance stats   
nodetool tablehistograms testkeyspace test_table  
  
### Q & A 
cell count in tablehistograms  



