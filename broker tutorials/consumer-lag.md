# Streams For Apache Kafka - Consumer Lag and Alerting 

Tutorials around Streams For Apache Kafka running on OCP - How to identify consumer-lag and how do we alert on that in OCP

## Overview of tutorial

Consumer lag is important to keep track of and understand.  We will do the following:

- First define a high level concept of consumer lag through our exportable metrics through KafkaExporter
- Understand the impact of this metric that may occur during rebalancing.
- Visualize the metric
- Create an alert based upon this metric.

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

Lets start by first define a high level concept of consumer lag.

Consumer lag is a very useful metric to determine how far behind a consumer group is relative to the head of the partition.  Meaning are more messages being wrote that can be consumed.

At a high level consumer lag is the difference between:

- Current log end offset (latest message in the partion)
- Consumer group offset (last committed offset by a consumer)

In Grafana or OCP Metrics, ultimate you want to have a PromQL that is roughly in the shape of

```bash
kafka_consumergroup_current_offset{consumergroup="my-group-a"} 
  - kafka_consumergroup_topic_partition_leader_offset{consumergroup="my-group-a"}
```

So there will be two main approaches here the easy way and the slightly less easy way.

Metrics with `Streams for Apache Kafka` comes with KafkaExporter as a quick metric export and it has metrics already exported for an approximatation of that lag imported when you scrape that endpoint.

Here would be the PromQL

```bash
kafka_consumergroup_lag{consumergroup="my-group"}
```

To setup metrics I have a tutorial that covers this in more detail called `metrics.md` in this folder.

At a high level, you want to enable `KafkaExporter`.  Then create a `Service` that points to that path and lastly add a `ServiceMontior`.

After the targets come online you will see those metrics available. 

Ok so now that you have KafkaExporter setup and you have a ServiceMonitor plus services setup and see the metrics in OCP, how do we create alerts?

There are two OCP Alert paths you can take a generic one and an OCP specific one i'll show you example of both `AlertingRule` + `PrometheusRule`.

`AlertingRule`

```bash
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: consumer-lag
  namespace: openshift-monitoring
spec:
  groups:
    - name: consumer-lag-rules
      rules:
        - alert: ConsumerLagHigh
          for: 5m
          expr: |
            kafka_consumergroup_lag{consumergroup="my-group"} > 100
          labels:
            severity: warning
          annotations:
            message: "Consumer group 'my-group' lag has exceeded 100 for over 5 minutes."
```

`PrometheusRule`

```bash
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: consumer-lag
  namespace: openshift-monitoring
  labels:
    app.kubernetes.io/managed-by: cluster-monitoring-operator
    app.kubernetes.io/name: kube-prometheus
    app.kubernetes.io/part-of: openshift-monitoring
    prometheus: k8s
    role: alert-rules
spec:
  groups:
    - name: kafka-lag-alerts
      rules:
        - alert: ConsumerLagHigh
          expr: |
            kafka_consumergroup_lag{consumergroup="my-group"} > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Consumer group lag is too high"
            description: "Consumer group lag has exceeded 100 for over 5 minutes."

```

With the above you should know how to get the consumer-lag metrics, understand how to see those metrics and capture them in OCP Monitoring and setup alerts.