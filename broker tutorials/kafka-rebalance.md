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

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

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