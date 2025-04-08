# Streams For Apache Kafka - Logging 

Tutorials around Streams For Apache Kafka running on OCP - How to enable and manage Logging

## Overview of tutorial

Being able to enable debug mode and view metrics are important aspects to long term management of a cluster.  In this tutorial we will talk about how to configure and view these items.

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

## Tutorial Steps

Lets start by learning how to enable metrics on one of our clusters.  For now we will update the KRaft one.

Since Apache Kafka isn't the only component in Strimzi we may want to log different parts of Streams for Apache Kafka.  Here are some components as you might have noticed from above...

- Kafka Nodes
- Kafka Entity Operator
- Kafka Cruise Control
- (Optionally) Zookeeper

Lets quickly go over the main components on our first approach for metrics and how to edit the Kafka CR to enable metrics.  Therea are two main compoents in this approach

- KafkaExporter
- JMX

Kafka Exporter is provided with Streams for Apache Kafka for deployment with a Kafka cluster to extract additional metrics data from Kafka brokers related to offsets, consumer groups, consumer lag, and topics.

The Exporter will add various metrics that arn't covered by JMX and host those.  JMX will have its own list of metrics and host those for prometheus to scrape.

Lets talk about the config map, which metrics we want to pull out.  First its important to note that this is the JMX side of things.

`tutorial-metrics-cm-east.yaml` is the file we will look at in part

```bash
- pattern: kafka.(\w+)<type=(.+), name=(.+)><>Value
      name: kafka_$1_$2_$3
      type: GAUGE
    # Emulate Prometheus 'Summary' metrics for the exported 'Histogram's.
    # Note that these are missing the '_sum' metric!
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Count
      name: kafka_$1_$2_$3_count
      type: COUNTER
      labels:
        "$4": "$5"
        "$6": "$7"
```

You'll notice that there is a wrapping up of the JMX / JVM metrics internally and then converted to proper prometheus patterns `GUAGE`, `COUNTER`.  There are a ton of variations of this.  To many to count, but I have added the standard ones promted by Strimzi in the `configMap` mentioned earlier.

For KafkaExporter it is very easy to setup.  You just annotate it in your Kafka CR.

Please look at `tutorial-cluster-east-v3.yaml`

```bash
          name: kafka-metrics
          key: kafka-metrics-config.yml
  entityOperator:
    topicOperator: {}
    userOperator: {}
  cruiseControl: {}
  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
```

In this case, we are pulling all metrics from all topics `".*"` and all consumer groups `".*"`.

Lets deploy the latest cluster AND the configmap referenced later:

```bash
oc apply -f tutorial-metrics-cm-east.yaml

oc apply -f tutorial-cluster-east-v3.yaml
```

OCP has internal metric providers that get created for a lot of resources but Kafka isn't really one of them, or at least not the internal metrics to the ApacheKafka / Strimzi components.

You'll notice that if you go into OpenShift and the Observe section that you don't have any JMX (for Kafka at least) metrics.

This is because there is no prometheus Target created for prometheus scraping and that Target isn't created automatically.  So how do we get that?

`ServiceMonitor` is to the rescue.  I created a tutorial for this at another location:

https://github.com/axodevelopment/ServiceMonitor/blob/main/README.md

This will help you setup and create user workload monitoring.

After that, you can do the following step which I have creatd at `tutorial-servicemonitor-east-v1.yaml`.  By deploying that resource it will createa service endpoint to the prometheus port and location that JMX and KafkaExporter host.

The `ServiceMonitor` resource can then be deployed using a service selector to the services mentioned above and the targets will be scraped.

Each scraper needs to have a seperate service monitor attached to it.

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-cluster-kraft-kafka-jmx
  namespace: kafka-tutorial-kraft-east
  labels:
    app: strimzi
spec:
  selector:
    matchLabels:
      app: strimzi-custom
      strimzi.io/cluster: my-cluster-kraft
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-cluster-kraft-kafka-exporter
  namespace: kafka-tutorial-kraft-east
  labels:
    app: strimzi
spec:
  selector:
    matchLabels:
      strimzi.io/cluster: my-cluster-kraft
      strimzi.io/kind: Kafka
      strimzi.io/name: my-cluster-kraft-kafka-exporter
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
```

Ok so lets deploy the files:

```bash
oc apply -f tutorial-servicemonitor-east-v1.yaml
```

If everything is up and going a Link will be created with a proper status:

![Targets](https://github.com/axodevelopment/Streams-For-Apache-Kafka/blob/main/images/observe-targets.jpg)

With that in mind you should also be able to see metrics flowing in, this will take around 15s as noted in the interval in the ServiceMonitor.

![Targets](https://github.com/axodevelopment/Streams-For-Apache-Kafka/blob/main/images/metrics-cm.jpg)


Lastly I wanted to add a good metric to use that I use which is storage capacity / availability of your topics.

```bash
kubelet_volume_stats_available_bytes{persistentvolumeclaim=~"data(-[0-9]+)?-(.+)-kafka-[0-9]+"} * 100 / kubelet_volume_stats_capacity_bytes{persistentvolumeclaim=~"data(-[0-9]+)?-(.+)-kafka-[0-9]+"}
```

You can query like this in the metrics section as well.

![Targets](https://github.com/axodevelopment/Streams-For-Apache-Kafka/blob/main/images/topic-capacity.jpg)

## Reference Documents