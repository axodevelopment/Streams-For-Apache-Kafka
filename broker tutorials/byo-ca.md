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

