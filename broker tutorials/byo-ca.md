# Streams For Apache Kafka - Console 

Tutorials around Streams For Apache Kafka running on OCP - How to BYO Cluster CA

## Overview of tutorial

In this tutorial we will learn to setup and manage the your own Cluster CA Broker.

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

Please note in the documentation that I will be heavily following is located at:

https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.9/pdf/deploying_and_managing_streams_for_apache_kafka_on_openshift/Red_Hat_Streams_for_Apache_Kafka-2.9-Deploying_and_Managing_Streams_for_Apache_Kafka_on_OpenShift-en-US.pdf

Specifically starting at 18.6.1

## Tutorial Steps

Creating the full chain cert.  Pre req is:

Cluster CA -> One ore More Intermediate CA's -> Root CA.

Create a dir for this.

cd into dir

### Root

this is ujust my dir structure

```
mkdir -p certs/root certs/intermediate certs/cluster
```

root key

```
openssl genrsa -out root/root.key 4096
```


raoot crt

```
openssl req -x509 -new -nodes -key root/root.key -sha256 -days 3650 -subj "/C=US/O=Mikes Company/CN=Mikes-Root-CA" -addext "basicConstraints=critical, CA:TRUE, pathlen:2" -addext "keyUsage=critical, keyCertSign, cRLSign" -out root/root.crt
```


### Intermediate 1

now onto the intermediate

int key

```
openssl genrsa -out intermediate/intermediate.key 4096
```

int csr

```
openssl req -new -key intermediate/intermediate.key -subj "/C=US/O=Mikes Company/CN=Mikes-Intermediate-CA" -out intermediate/intermediate.csr
```

int ca

```
openssl x509 -req -in intermediate/intermediate.csr -CA root/root.crt -CAkey root/root.key -CAcreateserial -out intermediate/intermediate.crt -days 3650 -sha256 -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:1\nkeyUsage=critical,keyCertSign,cRLSign")
```

### Cluster CA
onto the cluster ca

cluster ca key

```
openssl genrsa -out cluster/cluster-ca.key 4096
```

cluster ca csr

```
openssl req -new -key cluster/cluster-ca.key -subj "/C=US/O=Mikes Company/CN=Mikes-Cluster-CA" -out cluster/cluster-ca.csr
```

cluster ca

```
openssl x509 -req -in cluster/cluster-ca.csr -CA intermediate/intermediate.crt -CAkey intermediate/intermediate.key -CAcreateserial -out cluster/cluster-ca.crt -days 3650 -sha256 -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign")
```

### Build chain

build chain

```
cat cluster/cluster-ca.crt intermediate/intermediate.crt root/root.crt > cluster/cluster-ca-chain.pem
```

### p12

note -password is set to `mypassword` obv use yours

```
openssl pkcs12 -export -in cluster/cluster-ca-chain.pem -nokeys -out cluster/cluster-ca.p12 -password pass:mypassword -caname cluster-ca
```

### Verify setup so far

Cluster CA ok?

```

openssl verify -CAfile <(cat intermediate/intermediate.crt root/root.crt) cluster/cluster-ca.crt
```

Should return:

'cluster/cluster-ca.crt: OK'


Intermediate CA ok?

```
openssl verify -CAfile root/root.crt intermediate/intermediate.crt
```

Should return: 

'intermediate/intermediate.crt: OK'

### Create secrets

According to docs at 18.6.1 the secrets names are prefixed by the cluster name

My cluster name here is `kafka`

So we need `kafka`-cluster-ca => `kafka-cluster-ca`
also `kafka`-cluster-ca-cert => `kafka-cluster-ca-cert`

We will do the clients CA later which has the same format.

Also for clarity the project I am using in ocp is `kafka-byo-ca`

Also I am using `mypassword` from above

Create secret with chain + p21

Creating `kafka-cluster-ca-cert`

```
oc create secret generic kafka-cluster-ca-cert --from-file=ca.crt=cluster/cluster-ca-chain.pem --from-file=ca.p12=cluster/cluster-ca.p12 --from-literal=ca.password=mypassword
```

label and annotate

```
oc label secret kafka-cluster-ca-cert strimzi.io/kind=Kafka strimzi.io/cluster="kafka"

oc annotate secret kafka-cluster-ca-cert strimzi.io/ca-cert-generation="0"
```

creating `kafka-cluster-ca`

```
oc create secret generic kafka-cluster-ca --from-file=ca.key=cluster/cluster-ca.key

```

label and annotate

```
oc label secret kafka-cluster-ca strimzi.io/kind=Kafka strimzi.io/cluster="kafka"


oc annotate secret kafka-cluster-ca strimzi.io/ca-key-generation="0"
```



### Deploy cluster Nodepools then Kafka

Deploy nodepools

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: kafka
  namespace: kafka-byo-ca
  labels:
    strimzi.io/cluster: kafka
spec:
  replicas: 3
  roles:
    - broker
    - controller
  resources:
    requests:
      cpu: "500m"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
```


Deploy Kafka

```
# uses kafka-nodepool.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka
  namespace: kafka-byo-ca
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.9.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: external
        port: 9094
        type: route
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
      inter.broker.protocol.version: "3.9"
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 10Gi
          deleteClaim: false
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "4Gi"

  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "2Gi"

  entityOperator:
    topicOperator: {}
    userOperator: {}
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"

  cruiseControl:
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"

  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "300m"
        memory: "256Mi"

  # Note this for BYO
  clusterCa:
    generateCertificateAuthority: false

```


Once deployed Kafka should come up, test and confirm this.

### Clients CA

Now that we have tested the cluster CA we will now do the clients ca.  The same steps apply in general but we can skip a few steps since we have created a Root and Intermediate already.

Lets add another directory oruse your own structure.

```
mkdir -p certs/clients
```

making the clients ca key

```
openssl genrsa -out clients/clients-ca.key 4096
```

now for the csr

```
openssl req -new -key clients/clients-ca.key -subj "/C=US/O=Mikes Company/CN=Mikes-Clients-CA" -out clients/clients-ca.csr
```

create the ca

```
openssl x509 -req -in clients/clients-ca.csr -CA intermediate/intermediate.crt -CAkey intermediate/intermediate.key -CAcreateserial -out clients/clients-ca.crt -days 3650 -sha256 -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign")
```

building the chain pem.

```
cat clients/clients-ca.crt intermediate/intermediate.crt root/root.crt > clients/clients-ca-chain.pem
```



Spend a moment to review the ca chain but if you followed these instructions it should be set to clients ca -> int -> root ca.


### building the clients p12

The UserOperator needs this for mtls

creating the p12

```
openssl pkcs12 -export -in clients/clients-ca-chain.pem -nokeys -out clients/clients-ca.p12 -password pass:mypassword -caname clients-ca
```

quick verify

```
openssl verify -CAfile <(cat intermediate/intermediate.crt root/root.crt) clients/clients-ca.crt
```

Should return: 

'clients/clients-ca.crt: OK'


