# Streams For Apache Kafka - Message flow

Tutorials around Streams For Apache Kafka running on OCP

## Overview of tutorial

This tutorial will begin to describe how Kafka processes and stores data at a high level.

## Table of Contents

Tutorial listing

1. [Prereqs](#prerequisites)
2. [Tutorial Breakouts](#tutorials)
3. [Reference Docs](#reference-docs)

---

## Prerequisites

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

---

## Tutorial

It is very important to note that Kafka is designed to leverage the OS for IO and so depending on how your OS is configured these elements may change.

Kafka attempts to achieve data stability and availability to various forms of replication factors.  Meaning that data will be made available as replicas are available.  If you have a cluster with 3 replicas Kafka works to ensure that data is replicated first to os page cache and then to disk.  But before we get ahead of ourselves I want to talk abit about the flow of data in Kafka in general.

At a super high level the flow is

1.) Kafka Producers produce batches of data to be sent to the Kafka Broker
2.) The broker 'stores' this data and makes it available in its replicats
3.) The Consumers can then conumse this data from the broker.

Before we go any further I want to just talk a high level what replicas partition and isrs are.

TODO: Re org my approach here

Topics can be divided into multiple `partitions` whith each `partition` is basically a log of segements that store the action messages.  a partition has one leader and n followers.

`Replica`s is just a copy of a partition, which gets its data from the leader `partition`

And lastly, just to reiterate, an `ISR` is just a `replica` that is caught up to the leader `partition`

So if we take a look again at the bascis tutorial we have

```bash
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
```

This takes us finally to `min.isr` and `replicaiton.factor` but there is also `spec.kafka.replicas`

| Config | Description |
|-------|-------|
| spec.kafka.replicas | Number of Kafka Brokers in the cluster, also the maximum number of brokers possible |
| spec.kafka.config.default.replication.factor | default replication factor which means every new topic will have 3 replicas |
| spec.kafka.config.min.insync.replicas | minimum number of replicas that must be in-sync (ISR) for a partition to be considered available for writes  |

This means if you take the following:

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: mytopic
  namespace: kafka-test
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    min.insync.replicas: 2
```

You would get a total of 9 `replica`s

One of the more difficult questions to talk to is the specific cases in which data integrety is maintained in containerized Kafka specifically around how do we recover in single or multi node failures.

Expanding on the above #2) Kafka will store to OS Page cache

### Brief view of RAFT

This is especially true if users come from a pure RAFT implementation of a broker vs say the Apache Kafka approach.  To be clear Streams for apache kafka uses Apache Kafka through Strimzi (Strimzi works to make Apache Kafka container friendly).  But you may say that Strimzi supports RAFT.  Yes, sorta, but its Kraft and it is meant more leader election through metadata management and not the same thing as a RAFT replica quorum.

Before we move back to Apache Kafka and the main difference here I basically want to quickly highly RAFT and where the differenc is in related to storage.

RAFT is basically a gropu of servers which run the product to do leader elections, adding nodes, addressing data replication etc.

RAFT is also a state machine which themselves are a replicated log, which means RAFT itself depends on the log to be correct for the protocol to work.  This means that whatever leader is selected the other servers can delete part of the log.  That is unless you move from os page cache to disk itself through fsync.  This also means that fysnc is required for RAFT to work to handle failures.

### Now we are back to Apache Kafka

So reading that you might see that Apache Kafka doesn't need fsync's to be considered safe.  Its true, Apache Kafka writes to storage asynchronously to do this we really need to talk about how Apache Kafka handles recovery or termination of a node.

To recovery though we mean that if a node comes back that nodes broker will need to recover from a peer and a node that is recovering cannot be a candidate to be a leader until it has fully recovered the data.

That means when a broker is coming back online, it will determine how far behind it is, it will then from that offset pull that data it is missing and once it is caught up and fully functioning as a broker it can now become a leader if the need arises.

ISR are in-sync replicas.  This is a list of replicas that are considered caught up to the leader.  Now that replica can be a leader if election comes again.

TODO: need to add os page cache

TODO: need to add PVC Strimzi flavour to apache kafka

TODO: add dirty segement cleanup + compression

TODO: add consumer group scaling for partitions




