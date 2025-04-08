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

Lets start by learning how to enable debug logging on one of our clusters.  For now we will update both but lets first go over how logging works.

Since Apache Kafka isn't the only component in Strimzi we may want to log different parts of Streams for Apache Kafka.  Here are some components as you might have noticed from above...

- Kafka Nodes
- Kafka Entity Operator
- Kafka Cruise Control
- (Optionally) Zookeeper

We can set the logging in each item in a different way.  But there are some limitations to note...

First, to enable full debug mode some operators may require you to specify the logging configuration as either inline or external.  Strimzi uses Log4j/2 as the logging component.  As such at least at the time of this writing (3.9) the full spec for configuring lots is not fully impelmented in yaml, so external through a config map is the way to go in most cases.

If Kafka was deployed using the Cluster Operator, changes to Kafka logging levels are applied dynamically.
If you use external logging, a rolling update is triggered when logging appenders are changed.

Lets look at logging internal spec:

```bash
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
    logging:
      type: inline
      loggers:
        zookeeper.root.logger: INFO
        log4j.logger.org.apache.zookeeper.server.FinalRequestProcessor: TRACE
        log4j.logger.org.apache.zookeeper.server.ZooKeeperServer: DEBUG
```

This internal set is usuable in the following components:

- `Kafka`.spec.kafka.`logging`
- `Kafka`.spec.zookeeper.`logging`
- `Kafka`.spec.entityoperator.topicOperator.`logging`
- `Kafka`.spec.entityoperator.userOperator.`logging`
- `Kafka`.spec.cruiseControl.`logging`

Here is some inline examples from the strimzi docs

`Kafka`.spec.kafka.`logging`:

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
spec:
  kafka:
    logging:
      type: inline
      loggers:
        kafka.root.logger.level: INFO
        log4j.logger.kafka.coordinator.transaction: TRACE
        log4j.logger.kafka.log.LogCleanerManager: DEBUG
        log4j.logger.kafka.request.logger: DEBUG
        log4j.logger.io.strimzi.kafka.oauth: DEBUG
        log4j.logger.org.openpolicyagents.kafka.OpaAuthorizer: DEBUG
```

Breakdown of `Kafka`.spec.zookeeper.`logging`

```bash
  zookeeper:
    logging:
      type: inline
      loggers:
        zookeeper.root.logger: INFO
        log4j.logger.org.apache.zookeeper.server.FinalRequestProcessor: TRACE
        log4j.logger.org.apache.zookeeper.server.ZooKeeperServer: DEBUG
```

One of the more important ones will be topics, Kafka`.spec.entityoperator.topicOperator.`logging`

```bash
topicOperator:
      watchedNamespace: my-topic-namespace
      reconciliationIntervalMs: 60000
      logging:
        type: inline
        loggers:
          rootLogger.level: INFO
          logger.top.name: io.strimzi.operator.topic
          logger.top.level: DEBUG
          logger.toc.name: io.strimzi.operator.topic.TopicOperator
          logger.toc.level: TRACE
          logger.clients.level: DEBUG
```

For Users, `Kafka`.spec.entityoperator.userOperator.`logging`

```bash
entityOperator:
    # ...
    userOperator:
      watchedNamespace: my-topic-namespace
      reconciliationIntervalMs: 60000
      logging:
        type: inline
        loggers:
          rootLogger.level: INFO
          logger.uop.name: io.strimzi.operator.user
          logger.uop.level: DEBUG
          logger.abstractcache.name: io.strimzi.operator.user.operator.cache.AbstractCache
          logger.abstractcache.level: TRACE
          logger.jetty.level: DEBUG
```

Lastly for Cruise Control, `Kafka`.spec.cruiseControl.`logging`:

```bash
  cruiseControl:
    logging:
      type: inline
      loggers:
        rootLogger.level: INFO
        logger.exec.name: com.linkedin.kafka.cruisecontrol.executor.Executor
        logger.exec.level: TRACE
        logger.go.name: com.linkedin.kafka.cruisecontrol.analyzer.GoalOptimizer
        logger.go.level: DEBUG
```

Sometimes you need more configuration you can make it external and provide all settings you need for Log4j/2.  The Cluster Operator does not validate keys or values in the config object provided. If an invalid configuration is provided, the Kafka Connect cluster might not start or might become unstable. In this case, fix the configuration so that the Cluster Operator can roll out the new configuration to all Kafka Connect nodes.

Here is an example:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: topic-operator-logging
  namespace: kafka-tutorial-kraft-east
data:
  log4j2.properties: |
    # Root logger
    rootLogger.level = INFO
    rootLogger.appenderRefs = STDOUT
    rootLogger.appenderRef.STDOUT.ref = STDOUT
    
    # Topic Operator loggers
    logger.top.name = io.strimzi.operator.topic
    logger.top.level = DEBUG
    
    logger.toc.name = io.strimzi.operator.topic.TopicOperator
    logger.toc.level = TRACE
    
    logger.k8s.name = io.strimzi.operator.topic.K8sImpl
    logger.k8s.level = DEBUG
    
    logger.kafkaadmin.name = io.strimzi.operator.topic.KafkaAdminImpl
    logger.kafkaadmin.level = DEBUG
    
    logger.topicstore.name = io.strimzi.operator.topic.TopicStore
    logger.topicstore.level = DEBUG
    
    logger.reconciliation.name = io.strimzi.operator.common.ReconciliationLogger
    logger.reconciliation.level = DEBUG
    
    # Kafka clients
    logger.kafkaclients.name = org.apache.kafka.clients
    logger.kafkaclients.level = DEBUG
    
    # Appenders
    appender.console.type = Console
    appender.console.name = STDOUT
    appender.console.layout.type = PatternLayout
    appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss} %-5p [%c{1}] %m%n
```

To then use the above configuration you need to pull it in to the section you want and set a `type: external`

```bash
      logging:
        type: external
        valueFrom:
          configMapKeyRef:
            name: topic-operator-logging
            key: log4j2.properties
```



## Reference Documents