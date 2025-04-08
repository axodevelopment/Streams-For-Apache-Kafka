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

Here is a list of Day 2 Tasks that we will cover briefly.

| Task | Description | Link |
| Pod Management - Rolling Update | Strimzi exposes annotations on strimzipodsets that allow you to trigger management activities | [Pod Management](#pod-management) |
| Scaling Clusters | How to Scale clusters | [Scaling Clusters](#scaling-clusters) |
| Rebalacning Clusters | Cruise Control can be used Rebalance Cluster what does that imply and what are some restrictions | [Rebalance Cluster](#rebalance-clusters) |


### Pod Management

There are various ways to annotate and have actions take place here is a list of some annoations and what they effectively do.  These annotations do require the operator to be running.

This annotation is part of the migration from Zookeeper to Kraft, it will be covered in more depth in another tutorial (migrate-from-zk-to-kraft.md)

```bash
oc annotate kafka my-cluster strimzi.io/kraft="migration" --overwrite
oc annotate kafka my-cluster strimzi.io/kraft="enabled" --overwrite
oc annotate kafka my-cluster strimzi.io/kraft="rollback" --overwrite
oc annotate kafka my-cluster strimzi.io/kraft="disabled" --overwrite
```

If you need to delete a pod and its pvc (note this can cause data loss), the pvc will be deleted and recreated.

```bash
oc annotate pod <cluster_name>-kafka-<index_number> strimzi.io/delete-pod-and-pvc="true"
```

When node pools are created you can create id's and id ranges that get assigned like pools.  Sometimes I need to add new brokers or remove them and want to create id's for them

```bash
oc annotate kafkanodepool pool-a strimzi.io/next-node-ids="[0,1,2]"
```

```bash
oc annotate kafkanodepool pool-b strimzi.io/remove-node-ids="[9,8,7,10-20]"
```

Managing KafkaTopics breaks down into two types Managed KafkaTopics and UnManaged KafkaTopics.  There are some automation and tasks restrictions swhen you have a managed kafkatopic that you can't do.  Thankfully, you can change a topic from managed to unmanaged so you won't trigger reconciliation restrictions etc when you make your changes.  An example of a change `metadata.name` of a resource in managed mode isn't allowed since that creates the resource.

```bash
oc annotate kafkatopic my-topic-1 strimzi.io/managed="false" --overwrite
```

During cert renewal you typically have a 30 day window (by default) where both the new cert and the existing cert are valid.  There are some various states here that you can work with if you want to renew early or finish the renewal when needed.

```bash
oc annotate secret my-cluster-cluster-ca-cert -n my-project strimzi.io/force-renew="true"
oc annotate secret my-cluster-cluster-ca-cert -n my-project strimzi.io/force-renew="true"
oc annotate secret <ca_certificate_secret> strimzi.io/ca-cert-generation="<ca_certificate_generation>"
oc annotate secret <ca_key_secret> strimzi.io/ca-key-generation="<ca_key_generation>"
```

Another example might be cert renewal

```bash
oc annotate Kafka <name_of_custom_resource> strimzi.io/pause-reconciliation="true"
oc annotate secret <ca_certificate_secret> strimzi.io/ca-cert-generation="<ca_certificate_generation>"
oc annotate secret <ca_key_secret> strimzi.io/ca-key-generation="<ca_key_generation>"
oc annotate Kafka <name_of_custom_resource> strimzi.io/pause-reconciliation="false" --overwrite 
oc annotate Kafka <name_of_custom_resource> strimzi.io/pause-reconciliation-
```

During a Rebalance for example, when a scale-down op occurs there are checks to to ensure partitions on brokers before a scale-down occurs.  YOu may want to remove this behavior on a scale down where hihgh traffic tpics could block scale-dwn forever.

```bash
oc annotate Kafka my-kafka-cluster strimzi.io/skip-broker-scaledown-check="true"
```

Rebalance annotations will be covered more in `kafka-rebalance.md`

Rolling update annotation exists for Kafaka and other resources `Kafka`, `zookeeper`, `connect`, `mirrormaker2`

```bash
oc annotate strimzipodset <cluster_name>-<resource> strimzi.io/manual-rolling-update="true"
oc annotate pod <cluster_name>-<resource>-<index_number> strimzi.io/manual-rolling-update="true"

```

### Scaling Clusters

Cruise control can be used to scale up or scale-down the cluster

Please review kafka-rebalance.md

### Rebalance Clusters

Please review kafka-rebalance.md

### Partition reassignment

The longer a cluster is up and running, partitions may need to be redistributed.  For most current users Cruise Control has automated solution for this and is what is preferred / recommended by Red Hat to do.  But in case you want to run the legacy process you can use the `kafka-reassign-partitions.sh` tool.

Since this isn't currently recommended i'll just add a link for reference.

The short of it is you can create a JSON file which has the partitions and then call the above .sh.

```bash
oc run helper-pod -ti --image=registry.redhat.io/amq-streams/kafka-36-rhel8:2.6.0 --rm=true --restart=Never -- bash
```

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#generating_a_partition_reassignment_plan

### Retrieveing trouble shooting data

The AMQ streams software download comes with a `report.sh`


Downloading that tool you can use it to collect the data you would need

```bash
./report.sh --namespace=<cluster_namespace> --cluster=<cluster_name> --out-dir=<local_output_directory>
```

This generally collects 

- logs
- configurations
- operator details
- resources
- events

For Red Hat support you may be asked to run this tool.

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#assembly-distributed-tracing-procedures-str


### Reasons for a restart event

I'll just link to the Red Hat documents here but if the Cluster Operator initiates a restart event you may want to track these.

```bash
oc get events --field-selector reportingController=strimzi.io/cluster-operator
```

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#ref-operator-restart-events-reasons-str

### Connecting to ZooKeeper from a terminal

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#proc-connnecting-to-zookeeper-str

### Maintenance time windows for rolling updates

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.6/html-single/deploying_and_managing_amq_streams_on_openshift/index#assembly-maintenance-time-windows-str

