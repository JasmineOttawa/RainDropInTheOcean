---
layout: post
title: Cassandra - Monitoring via Prometheus and Grafana 
category: Cassandra
tags: [Cassandra]
---
https://blog.pythian.com/step-step-monitoring-cassandra-prometheus-grafana/

+ Monitor VM  
Install Prometheus  
Configure Prometheus  
Install Grafana  
+ Cassandra VMs  
Download prometheus JMX-Exporter  
Configure JMX-Exporter  
Configure Cassandra  
Restart Cassandra  

# on monitoring VM   
step1, Install Prometheus  
apt install prometheus 

step2, Configure Prometheus  
vi /etc/prometheus/prometheus.yaml
global:
  scrape_interval: 15s
 
scrape_configs:
# Cassandra config
  - job_name: 'cassandra'
    scrape_interval: 15s
    target_groups:
      - targets: ['cassandra01:7070', 'cassandra02:7070', 'cassandra03:7070']

Step3. Create storage and start Prometheus
cd /etc/prometheus
mkdir data
chown prometheus:prometheus data
prometheus --config.file=/stage/prometheus-2.3.1.linux-amd64/prometheus.yml
http://IP:9090 

	  
step4, Install Grafana  
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.1.4_amd64.deb
sudo apt-get install -y adduser libfontconfig
sudo dpkg -i grafana_5.1.4_amd64.deb
sudo service grafana-server start

# on Cassandra VMs  
step1, Download prometheus JMX-Exporter  
mkdir /opt/jmx_prometheus;cd /opt/jmx_prometheus;
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.0/jmx_prometheus_javaagent-0.3.0.jar

step2, Configure JMX-Exporter  
$ vi /opt/jmx_prometheus/cassandra.yml
use the sample config file here, monitor all metrics 
https://github.com/prometheus/jmx_exporter/blob/master/example_configs/cassandra.yml

step3, Configure Cassandra  
echo 'JVM_OPTS="$JVM_OPTS -javaagent:/opt/jmx_prometheus/jmx_prometheus_javaagent-0.3.0.jar=7070:/opt/jmx_prometheus/cassandra.yml"' &amp;gt;&amp;gt; conf/cassandra-env.sh

step4, Restart Cassandra  
systemctl restart dse 

-- another sample config file  /opt/jmx_prometheus/cassandra.yml  
```
lowercaseOutputName: true
lowercaseOutputLabelNames: true
whitelistObjectNames: [
"org.apache.cassandra.metrics:type=ColumnFamily,name=RangeLatency,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=LiveSSTableCount,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=SSTablesPerReadHistogram,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=SpeculativeRetries,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=MemtableOnHeapSize,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=MemtableSwitchCount,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=MemtableLiveDataSize,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=MemtableColumnsCount,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=MemtableOffHeapSize,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=BloomFilterFalsePositives,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=BloomFilterFalseRatio,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=BloomFilterDiskSpaceUsed,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=BloomFilterOffHeapMemoryUsed,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=SnapshotsSize,*",
"org.apache.cassandra.metrics:type=ColumnFamily,name=TotalDiskSpaceUsed,*",
"org.apache.cassandra.metrics:type=CQL,name=RegularStatementsExecuted,*",
"org.apache.cassandra.metrics:type=CQL,name=PreparedStatementsExecuted,*",
"org.apache.cassandra.metrics:type=Compaction,name=PendingTasks,*",
"org.apache.cassandra.metrics:type=Compaction,name=CompletedTasks,*",
"org.apache.cassandra.metrics:type=Compaction,name=BytesCompacted,*",
"org.apache.cassandra.metrics:type=Compaction,name=TotalCompactionsCompleted,*",
"org.apache.cassandra.metrics:type=ClientRequest,name=Latency,*",
"org.apache.cassandra.metrics:type=ClientRequest,name=Unavailables,*",
"org.apache.cassandra.metrics:type=ClientRequest,name=Timeouts,*",
"org.apache.cassandra.metrics:type=Storage,name=Exceptions,*",
"org.apache.cassandra.metrics:type=Storage,name=TotalHints,*",
"org.apache.cassandra.metrics:type=Storage,name=TotalHintsInProgress,*",
"org.apache.cassandra.metrics:type=Storage,name=Load,*",
"org.apache.cassandra.metrics:type=Connection,name=TotalTimeouts,*",
"org.apache.cassandra.metrics:type=ThreadPools,name=CompletedTasks,*",
"org.apache.cassandra.metrics:type=ThreadPools,name=PendingTasks,*",
"org.apache.cassandra.metrics:type=ThreadPools,name=ActiveTasks,*",
"org.apache.cassandra.metrics:type=ThreadPools,name=TotalBlockedTasks,*",
"org.apache.cassandra.metrics:type=ThreadPools,name=CurrentlyBlockedTasks,*",
"org.apache.cassandra.metrics:type=DroppedMessage,name=Dropped,*",
"org.apache.cassandra.metrics:type=Cache,scope=KeyCache,name=HitRate,*",
"org.apache.cassandra.metrics:type=Cache,scope=KeyCache,name=Hits,*",
"org.apache.cassandra.metrics:type=Cache,scope=KeyCache,name=Requests,*",
"org.apache.cassandra.metrics:type=Cache,scope=KeyCache,name=Entries,*",
"org.apache.cassandra.metrics:type=Cache,scope=KeyCache,name=Size,*",
"org.apache.cassandra.metrics:type=Client,name=connectedNativeClients,*",
"org.apache.cassandra.metrics:type=Client,name=connectedThriftClients,*",
"org.apache.cassandra.metrics:type=Table,name=WriteLatency,*",
"org.apache.cassandra.metrics:type=Table,name=ReadLatency,*",
"org.apache.cassandra.net:type=FailureDetector,*",
]
rules:
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(Connection|Streaming), scope=(\S*), name=(\S*)&amp;gt;&amp;lt;&amp;gt;(Count|Value)
    name: cassandra_$1_$3
    labels:
      address: "$2"
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(ColumnFamily), name=(RangeLatency)&amp;gt;&amp;lt;&amp;gt;(Mean)
    name: cassandra_$1_$2_$3
  - pattern: org.apache.cassandra.net&amp;lt;type=(FailureDetector)&amp;gt;&amp;lt;&amp;gt;(DownEndpointCount)
    name: cassandra_$1_$2
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(Keyspace), keyspace=(\S*), name=(\S*)&amp;gt;&amp;lt;&amp;gt;(Count|Mean|95thPercentile)
    name: cassandra_$1_$3_$4
    labels:
      "$1": "$2"
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(Table), keyspace=(\S*), scope=(\S*), name=(\S*)&amp;gt;&amp;lt;&amp;gt;(Count|Mean|95thPercentile)
    name: cassandra_$1_$4_$5
    labels:
      "keyspace": "$2"
      "table": "$3"
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(ClientRequest), scope=(\S*), name=(\S*)&amp;gt;&amp;lt;&amp;gt;(Count|Mean|95thPercentile)
    name: cassandra_$1_$3_$4
    labels:
      "type": "$2"
  - pattern: org.apache.cassandra.metrics&amp;lt;type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?,
      name=(\S*)&amp;gt;&amp;lt;&amp;gt;(Count|Value)
    name: cassandra_$1_$5
    labels:
      "$1": "$4"
      "$2": "$3"
```
Restart prometheus, to reload updated cassandra.yaml   
The output is pretty undiscriminating, how to highlight/set waring threshold on specific metrics?   

# in kubernetes 
[root@masternode]# kubectl get services -n common | grep prometheus
common-prometheus-alertmanager         ClusterIP   10.43.186.217   &amp;lt;none&amp;gt;        80/TCP           2d
common-prometheus-kube-state-metrics   ClusterIP   None            &amp;lt;none&amp;gt;        80/TCP           2d
common-prometheus-node-exporter        ClusterIP   None            &amp;lt;none&amp;gt;        9100/TCP         2d
common-prometheus-pushgateway          NodePort    10.43.177.43    &amp;lt;none&amp;gt;        9091:30714/TCP   2d
common-prometheus-server               ClusterIP   10.43.175.37    &amp;lt;none&amp;gt;        80/TCP           2d

http://10.43.177.43:9091 
http://10.43.177.43:30714 
curl http://localhost:30714 
curl http://127.0.0.1:30714  works 
curl http://10.247.136.75:30714 works 
Image:         prom/prometheus:v2.3.2




