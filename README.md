# Tutorials and information around Streams For Apache Kafka

Tutorials around Streams For Apache Kafka running on OCP

## Overview of this tutorial

In this Repo I want to spend some time covering various types of Kafka setups which include most types of configurations and setups.  Ideally i'll be able to elaborate a bit on pros and cons for different designs and architectures but at the very least give you a step by step on setting up more interesting designs that you may actually use rather then just high level tutorials that don't get you close to an actual prod release.

## Table of Contents

Tutorial listing

1. [Prereqs](#prerequisites)
2. [Tutorial Breakouts](#tutorials)
3. [Reference Docs](#reference-docs)

---

## Prerequisites

- AMQ Streams
- Streams for Apache Kafka
- Strimzi based kafka deployments

---

## Tutorials

These tutorials are mainly OCP focused.  However alot of this material can be used in any Strimzi based deployment.

Under the `broker tutorials` folder you will see some tutorials called `<tutorial>.md`

| Name               | Description                    | Status           |
|--------------------|--------------------------------|------------------|
| basics     | basics    | In Progress          |
| logging     | How to configure logging    | Draft* |
| metrics     | How to configure metrics    | Draft* |
| cruise control     | How to configure and use cruise control    | In Progress* |
| Console    | Explore the Console    | Not Started |
| Consumer Lag    | Define Consumer lag metric and create alerts   | Not Started |
| migrate from zk to kraft    | How do we migrate from ZK to Kraft based brokers   | Not Started |
| topics and users as code   | How can we gitops our kafka cluster  | Not Started |


---

## Reference Docs

TODO