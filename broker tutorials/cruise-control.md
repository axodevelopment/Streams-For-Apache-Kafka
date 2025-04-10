# Streams For Apache Kafka - Cruise Control

Tutorials around Streams For Apache Kafka running on OCP - Cruise Control

## Overview of tutorial

Cruise control is a powerful operator that can be deployed and configured with streams for apache kafka (strimzi).

## Table of Contents

Tutorial listing

1. [Prereqs](#prerequisites)
2. [Tutorial Breakouts](#tutorials)
3. [Reference Docs](#reference-docs)

---

Lets briefly review the high level architecture of Cruise Control

Lets start with the components:

- Rest API
- Load Monitor
- - Samplers
- Analyzer
- Anomaly Detector
- Executer
- Plugins (Pluggable modules)

For the Rest API its pretty straight forward, you can query the state, query load, resource utilization, proposals etc.  You can also POST requests that trigger rebalancers, add details to brokers, address offline replicas, control some of the other components, like samplers, update topics etc

The Monitor will manage and collect metrics and create models that cruise control can use to create plans for the Executor.  Samplers are managed that do the actual sourcing of metrics

The Analyzer is the brain.  It uses the models created  internal to create proposals based upon the Monitor and specified goals.

The anomaly detector detects 5 main types of anomalies, Broker failures, Goal Violations, Disk failures, metric anomalies, topic anomalies.

Executor actually does the optimization proposals.

---

First lets explore the yaml to see some configuration of Cruise Control

```bash
cruiseControl:
    config:
      goals: >
        com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.MinTopicLeadersPerBrokerGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.PotentialNwOutGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskUsageDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkInboundUsageDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.NetworkOutboundUsageDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.CpuUsageDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.TopicReplicaDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderReplicaDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.LeaderBytesInDistributionGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.PreferredLeaderElectionGoal
      # Note that `default.goals` must be a superset `hard.goals`
      default.goals: >
        com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal
      hard.goals: >
        com.linkedin.kafka.cruisecontrol.analyzer.goals.RackAwareGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.ReplicaCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.DiskCapacityGoal
```

Cruise control has various components that are shown in this config above and are for optimizations that Cruise Control identifies.

- Hard Goals
- Soft Goals
- Default goals
- User-provided goals

Hard goals are goals that must bbe met in order for an optimization for proposal of something like a rebalance to be successful.

Soft Goals do not need to be satisified but ideally are met.

Kafka Rebalance resource can also have user provided goals seperate and in addition to what is specified in the defaults.


Optimizations can be done for

- full (when a full rebalance is run)
- add-brokers (when you are scalingup)
- remvoe-brokers (when youa re scaling-down)

Goals order of priority is located here:

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#goals_order_of_priority

---

Summary of goals:

### Rack-Awareness
Overview:

This goal ensures that partition replicas are deployed in a rack‐aware manner. The idea is to distribute replicas across different physical racks, so a rack-level failure does not impact all replicas of a partition.

Annotations & Dependencies:

- RackAwareDistributionGoal Variant: In contrast to strict rack-awareness, this variant permits multiple replicas in one rack if it results in an even distribution across racks.
- Dependencies: Requires that each broker be annotated or configured with rack information (typically via node labels) so that Cruise Control can make intelligent placement decisions.

### Minimum Number of Leader Replicas per Broker for a Set of Topics
Overview:

This goal targets ensuring that every broker holds at least a minimum number of leader replicas for a given set of topics. It is aimed at balancing the leadership load across the cluster.

Annotations & Dependencies:

- Annotation: Often enforced via broker configuration or topic-level settings.
- Dependencies: Relies on the metadata that defines leader election and may require custom thresholds to be set per workload.

### Replica Capacity Goal
Overview:

This goal attempts to ensure that no individual broker holds more than a specified number of partition replicas. Its primary aim is to avoid overly burdening any single broker with too many replicas, which could lead to resource strain.

Annotations & Dependencies:

- Configuration Dependency: Must define the maximum allowed replica count per broker in the Cruise Control or broker configuration.
- Operational Impact: Plays a vital role during rebalancing operations to maintain even load.

### Capacity Goals
Overview:

Capacity goals collectively ensure that a broker’s resource utilization for critical aspects is kept below defined thresholds. The specific capacity goals include:

- DiskCapacityGoal: Keeps disk usage below a set threshold.
- NetworkInboundCapacityGoal: Limits the inbound network traffic.
- NetworkOutboundCapacityGoal: Controls the outbound network traffic.
- CpuCapacityGoal: Ensures CPU usage stays within acceptable limits.

Annotations & Dependencies:

- Metric Requirements: These goals depend on accurate real-time metrics from the broker’s system (disk, network, CPU).
- Configuration: Thresholds need to be explicitly defined, and monitoring must be in place for effective enforcement.

### Replica Distribution Goal
Overview:

This goal works to balance the total number of partition replicas across all brokers. The intent is to distribute replicas evenly so that no single broker is overloaded.

Annotations & Dependencies:

- Evenness: Often relies on broker-level labels or capacity metadata to determine the “evenness” of distribution.
- Rebalancing: It is a key factor during cluster rebalancing tasks.

### Potential Network Output Goal
Overview:

Also called the potential network output goal, its purpose is to ensure that if all replicas on a broker became leaders simultaneously, the broker’s network outbound capacity would not be overwhelmed.

Annotations & Dependencies:

- Metrics Dependency: Requires estimates for leader traffic rates and predefined outbound bandwidth capacities for each broker.
- Operational Use: Useful to prevent future performance bottlenecks during leader elections or traffic spikes.

### Resource Distribution Goals
Overview:

These goals aim to constrain the variance in resource utilization (such as disk, network, and CPU) among brokers. They are activated primarily when broker resource usage is above a configured percentage (i.e., in medium to high utilization scenarios). They include:

- DiskUtilDistributionGoal
- NetworkInboundUtilDistributionGoal
- NetworkOutboundUtilDistributionGoal
- CpuUtilDistributionGoal

Annotations & Dependencies:

- Utilization Threshold: Typically only engage when overall cluster usage is high.
- Dependencies: Depend on accurate, real-time resource utilization metrics, and proper threshold definitions within Cruise Control’s configuration.

### Topic Replica Distribution Goal
Overview:

Focuses on distributing the replicas of a single topic evenly across the cluster. This helps ensure that topic-specific load and failures are more evenly mitigated across the brokers.

Annotations & Dependencies:

- Topology Awareness: May use topic metadata and broker capacity data.
- Dependence on Distribution Metrics: Relies on the ability of Cruise Control to assess the current distribution of replicas across brokers.

### Leader Replica Distribution Goal
Overview:

This goal is designed to balance the number of leader replicas across brokers, ensuring that leadership responsibilities (and their attendant load) are spread evenly throughout the cluster.

Annotations & Dependencies:

- Load Balancing: Directly impacts the distribution of client connection and processing responsibilities.
- Metric Dependency: Requires monitoring leader replica counts on each broker.

### Leader Bytes-In Distribution Goal
Overview:

Aims to balance the incoming bytes rate handled by the leader replicas across brokers. This ensures that no broker faces a disproportionate load in terms of incoming network traffic for its leader partitions.

Annotations & Dependencies:

- Network Metrics: Depends on precise measurements of leader bytes-in rates per broker.
- Usage: Helps optimize the broker’s network usage and prevent bottlenecks.

### Preferred Leader Election Goal
Overview:

Ensures that, for each partition, the first replica listed becomes the leader. This goal simplifies leadership transitions and can help standardize the election process, leading to predictability in leadership assignments.

Annotations & Dependencies:

- Configuration: Often driven by a preferred leader configuration setting in Cruise Control or broker configs.
- Impact: Affects client connection patterns and can reduce unnecessary load shifts during leader reassignments.

### Kafka Assigner Goals
Overview:

When Cruise Control is operated in a mode similar to the Kafka assigner tool (enabled via the kafka_assigner parameter), it picks up a different set of goals aimed at optimizing replica assignments, including:

- KafkaAssignerDiskUsageDistributionGoal: Ensures replicas are assigned in a rack-aware manner.
- KafkaAssignerEvenRackAwareGoal: Attempts to balance the number of replicas across brokers.

Annotations & Dependencies:

- Mode Activation: Only in effect when the kafka_assigner parameter is set to true.
- Rack Configuration: Still relies on accurate rack or topology metadata for brokers.

### Intra-Broker Disk Capacity Goal
Overview:

This goal is focused on ensuring that the disk utilization on individual brokers remains below a specified threshold. It acts to prevent overloading a broker’s disk capacity.

Annotations & Dependencies:

- Activation: This goal is only picked up when the rebalance_disk parameter is set to true.
- Version Dependency: Not available in older branches.

### Intra-Broker Disk Usage Distribution Goal
Overview:

Similar in intent to the Intra-Broker Disk Capacity Goal, this goal attempts to balance the utilization across multiple disks within the same broker so that no single disk becomes a bottleneck.

Annotations & Dependencies:

- Activation: Also requires that rebalance_disk be set to true.
- Version Dependency: Not available in older branches.

### BrokerSet Aware Goal
Overview:

This goal targets replica movements within a defined subset of brokers (a BrokerSet). It ensures that any rebalancing actions or replica movements remain within these set boundaries.

Annotations & Dependencies:

- Annotations: Requires brokers to be appropriately tagged or grouped into BrokerSets.
- Dependency: Useful where brokers share similar hardware, network characteristics, or are located in the same data center to limit cross-boundary movements.

---

Directly from the source I want to add in some additional notes.  You can create your own goals and import them into CruiseControl.  

Please refer to this `abstract class` that would be used as a base at the following github location.

```bash
https://github.com/linkedin/cruise-control/blob/migrate_to_kafka_2_4/cruise-control/src/main/java/com/linkedin/kafka/cruisecontrol/analyzer/goals/AbstractGoal.java
```

I'll also add the high level of which Analyzer rules are generally available across implementations

The Analyzer is the "brain" of Cruise Control. It uses a heuristic method to generate optimization proposals based on the user provided optimization goals and the cluster load model from the Load Monitor. Cruise control allows specifying hard goals and soft goals. A hard goal is one that must be satisfied (e.g. replica placement must be rack-aware). Soft goals on the other hand may be left unmet if doing so makes it possible to satisfy all the hard goals. The optimization would fail if the optimized results violate a hard goal. We have implemented the following hard and soft goals so far:

- Replica placement must be rack-aware
- Broker resource utilization must be within pre-defined thresholds
- Network utilization must not be allowed to go beyond a pre-defined capacity even when all replicas on the broker become leaders
- Attempt to achieve uniform resource utilization across all brokers
- Attempt to achieve uniform bytes-in rate of leader partitions across brokers
- Attempt to evenly distribute partitions of a specific topic across all brokers
- Attempt to evenly distribute replicas (globally) across all brokers
- Attempt to evenly distribute leader replicas (globally) across all brokers
- Disk utilization must be within pre-defined thresholds for each disk within a broker (not available in kafka_0_11_and_1_0 branch)
- Attempt to evenly distribute disk utilization across disks of each broker (not available in kafka_0_11_and_1_0 branch)

The high order goal optimization logic:

```bash
For each goal g in the goal list ordered by priority { 
 For each broker b { 
   while b does not meet g’s requirement { 
     For each replica r on b sorted by the resource utilization density { 
       Move r (or the leadership of r) to another eligible broker b’ so b’ still satisfies g and all the satisfied goals 
       Finish the optimization for b once g is satisfied. 
     } 
     Fail the optimization if g is a hard goal and is not satisfied for b 
   } 
 } 
 Add g to the satisfied goals 
}
```

