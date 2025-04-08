# Streams For Apache Kafka - Basics around Broker

Tutorials around Streams For Apache Kafka running on OCP

## Overview of tutorial

This tutorial will give a high level overview of the Broker yaml, we will also explore the general config options what the values are and what they imply in behaviors when setting them.

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

Here is an example of a Kafka cluster yaml.  We will use this to explore various components of it.

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka-test
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.8.0

    authorization:
      type: simple
    # The replicas field is required by the Kafka CRD schema while the KafkaNodePools feature gate is in alpha phase.
    # But it will be ignored when Kafka Node Pools are used
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: external
        port: 9094
        type: route
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.6"
    # The storage field is required by the Kafka CRD schema while the KafkaNodePools feature gate is in alpha phase.
    # But it will be ignored when Kafka Node Pools are used
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
  # The ZooKeeper section is required by the Kafka CRD schema while the UseKRaft feature gate is in alpha phase.
  # But it will be ignored when running in KRaft mode
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
  cruiseControl: {}
```

---

Lets start with the annotations section of this cluster yaml

```bash
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
```

Here to enable `Kraft` and `NodePools`.  Lets talk about `Kraft` first.

`Kraft` is Kafka with RAFT protocol instead of using ZooKeeper.  In Kafka there is something called a Quorum Controller that was introduced a few years ago.  Prior to this all cluster meta data was using ZooKeeper to maintain an accurate and replicated set cluster data that could be used to identify and select leaders among many other things.  This `Quorum Controller` was added so that new methods could be used to replace that process.  KRaft being more event driven means that the new controller would not need to load any / all state from ZooKeeper to beomce active.  When leadership change events occur the broker would already have that metadata recorded.


Ultiamtely one of the major downsides to a zookeeper backed kafka cluster is that all of this metadata would need to be loaded and prepared before partitions could be scaled effectively.  This approach dramatically increases thost types of events both in shutdown and recovery.

Version 3.9 recently implemented a dynamic quorum feature and is now the recommended / preferred way of configuring a KRaft based cluster.

`NodePools` are a convienent way of describing resources needs for your cluster.  They will need to be deployed before you deploy your cluster.

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: kafka
  namespace: kafka-test
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
    - controller
  storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
```

As I mention above a NodePool will overwrite what is put in your Cluster config.

---

Next lets go over the standard configuration points

```bash
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
```

TODO: Highlevel overview of these.

---

There is a concept of Entity Operators in Strimzi.  These operators are basically operators that allow you to manage things like `Users` and `Topcis`... Lets take a look

```bash
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

If these are not present then you are not able to use the CRD's relevant to these operators.  And you will have to create them through the managed api of some kind or through additional configs etc.

Entity Operator Topic example.

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: words
  namespace: kafka-test
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    min.insync.replicas: 2
```

Entity Operator User example.

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: app-user
  namespace: kafka-test
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      - operations: ["All"]
        resource:
          name: words
          patternType: literal
          type: topic
      - operations: ["All"]
        resource:
          name: mytopic
          patternType: literal
          type: topic
      - operations: ["Read"]
        resource:
          name: app-user
          patternType: literal
          type: group
```

---

Lastly lets talk about `CruiseControl`.  Here this `cruiseControl{}` adds Cruise Control to manage common tasks that work to keep clusters healthy.  It was developed by Linkedin and then made open source.  There is a lot to go over in this tool and i'll cover it in more detail but without this you would need to handle post production release tasks, like topic rebalacning etc.  `KafkaRebalance` is a task that has a resource dedicated to do just that through Cruise Control.

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec: {}
```

This process, once deployed, begins consuming metrics and prepares an optimization task.  Once ready, you can approve this task by annotating the `KafakRebalance` as such to have CruiseControl do the actual rebalance.

```bash
oc annotate kafkarebalance rebalance strimzi.io/rebalance=approve
```

You can also use the `rebalance-auto-approval` annotation to not require the approval process going forward.  This task among others is used pretty much on alerting of idle cpu or other various metrics as well as scaling up and adding brokers.

---

In most cases, you do not want all Kafka brokers / controllers on the same node.  THe default practice is to distribute them across multiple nodes for fault tolerance, high availability, and better performance characteristics.

### Availability and Redundancy
Single-node risk: If all brokers (and controllers) are pinned to one node, then losing that node means your entire Kafka cluster goes down.

Recommended: Spread the brokers (and Zookeeper or KRaft controllers, if applicable) across multiple nodes or zones. That way, if one node fails, the cluster can continue serving traffic.

### Strimzi Defaults
Strimzi Operator: By default, Strimzi will create a StatefulSet for your brokers. Each broker pod is assigned its own PersistentVolumeClaim.

Pod anti-affinity: Strimzi has built-in support for configuring affinity or podAntiAffinity so that broker pods prefer not to land on the same node.

This helps ensure the cluster naturally spreads out (assuming you have enough worker nodes available).

### Performance Considerations
I/O contention: Putting multiple brokers on a single node can lead to disk I/O contention. This can degrade performance, especially if they share the same underlying storage resource.

Network throughput: If multiple high-traffic brokers share a single node’s network interface, that node could become a bandwidth bottleneck.

### Exceptions or Special Cases
Resource isolation: Sometimes you might have a dedicated node pool or set of nodes specifically for Kafka. In this scenario, you might use “affinity” rules to limit Kafka pods to that group of nodes, but still spread them across multiple nodes in that pool—not just a single node.

Development or very small clusters: In a dev environment with limited resources, you might end up with all pods on a single node by necessity. This is usually not recommended in production.

Edge cases: If there’s a very specific hardware configuration or ultra-low latency scenario (and you accept the risk of single node failure), you might temporarily force co-location. But this is rare and typically not aligned with Kafka’s high-availability design.

### Recommended Configuration in Strimzi
Pod Anti-Affinity

By default, Strimzi sets podAntiAffinity so that the broker pods prefer different nodes. You can confirm this in the Kafka custom resource under .spec.kafka.template.pod.

Zone Spreading

If running in a multi-zone setup, you can use labels and topologyKey: failure-domain.beta.kubernetes.io/zone to spread pods across Availability Zones (in AWS or other clouds).

Sufficient Node Count

Ensure your OpenShift cluster has enough worker nodes so the scheduler can distribute the Kafka pods across them.

## Reference Docs

TODO