---
layout: post
categories: kafka
title: "Kafka operation tips"
description: "Some troubleshooting advices about kafka"
keywords: "kafka, tips"
---
# Table Of Contents
* TOC
{:toc}

# How to increase topic Replication Factor in Kafka

## How to look current topic configuration

```
kafka-topics --topic f6b77500-a1f9-4c1f-8fe1-291ae28aff47  --describe --zookeeper localhost:2181
 
Topic:f6b77500-a1f9-4c1f-8fe1-291ae28aff47  PartitionCount:1    ReplicationFactor:1 Configs:retention.ms=2592000000,segment.ms=86400000
    Topic: f6b77500-a1f9-4c1f-8fe1-291ae28aff47 Partition: 0    Leader: 3   Replicas: 3 Isr: 3
```

We interested in following parameters:

* `ReplicationFactor` (count of replicas for each partition in this topic)
* `Isr` (ids of replicas which have latest actual data, same as `broker.id` in kafka config)
* `Leader` ('main' node for this partition)

*Example 1: ReplicationFactor=3, all nodes are available*

```
kafka-topics --topic 9146aeea-1b19-48a0-9001-c77e35d74721  --describe --zookeeper localhost:2181
 
Topic:9146aeea-1b19-48a0-9001-c77e35d74721  PartitionCount:1    ReplicationFactor:3 Configs:retention.ms=2592000000,segment.ms=86400000
    Topic: 9146aeea-1b19-48a0-9001-c77e35d74721 Partition: 0    Leader: 1   Replicas: 1,2,3 Isr: 1,3,2
```

*Example 2: ReplicationFactor=3, one node is unavailable*

```
kafka-topics --topic 9146aeea-1b19-48a0-9001-c77e35d74721  --describe --zookeeper localhost:2181
 
Topic:9146aeea-1b19-48a0-9001-c77e35d74721  PartitionCount:1    ReplicationFactor:3 Configs:retention.ms=2592000000,segment.ms=86400000
    Topic: 9146aeea-1b19-48a0-9001-c77e35d74721 Partition: 0    Leader: 1   Replicas: 1,2,3 Isr: 1,3
```

## How to change topic Replication Factor

1) Need to create JSON file with new topic configuration, for example:

```
{"partitions":                       
    [{"topic": "f6b77500-a1f9-4c1f-8fe1-291ae28aff47",                   
      "partition": 0,                   
      "replicas": [1,2,3] }],           
"version":1                          
}
```

and store it somewhere on Kafka server (`/tmp/reassign.json` in my case)

2) Then need to apply this file

```
kafka-reassign-partitions --zookeeper localhost:2181 --execute --reassignment-json-file /tmp/reassign.json
 
Current partition replica assignment
 
{"version":1,"partitions":[{"topic":"f6b77500-a1f9-4c1f-8fe1-291ae28aff47","partition":0,"replicas":[3]}]}
 
Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions {"version":1,"partitions":[{"topic":"f6b77500-a1f9-4c1f-8fe1-291ae28aff47","partition":0,"replicas":[1,2,3]}]}
```

3) Check how reassignment is going

```
kafka-reassign-partitions --zookeeper localhost:2181 --verify --reassignment-json-file /tmp/reassign.json
 
Status of partition reassignment:
Reassignment of partition [f6b77500-a1f9-4c1f-8fe1-291ae28aff47,0] completed successfully
```

# How to increase partition count

```
kafka-topics --zookeeper localhost:2181 --alter --partitions 5 --topic 9bd6a5bf-5fdb-4900-91af-c7b6d560968f
```

# How to fix 'Leader: -1' for Kafka partition

## How it looks like

```
kafka-topics --topic _schemas --describe --zookeeper localhost:2181
Topic:_schemas    PartitionCount:1    ReplicationFactor:3    Configs:cleanup.policy=compact
    Topic: _schemas    Partition: 0    Leader: -1    Replicas: 1,2,3    Isr:
```

That partition has no In-Sync Replicas (Isr) and cannot elect a Leader. Usually, that means we can't read/write from/to this partition.

## How to fix it

TIP. We can get a list of all topics that have partitions without Leader with this one-liner:

```
kafka-topics --describe --zookeeper localhost:2181 | grep "Leader: -1" | cut -f 2 | uniq | cut -f 2 -d " "
```

Kafka can do Unclean Leader Election. By default, Kafka elects partition leader from in-sync replicas, to guarantee data consistency. But, when the partition has no in-sync replicas, we can switch to a special mode, which can elect a leader from the random available replica. This is the dangerous operation, because we may lose some data, but this is better than nothing.

1) Enable `unclean.leader.election.enable=false` for topic:

```
kafka-topics --config unclean.leader.election.enable=true --zookeeper localhost:2181 --topic _schemas --alter
Updated config for topic "_schemas".
```

2) Restart Kafka

```
systemctl restart kafka
```

3) Check that leader was elected and in-sync replicas was appeared

```
kafka-topics --topic _schemas --describe --zookeeper localhost:2181
Topic:_schemas    PartitionCount:1    ReplicationFactor:3    Configs:unclean.leader.election.enable=false,cleanup.policy=compact
    Topic: _schemas    Partition: 0    Leader: 1    Replicas: 1,2,3    Isr: 3,1,2
```

4) Disable `unclean.leader.election` back

```
kafka-topics --config unclean.leader.election.enable=false --zookeeper localhost:2181 --topic _schemas --alter
Updated config for topic "_schemas".
```

5) And restart Kafka again

```
systemctl restart kafka
```