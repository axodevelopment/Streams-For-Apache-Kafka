# Streams For Apache Kafka - Tuning

Tutorials around Streams For Apache Kafka running on OCP

## Overview of tutorial

This tutorial will give a high level overview of Tuning the Broker yaml,

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

### Key Questions:

#### Message Throughput and Size:
- What is the expected incoming message rate per topic (in MB/s or KB/s)?
- What is the average size of each message?
- Do you expect high peaks in message throughput? If so, what is the peak message rate?

##### Calculation:
- **Write rate**: `<Write rate> = <Topic Throughput> * <Replicas>`
- **Read rate**: `<Read rate> = <Topic Throughput> * (<Consumer Groups> + <Replicas> - 1)`

These formulas help estimate the disk and network throughput required.

---

#### Topic and Partition Configuration:
- How many topics will be created, and what is the expected number of partitions per topic?
- Will there be a need for message ordering within topics? This affects the partitioning strategy.

##### Calculation:
- More partitions allow parallel consumption but increase resource usage. Ensure that the number of partitions fits the expected throughput and scalability requirements.

---

#### Replication Factor:
- What replication factor will be used for each topic (e.g., 2, 3)?
- Are there any specific disaster recovery or high availability requirements, such as mirroring data to different data centers?

##### Calculation:
- **Storage Capacity**: `<Storage capacity> = <Retention in seconds> * <Topic write rate> * <Replication factor>`
  
Consider higher replication factors for greater availability, which directly impacts the resource and storage requirements.

---

#### Retention Policies:
- Will retention be time-based or size-based for topics?
- What is the expected message retention time (e.g., 1 day, 7 days)?
- Will size-based retention be applied, and if so, what is the maximum retention size per topic?

##### Calculation:
- **For time-based retention**: `<Storage capacity> = <Retention in seconds> * <Topic write rate> * <Replication factor>`
- **For size-based retention**: `<Storage capacity> = <Retention in bytes> * <Replication factor>`

---

#### Consumer Groups and Message Processing:
- How many consumer groups will be reading from each topic?
- How many consumers are expected per group, and what is the expected consumer lag (i.e., how far behind can consumers be from producers)?

##### Calculation:
- **Read rate** can be calculated as: `<Read rate> = <Topic Throughput> * (<Consumer Groups> + <Replicas> - 1)`
- Use **maximum lag** to estimate when messages might need to be read from disk: `<Maximum lag in seconds> = <Disk cache memory> / <Write rate>`