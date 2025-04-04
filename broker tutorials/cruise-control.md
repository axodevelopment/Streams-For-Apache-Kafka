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