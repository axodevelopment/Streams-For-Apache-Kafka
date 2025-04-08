# Streams For Apache Kafka - Day 2 Operations 

Tutorials around Streams For Apache Kafka running on OCP - How to manage day 2 tasks

## Overview of tutorial

There are several Day 2 tasks that need to be handled either ad hoc or at some reasonable cadence.

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

The goal is to have following resources deployed:

- tutorial-cluster-east-zk-v1.yaml
- tutorial-cluster-east-v1.yaml
- tutorial-nodepool-east-v1.yaml

If you have been doing other tutorials these will be deployed in part, you can either use what you have deployed or deploy the v1 of the yamls -or- you can do the following steps to get the basic clusters deployed.



Lets start with deploying the basic no frills kafka cluster that includes two clusters, (resize if you want), one that has kraft, the other zookeeperbased.

First lets createa  project

```bash
oc new-project kafka-tutorial-east
oc new-project kafka-tutorial-kraft-east
```

Here we will do our testinging out of `kafka-tutorial-east`

Now we will want to deploy two yaml files that are in the ./ref folder.

```bash
oc apply -f tutorial-cluster-east-v1.yaml

oc apply -f tutorial-cluster-east-zk-v1.yaml
```

Before continuing any further please wait for all pods to be created.

```bash

oc get pods -w
```



This process will take some time, we will need to create zookeeper instances, operator instances, brokers with zookeeper and brokers with kraft.

`WARNING` if you see no activity in the `oc get pods -w` you either have already completed this step or have some yaml issue take note of a common one for kraft below

To deploy using kraft not only do you need the Kafka resource but you also need to enable nodepool resource you can see the operator will likely fail (current version it does) here is how you can tell in the logs.

Presently the operator is still called amq-streams-cluster-operator that is subject to change in future releases.

```bash
oc get pods -n openshift-operators | grep streams

oc logs amq-streams-cluster-operator-v2.9.0-0-b79d9649-ntr9l -n openshift-operators
```

Here you will see some reconciliation failure if KRaft is enabled and no NodePools are deployed.  Something like the following.

```bash
WARN  AbstractOperator:566 - Reconciliation #99(timer) Kafka(kafka-tutorial-kraft-east/my-cluster-kraft): Failed to reconcile
io.strimzi.operator.common.InvalidConfigurationException: KRaft can only be used with a Kafka cluster that uses KafkaNodePool resources.
```

If you get this just note what I mentioned above when a KRaft based Kafka cluster is deployed it will search for the configuration from a KafkaNodePool.  I'll explain this resource in another tutorial, but to keep it simple, it is used by the KRaft broker to configure and setup the broker with those configurations in the nodepool.

```bash
oc apply -f tutorial-nodepool-east-v1.yaml

oc apply -f tutorial-cluster-east-v1.yaml
```

If you want to follow along what steps are happening

```bash
oc logs amq-streams-cluster-operator-v2.9.0-0-b79d9649-ntr9l -n openshift-operators --follow
```

To know when your cluster is complete, for KRaft enabled kafka with the configured i have in this tutorial you should end up with something along the lines of:

```bash
oc get pods -w

NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-kraft-cruise-control-5778f6468-ckqpl     1/1     Running   0          30s
my-cluster-kraft-entity-operator-545ccd488d-ldvz6   2/2     Running   0          52s
my-cluster-kraft-kafka-np-0                         1/1     Running   0          78s
my-cluster-kraft-kafka-np-1                         1/1     Running   0          78s
my-cluster-kraft-kafka-np-2                         1/1     Running   0          78s
```

For a zookeeper setup you should end up with something like:

```bash
oc get pods -n kafka-tutorial-east
NAME                                             READY   STATUS    RESTARTS   AGE
my-cluster-zk-cruise-control-5d7dcbd84f-9txf7    1/1     Running   0          28m
my-cluster-zk-entity-operator-769d59764d-7bc7m   2/2     Running   0          28m
my-cluster-zk-kafka-0                            1/1     Running   0          29m
my-cluster-zk-kafka-1                            1/1     Running   0          29m
my-cluster-zk-kafka-2                            1/1     Running   0          29m
my-cluster-zk-zookeeper-0                        1/1     Running   0          30m
my-cluster-zk-zookeeper-1                        1/1     Running   0          30m
my-cluster-zk-zookeeper-2                        1/1     Running   0          30m
```

## Tutorial Steps

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

The different states of a rebalance:

```bash
oc annotate kafkarebalance <kafka_rebalance_resource_name> strimzi.io/rebalance="refresh"
oc annotate kafkarebalance <kafka_rebalance_resource_name> strimzi.io/rebalance="approve"
kubectl annotate kafkarebalance <kafka_rebalance_resource_name> strimzi.io/rebalance="stop"
```