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

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

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