# Streams For Apache Kafka - BYO Cluster and or Clients CA 

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

### Create clients secrets

According to docs at 18.6.1 the secrets names are prefixed by the cluster name

My cluster name here is `kafka`

So we need `kafka`-clients-ca => `kafka-clients-ca`
also `kafka`-clients-ca-cert => `kafka-clients-ca-cert`

We will do the clients CA later which has the same format.

Also for clarity the project I am using in ocp is `kafka-byo-ca`

Also I am using `mypassword` from above

Create secret with chain + p21

Creating `kafka-clients-ca-cert`

```
oc create secret generic kafka-clients-ca-cert --from-file=ca.crt=clients/clients-ca-chain.pem --from-file=ca.p12=clients/clients-ca.p12 --from-literal=ca.password=mypassword -n kafka-byo-ca
```

label and annotate like before

```
oc label secret kafka-clients-ca-cert strimzi.io/kind=Kafka strimzi.io/cluster="kafka" -n kafka-byo-ca

oc annotate secret kafka-clients-ca-cert strimzi.io/ca-cert-generation="0" -n kafka-byo-ca
```

Nowe we can add the `kafka-clients-ca`

```
oc create secret generic kafka-clients-ca --from-file=ca.key=clients/clients-ca.key -n kafka-byo-ca
```

again label and annotate

```
oc label secret kafka-clients-ca strimzi.io/kind=Kafka strimzi.io/cluster="kafka" -n kafka-byo-ca

oc annotate secret kafka-clients-ca strimzi.io/ca-key-generation="0" -n kafka-byo-ca
```

Now to test the deployment...


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

  # Note this for BYO now adding clients ca
  clusterCa:
    generateCertificateAuthority: false
  clientsCa:
    generateCertificateAuthority: false

```


Once deployed Kafka should come up, test and confirm this.


#### NOTE !!!

If you reuse pvc's for fresh installs you'll may come across an error like:

```
Invalid cluster.id in: /var/lib/kafka/data-0/kafka-log0/meta.properties. Expected Aqi5S-O4QOuXGSO-ByTO9g, but read mP42GcQJSh-W8OkoJDBQaw
```

Invalid cluster.id is key, in this case a previous installs data is on that PVC and Kafka init doesn't wipe a pvc because the scope is at the pod level.

This will happen especially on:

```
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
```

That is on the kafka nodepool

Regardless delete the pvc is you come across that.

### Testing clients ca

Now since we are using MTLS or want to we can test this in multiple ways.  Lets do the easy way which is creating a Kafkauser and we should see a cert issued by the chain you specified.

Note the label

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: test-user
  labels:
    strimzi.io/cluster: kafka
spec:
  authentication:
    type: tls

```

oc apply the file

We should see in the yaml a CN value = to   username: CN=test-user

A secret will be created called test-user and in it we should see our client chain and an issued cert from that.

Another test to pull the user.crt

```
openssl x509 -in user.crt -noout -issuer
```

Should return in my case at least: `issuer = C=US, O=Mikes Company, CN=Mikes-Clients-CA`

If you have used the steps exactly you should see the full secret for the user accoutn like this

```
kind: Secret
apiVersion: v1
metadata:
  name: test-user
  namespace: kafka-byo-ca
  uid: 4c2e818f-dfb4-47a4-8e33-ae0351fac62c
  resourceVersion: '178988252'
  creationTimestamp: '2025-12-13T01:12:02Z'
  labels:
    app.kubernetes.io/instance: test-user
    app.kubernetes.io/managed-by: strimzi-user-operator
    app.kubernetes.io/name: strimzi-user-operator
    app.kubernetes.io/part-of: strimzi-test-user
    strimzi.io/cluster: kafka
    strimzi.io/kind: KafkaUser
  ownerReferences:
    - apiVersion: kafka.strimzi.io/v1beta2
      kind: KafkaUser
      name: test-user
      uid: 9400bd5a-7258-4aa3-864c-9d4fbce8ae07
      controller: false
      blockOwnerDeletion: false
  managedFields:
    - manager: strimzi-user-operator
      operation: Update
      apiVersion: v1
      time: '2025-12-13T01:12:02Z'
      fieldsType: FieldsV1
      fieldsV1:
        'f:data':
          .: {}
          'f:ca.crt': {}
          'f:user.crt': {}
          'f:user.key': {}
          'f:user.p12': {}
          'f:user.password': {}
        'f:metadata':
          'f:labels':
            .: {}
            'f:app.kubernetes.io/instance': {}
            'f:app.kubernetes.io/managed-by': {}
            'f:app.kubernetes.io/name': {}
            'f:app.kubernetes.io/part-of': {}
            'f:strimzi.io/cluster': {}
            'f:strimzi.io/kind': {}
          'f:ownerReferences':
            .: {}
            'k:{"uid":"9400bd5a-7258-4aa3-864c-9d4fbce8ae07"}': {}
        'f:type': {}
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZlVENDQTJHZ0F3SUJBZ0lVUlp5N2FaOHlpR2xZZjh1RnJyayszMXB0LzVzd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1JURUxNQWtHQTFVRUJoTUNWVk14RmpBVUJnTlZCQW9NRFUxcGEyVnpJRU52YlhCaGJua3hIakFjQmdOVgpCQU1NRlUxcGEyVnpMVWx1ZEdWeWJXVmthV0YwWlMxRFFUQWVGdzB5TlRFeU1UTXdNREEwTlRWYUZ3MHpOVEV5Ck1URXdNREEwTlRWYU1FQXhDekFKQmdOVkJBWVRBbFZUTVJZd0ZBWURWUVFLREExTmFXdGxjeUJEYjIxd1lXNTUKTVJrd0Z3WURWUVFEREJCTmFXdGxjeTFEYkdsbGJuUnpMVU5CTUlJQ0lqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQwpBZzhBTUlJQ0NnS0NBZ0VBMHB2cHlVQU5zZEN5V0RXcm9oKzh3K2JIYk9Yd3FXYW5mbzFVREFnaGM3Ym5UWHJYCm92R2FUNjU2cXRTYTN2bGh2R2E0ZmRReE5ocTFVSkMwQUxhQXRocXU2cE50ZUxseE9zb2FFR1ZXTy9lT3dIaEoKSGNWTWtwcXdIRFF2dDJ3eVdIWHJaeE9vK3NHZysrZmJ5VUN5Rm9xNUFYQUFXMlJOaWtIczA5dCtFbEFiREZ3TgpOR2tFbnpTdHdQWDY0eU1iMUNta0grdkxBVkh4N1c2L1lrTzVtSFN3SXZiaE1CeFkvWWo0RVhVdWtNdkN5b0E4CjBkQ01DcmxTY20weS9UT0N4czZEeHdMUTk1aE1KM0RGT0EweFQ5ZjduVGVrTG43akVMQUpNVFE5SzFQNEpCeDYKQ012b1B5a21FUFg2ZlpDUXp0cnJqVGxVRDRBcEZrUmh1Zi9lbzRvN3F5M1Zndm1VUHhxTkkzKzBpQnIzV05DMApzUyt0cHVKSE9YaTladTUvTy9MR2dCR1dQYk1SaXVLTDI1Z3Z5MCs2TCtIOUowajdpcGdKSldKaHNBTGFQZ0JpCkxJZXBTWDQ0NmxYMWVsVXVWN3NncUcwMmRxOENFWW44dnM0WWp0eTA1MHlDTGVKU1BPbmsrbTA4UXNPQk5OdW4KRFkzeW9TSlVUNXk5eTdncHhaTGR1cTJyYXgyY2JQRjZ6WXU0N2hVUVlBOGV0MzdRcW9rNEI4SUl4clU4Vnl6WgpIVDBDYjZTM2hqUGhZTG1nT1o4bEV3YjM3emVuTGwydVE4UitvcTR4N2hJVzVGa3NGZ2w3eS85bzE1VERRSkwyCnJnT0V6a2xTbVJNTlhWOERsM0JqTUVSejhOZVhpcXIzN3pZSTYwRUM3OHo1cHd0REVTRUQwNi9BcmUwQ0F3RUEKQWFObU1HUXdFZ1lEVlIwVEFRSC9CQWd3QmdFQi93SUJBREFPQmdOVkhROEJBZjhFQkFNQ0FRWXdIUVlEVlIwTwpCQllFRkIrbE1ZRTZZMTNNajBIMzZIWFQxSmVIQ2dTTU1COEdBMVVkSXdRWU1CYUFGTXVrQmlZZ3RndlpEbzVJCkU4TGpNRzZEaHNqck1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQ0FRQUE5NFB2VlE4c3lOTnNub3pka1NxVlZNMUUKSG9mbmgzVjdHMERVZytUbmFBT1RCNXpjWDRwUUU0MjNZY2Q4Q2piTnJJZEprNUVKNjFyQnVqZndVc0xCSVJBdQpRRUJQUUZOalI0STFyZzJibzJnS3ZKaDdDLzhyVFJZOFBLZ09lZHplbnhLaFNHckIrdmlPTjFaMkh0QlBDSXl0CnMrVU9DZUROcG1pUnFydGdmK2VTNndubTU1Y2Y0TjZpRUNMS1JQYVNxY2E2YjcxaXA1WVdUVXdUdmNYY1hveUEKNlFRa1hueVJKQ01mdVdNaDk4SWFOcktKb01IQjEzZFU2M21Jb2pOdllQb0t3anRQN3hyb1hzMG5McVliTHcycQprTmo1NHYxRU40cWFEZlBQR25tOE1pbzZzNU9adW9JN29CL1RtSWN6RjREa09TdWNCa3RmeDh3R1hKWmU5NnlTCkFxZEtXY09YK01UWkRMSitBL2xSN3JmNjhOb2JyZmIvOUYvcmxuNUVWMDRMK3hZbGZwYjZwNmpEdU9Eb1pHR2sKdlNIOFllZjlNcVRzK2grei9oanpHYkUyM2t3Wk5Ba3laWGtyV0YyelUvV3J3Tk5wSjFIbUVMMFF1Zml6aVVobQpiQzBWWTNIMWZMY2lCbmFFZ0hSSERoRlNsemNvVGQwaVNYNnpLdVRIWGJYOWlYT0RyVWQxOG9QMkZ4UWhHWjRTCjFpUEIwREQveVpiZk9mbWNuT0w3TGJmN3Q1MlNiMnAwZjVMK0Q2VllYL2pMbkkyV0p6S2Z3dmtUQkE4d1k3MGUKZXpQT09zWXJNb3NQOCtRWFNkNVlDUUdFZVNERlUrMlhZeXRLNXFSWVRsWDBMdm5WNGN5TzFQcjZ3QStHRjdCYgpua2szcjJ4UzV3K3JzMVFlUlE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlGZGpDQ0ExNmdBd0lCQWdJVU9DVXJvcjV4VU9CSlRMMko4ZGhZZERXMENMTXdEUVlKS29aSWh2Y05BUUVMCkJRQXdQVEVMTUFrR0ExVUVCaE1DVlZNeEZqQVVCZ05WQkFvTURVMXBhMlZ6SUVOdmJYQmhibmt4RmpBVUJnTlYKQkFNTURVMXBhMlZ6TFZKdmIzUXRRMEV3SGhjTk1qVXhNakV5TWpBd056TXdXaGNOTXpVeE1qRXdNakF3TnpNdwpXakJGTVFzd0NRWURWUVFHRXdKVlV6RVdNQlFHQTFVRUNnd05UV2xyWlhNZ1EyOXRjR0Z1ZVRFZU1Cd0dBMVVFCkF3d1ZUV2xyWlhNdFNXNTBaWEp0WldScFlYUmxMVU5CTUlJQ0lqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FnOEEKTUlJQ0NnS0NBZ0VBcWVNTFMxWU1paWVucEpHVzA0L0swSFdFd0w4R3FwUVpTMHMwMU9pUHRDREVtdzFYcmJxVwoyTk4weCswSjluS2szbk9RRGh4N0lVclMxVUJRSlZlcGhJb0JwMU5kcm5URGwzNHVtZkFKUXJ5WVZBQWtSSEVaClB1c1ZMUGFuSEduWE5qYytwZDE3bzJ0VVpxSURpVVVoUHhudjZvblpKcXhuenlKNUpUN1c1WjJiVjg3M2ZMS3QKdThLTmp1bzJSaGpLdDN3blEwSjdsb05pYXJqVlU4Rk1TbmxFRUNUTXQ4NmdwZm12U0dFb0RkbjJ2VVhqV1pNVwpHNkZySmVQdUhKQkpVV1VsbS83Q1hFR2ZMR1FQcUpPcmJPdDhrZGdkYndJRnVmZ0hleXRCQ0cySDY2M0dFVktxCmRscVo1RlhZVWtUTXBUUHo3UUU1amxLNzFkMERXbC9FdzZmWmZGRkVwTUZEZ1RLd0h5SEkwdWsvdW93VmVSeTMKSzhMcjJBVTFJV1Q2ZmgrUEJQbWNIR2VmZDE5TzVCd3B1VUsxVEtrdEcxYjN2d1RWUXlkZG1TSmRINDBuK25GNQpUbTVBdUhCYWp6eGRKL2VrQTYyeTVudjJSUXNHajRLaFJVaGhKWEZyNWFGV0kxQTFoenlSSjhFdHRsWFpQQ2l1CkgvV2FpWjM5R2hPOU04QVR1WDRaTHlma3Z3RmNmZ3dvOFJXS1JMdWkveXZsd2c3Q0JYWGcwcEFlY0tNeDViKzQKQkNsQUZTYWtka0s4cWFDOE1iemhpNFkyY01FUVBCYWVjSEdOcUhmRGlqLytLWitTWVh1Yk40ZnA1K3ZjU1krWQo2MGpkNDUzQ2kxVTRZOGFZMWFQV0R0NjlpMDhzUmlGajdyTWNBc2Y0djRtcURTbTRBRUpwRk9VQ0F3RUFBYU5tCk1HUXdFZ1lEVlIwVEFRSC9CQWd3QmdFQi93SUJBVEFPQmdOVkhROEJBZjhFQkFNQ0FRWXdIUVlEVlIwT0JCWUUKRk11a0JpWWd0Z3ZaRG81SUU4TGpNRzZEaHNqck1COEdBMVVkSXdRWU1CYUFGRGNWRnEyakFVai9wUVVXTjVJdApaOXc3LzNoaE1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQ0FRQ2EwWjVKcnFUSU9qK1NsSE54U0Zlc1hpeFNOOEZMClBWclRZNi8rWjZnSVUwOVQ2V1pIZXhqTXMvbjZJUkVzbjVPdTdZYW80eGhMYzNOcUhBc2tEYnBRZjdDNlRIVXkKaUpPbnQvOTUwb3BBYk96cWxOMERJMThWQmVSNnd6aG1zMkM4djZjTjNLZnFVT1pDUHFsZkJQVS94bWNJS0QrZgo3LzkwTmtnZmhLVVA2cFdmZklUTEx2MndranNEUXY3OXhVS1RabnVsSUc2aG8yT1pGU2dQa2dJUjBaUEdOSDlGCks2azBZa2FzRU1xTGFOQ0ZJSWN6d09vTVRVUTB3aHZpQ281cGI2akU1R1lGaVJjTFBRZmtmZlVBbWdFbEU0OW0KT0d3dk5pZDlwd1lVdERvQVl1M1hrSi9DejRnWmxjVkFtRlNxeXdwZGtIemRnQjE1b3lHV3pJOEJlQ3hYRnl4WApXeVRlbHlVMmErZlJZK0RFdlNidEhXMkgzKzVkakdOUklOSTRQOE1PbmFYa3dzS0pOaGc0QkhDeTBncEtQeHFlCjd0L0hVaDZHZEcyRmVhdlBaNTVpTFFvOTdMWnlYdDd4UDZRODduZG4wNGxVVTNLQndZMU4xVU5XbzY3UzJzRFkKNzQ0dlNFakxrdlNLWCtNMWczTkFOS2t2dnYzSlYyNXU5c1pZZkFLUmVFQWRsMVcyU0FVaUwyUFlmNWFsS1lXMwpLdzljOE03bFM2OGJ5aEJ4Y0x0SWJuRTdRb2cydytSMnVHenhOVFVHUDZUZ1ZKWG1qSWNDQi85QzBuOE1mdVNTCjJka3pCQWJHSDYxNjA0clFmbVZkZVQzSWo1VHhRSTRyS2JSZEJZRFBaREhkRysybzA3bDgxcHh2ZUExck1xd3kKMFJnUGxNY3d0N3dsVkE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlGYmpDQ0ExYWdBd0lCQWdJVUlZNWpWL1VJNUFoenFCejRteEZSMmx1ZXhPd3dEUVlKS29aSWh2Y05BUUVMCkJRQXdQVEVMTUFrR0ExVUVCaE1DVlZNeEZqQVVCZ05WQkFvTURVMXBhMlZ6SUVOdmJYQmhibmt4RmpBVUJnTlYKQkFNTURVMXBhMlZ6TFZKdmIzUXRRMEV3SGhjTk1qVXhNakV5TVRrME9ETTFXaGNOTXpVeE1qRXdNVGswT0RNMQpXakE5TVFzd0NRWURWUVFHRXdKVlV6RVdNQlFHQTFVRUNnd05UV2xyWlhNZ1EyOXRjR0Z1ZVRFV01CUUdBMVVFCkF3d05UV2xyWlhNdFVtOXZkQzFEUVRDQ0FpSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnSVBBRENDQWdvQ2dnSUIKQU05bjU1SU9rSGtaR2dDeGl2WmVWK3dkcEJxbDgySTA0WlFIWW5JLzlzZ2RwVkdiQ3pqVGNQR2QyeGRlOTZXVAo1MUlJU2VMdkNESnF3d1RoVmRoUjdxR3lONWl1TmpITVYzVWNtaGVMMnZCNTdQbEZ3OGZQdmRDaVJqNHlSSmt5CjZXamJudFdoLzUvUDlGTm1UNkFDSVoxdGNENXBXZUl6Vmp6Tk9PSmxLS1lYdnMxM3VxZ1psbUhzNG1oMGxTNlIKREVZZWRHU284MVdSYXpsZU9nZlhpYmU1cEpyRVhrWlpHOTR5a2FRbGJsYStQZXJGbEhEQ29zeWdMcWFZZzhtcQpQM3U3WWVHMDJpbnRCSWVHZjRDbk5YOXUxU05wTUZaNzJtbWlzRUlzb21MRkx0TUFtRmQ2NWFKcHZWaituNzE2CkhtWEtaRG4vWDlHSFlaRkV5eWxTanJNdkgvbisyR3NxeUZXcnFTOW5HY1pqYUZmbkhWZm5CZ1pIS1RLUjZ4c0cKQm81OU9QeE02K3lzRGhnYzFEYjIySm5EZEtqZVlrRlVIRWdsZ0hYRTAweUtqVXl1VlVrYmhFQjk2S3VDc2dkZwpDR0c2VmRZRnRiK1dhRVNsZUc5TnZ3Mk5PaDdoSXI4RjNPVWp0RjVWMmh0Qi9Na2VyRW1FRWpNQUZCT0k0bldVClNaRWdxanZmd0ZmTkZGV05CdjQ1aEk2eEs3SkltUTZBS041U2JkSnJ0NWdYZUVSNERJSlZRaGoxd0hIZTNyd0IKZlBpOVlhQzZCUXZZTVBZM3ZsaEUrNUhhTS9jZU1lS2wvcExLTFdWU1NuSStnZ2dMUk5ESmVQVmtLQ05sZ1cyQQpWQVVxTW9OMC9xNDVGWU1rOENrY3ZFb2l3eHQ3dFdDbFkwMmU4OUVRTVRKbEFnTUJBQUdqWmpCa01CMEdBMVVkCkRnUVdCQlEzRlJhdG93RkkvNlVGRmplU0xXZmNPLzk0WVRBZkJnTlZIU01FR0RBV2dCUTNGUmF0b3dGSS82VUYKRmplU0xXZmNPLzk0WVRBU0JnTlZIUk1CQWY4RUNEQUdBUUgvQWdFQ01BNEdBMVVkRHdFQi93UUVBd0lCQmpBTgpCZ2txaGtpRzl3MEJBUXNGQUFPQ0FnRUFoQVc2cFUyYWhGZXVPOVhFVmZqQ09OS0l2UnVQWmpKWTFvNlgwbHppClB5L0daUkFpVGRoZE9yeHArVGpJNkRWaWZ2dDA1WUNXK3hqVUtseU1HTDZ0Wjh6U2pabjMrd1dnQUxueTFIbjEKcTRJS0NseUVTQk1tNXA2YTAzdmFoZXhad0VsVzBoOEZ1UGptTnRQQjJVRzljdWUvanN5WXlxZjUxSDhYYzVwQwpvYjlja2x0R2xuQjhwR1grNDZkQm5vMWFYQ09sM1FSOUowRlVJUGZzcEthUW9xVnNleHF2bS9FN1RjSUIxREV1CjQ2ZFF5Z0g0cUM2bGdKTnA4VFNqK2pON1o1VnMydGJQcWFXbTN5S2Vxb0ZnZXJaN3ovSUIxeE1kRHl1UVVsTkUKaHpsWW9QNll2c1RSZ0NSSnhTd3NQVVpxUjBWTzNoUnFYa3dnYU45eDZRVGxsT3JHcG9zRkh6Rk5GSFJBS3E2RAo3Q0RscGlJZlozaVhieUpUU3M1N0NqSzhNdVUzYW1jZ2Zjd3RGY3hLSlZxSkI4aEF6d2s1a1hVQlBzVGNCc3VDCmVHbzJ0czZkUjVjK1B2SVFqc1RPN2l5ZXNWa1V2amN1MVhOTjErS3lNZW05UHp4UTJzOWhJSGR2WWh0Q0h0RDQKT2REb2NVbGQxUjdsODZUMmlYT2NFdHFkdk96MHpqSnVQdXV2VVJ0WHZqU3EyKyt3WjVqOXF3bFFFRGk2L3dQMwpQeWdJNFdiSUxwOTlYend2K28xcWdqTjV1UndoU3RuMjU5Y3dMV1BBRzJVRzI2c2NsdVRtN2ZNcHlSc2NlcW5XCnpHUzB6NUJWMXBxenMxUVl4L0pZTlR0TnBKSXFBNjJRdUVTOW8yTk5teXFBbHlFWGJJamtLOURLT3NwaTNscGUKRHUwPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  user.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVRakNDQWlxZ0F3SUJBZ0lVSlJyZDZJeUFaMjFPL0ZEWmZid2VFTlFDRElrd0RRWUpLb1pJaHZjTkFRRU4KQlFBd1FERUxNQWtHQTFVRUJoTUNWVk14RmpBVUJnTlZCQW9NRFUxcGEyVnpJRU52YlhCaGJua3hHVEFYQmdOVgpCQU1NRUUxcGEyVnpMVU5zYVdWdWRITXRRMEV3SGhjTk1qVXhNakV6TURFeE1qQXlXaGNOTWpZeE1qRXpNREV4Ck1qQXlXakFVTVJJd0VBWURWUVFEREFsMFpYTjBMWFZ6WlhJd2dnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQUE0SUIKRHdBd2dnRUtBb0lCQVFDaGJyUTE1dGpNNTYwMlpiVTFLNWtna2l0QVc2K3c2OGd5OU9IZlljaU1ibXQyaVVUNgo5NlA3YnU0L2RaQXFJQnl6SG5nVWdod2tGdUg4ZkZWZktxYnY4REU1K2NVMzJmcmlJbzRqS0dsRi9aeDN4RlMvCmQ1bllKaENaUTlTY2s3Ymx1cWtTWkw2Qk1xVG5nOExsWmVGU2o5TEMrcVdORjJoR2EvV1l4bi8xU1hrWVFhT2oKYnY1NzhxWk9uYlRMMlJrUUtOZUdMaVdYRk15THNDZjN1cWxocjUzcmk1c0pMenMwWTBSamFBZUc2cjJIUklicApyU2k0VUdJTkdqRUo3YnJSNENoVXBnamxYZGtCbDlhdEIxTW9QdlJuUGcvNjZjU3owZktZekpmVWRqUFljQXNzCnFEUTJteXZHWnpUdC9pVnIrVi9sbzF3Wk1TVURZa0tqTmhhdEFnTUJBQUdqWURCZU1CMEdBMVVkRGdRV0JCVFgKVjc0NTFxZU9sRDFyQVZadzB1M1RxUDI1OURBTUJnTlZIUk1CQWY4RUFqQUFNQTRHQTFVZER3RUIvd1FFQXdJRgpvREFmQmdOVkhTTUVHREFXZ0JRZnBUR0JPbU5kekk5QjkraDEwOVNYaHdvRWpEQU5CZ2txaGtpRzl3MEJBUTBGCkFBT0NBZ0VBZFd4REhvOFY5OCt4aFpvNHhEWmYvWXJ0c2ZLTGZrZyt5bWV1Y1Z5a2tEYVdSSXVVTThQNnJVbUsKOHYrcmJnK0lVeEF1cTlFMk45bjJGNXIxbm9SV2Z5TytHSW1aRkZLS2N3Sy9IdTk2RGV4dTZjb25nZ0FnSGtrSQpJRWRDcVI4aFA0NTdjTVBmZ0RneDVOOE44Qk4rMTM3a3A0THNjVjd0K1F5aFBhSitQOHp5VEFzc1JZa29LN3Z1CkhidElLQWpHQVBQRG1Qdzd5eklwdGRNaTZZYW0va1hYbFRiWDBjYU4xbUMrbVQ5RDNLMW5aSmxOQlIzUTBhdTIKcC9IRGN0V0Q2TlBVUlBZLy82dVltZE1SVGU2Mmg4d25FejBtU2tDaW1LVUl5cDVTN3hQbUlISGlta09pbjFIKwpvZUw4djByaFgxWDYxK0xoeW5SdUNTWWdSR29Lc1JLMGRyZ05INjhQaUlQUU5FOW9XZTBTcXlVbThhdngzZEFUCjM3UUVVZ1RXR1MyaE1DUTRiOE9RWGdKbGQ2M2Zzb3JvemZpbGRoYzdCRytXWmptUy91MGM2dzB6MWx5NlhqYXIKL2NhdlRIZFZLT1ZLdjc4UkQwKzBnUGJDZ21YRERmOTdScEhWNE0rWklIaW9OdS9lbllhZXhDQXAvSU1DNHR1QgpLRTViNkRlNmFSdjBmOGJMeHZ3VDJQa1JRakVVWEhqb3FFQ204Q0dsM0VuODFoTEt6bTN0dVRNY3NhSStKM1loCkpmZHU1V0ZKM1Qrb0dGY0NuUFQ2UTVlaEl2QVZaU2NoQVYwUnl2Y01DUDVGREl4RHJORWE0SFhMMnBXdkJHT3kKQnh6TlJSN29DaHlJR0lWZU9Xekxsak54SW9QT0pzL0pvS1ZUN3FhVTcyN2RkU3ZFVDlRPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  user.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2QUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktZd2dnU2lBZ0VBQW9JQkFRQ2hiclExNXRqTTU2MDIKWmJVMUs1a2draXRBVzYrdzY4Z3k5T0hmWWNpTWJtdDJpVVQ2OTZQN2J1NC9kWkFxSUJ5ekhuZ1VnaHdrRnVIOApmRlZmS3FidjhERTUrY1UzMmZyaUlvNGpLR2xGL1p4M3hGUy9kNW5ZSmhDWlE5U2NrN2JsdXFrU1pMNkJNcVRuCmc4TGxaZUZTajlMQytxV05GMmhHYS9XWXhuLzFTWGtZUWFPamJ2NTc4cVpPbmJUTDJSa1FLTmVHTGlXWEZNeUwKc0NmM3VxbGhyNTNyaTVzSkx6czBZMFJqYUFlRzZyMkhSSWJwclNpNFVHSU5HakVKN2JyUjRDaFVwZ2psWGRrQgpsOWF0QjFNb1B2Um5QZy82NmNTejBmS1l6SmZVZGpQWWNBc3NxRFEybXl2R1p6VHQvaVZyK1YvbG8xd1pNU1VECllrS2pOaGF0QWdNQkFBRUNnZ0VBSEpQbU01K1c5Q2swUk91NjZUdDdXMlI4NHlNSUJtd3JEL2hCYnlQV2xxT3EKZ3Z5NGVTd3pPNXpPOE8xOU5MUGtHTUoxVS80WGdMMExTd0R3dFF6dUtnNHRqTHVlY2UxMUNDakJYRkI0V0hOVgp5bTczYU1EQnU1MjdkUUpveGtJeEQ5aWNPeDBhQzNHZGR6MmdXQzlSdE9Xd2xDTStnT3hxb1lMVm9yTExMcThICmZUNWZPdWFKSys3ZTlWS2VYd2taT0lZdGNJS1NQY2JLYmN2SWFvYU42dXJFaUhUVGNpNjlsTW10U0t5a0JRMkEKcVNzYkNhMlRKczNlUHpYMklkcm1Hc1FZRjBabjRuRFl6NDRScyt5amlLWEhRc25Ud3lrT2E4cWZ6VVB5UVY0Kwp5QndWQmVrSnZHQUZWWVFsU0dlb1FTQ2tBYTVKYUJhdCsyK0ZLdWpIQVFLQmdRRGdWcVdPZ3pPNmc0REhCZDA0CkhKVWtqb2JTMzliK2gzQk54RlBEeFVHMXJqNUhaVU1ING53UFRtQkJpSWdFVGJKc1JVcHVNKzNCSHNBNU45cVoKREwveE4yakFOTDlDTHpPWnp5K3JSNXhkNXRxeklZNWV4ZVRzLzE0MDk2WGd3eVlmTTd1WVBnWmczUmx1bEtrYgpycUVWdjVUbGVLa1ZPdm9RRU9ESFV6TkRBUUtCZ1FDNE4wTGZLZlRpdExKay8yZHVicTRVWTNYYXhxMXBvdkRtCkFzdW9nQU1ESzJvK3QxOGlNUVJZV3NjRXhGWXZ0QjB6RmE5eHBRN2NmVjVLUCtva2VWSEhCM01pZWpHeVVpQVgKVTVML3JjTjBCQ1dtdVJmWXJzd3l4SUptdTJCVCtaWXpzV1BlS25INUdFWnRjVmpqVXREUnpsMW5PaDJLa3R2dQp3VEJreW1UUHJRS0JnQ05aby9iajk4ZkJKdzYxZnRsenI1QzJJTXFqMlFYOG81YXRoQ0dLT01OL05ITWRvc1ZnClMvcEJlR3Q3THl1MmJwSWZEUTUyZ2xWM0dnVXFKdmtOQ0VYalhFOUZRSW9XVkFROW9KNVZ4MjhJakpmRGh1S3EKUGx1V0ZlczB4dCszQUkvVUlCQnFYYWp2emkwZG9kUXAzVnBHK1JoN3ZmRUpmUlFCQk5xRDRzVUJBb0dBREJZMApJUWhUdFB3K0tEcEp3d2tvQ3RacnlTcjMvZEpmRS9oaS9HOUp3MDk1N1J1QzltOVk1YU12STdUdUlyc2luMU53CjYzZjAvYXFNSVRzSVZkUlA5VXNiMXN0RnIzbUwrWHZXVFVoTlpyTk85UjEzM3hPNCtpdkNrcE1Bd3dIQlJTc0MKYm5WQ2ZTR0duVyt1Y1Z2aHI2Sm1wbnM5clBYdDBFQ0V1RmcvUFJFQ2dZQUVIekxFbDVUL2NQdk5RZW1JTDNENQo1Y24yUmxIVWoxMFpZZzRabFV6Q3NNZTZ0MlJqTnMxdDE3dDFWM1pDYlJiSWFjMGxCcFo2THA1UCtkK2x6dDFrCmtmamhmOEN3bUhVTTBPZ2t2Ui80cWpkaVgzWm9wT3pEN0FEanE1OWdkRHVsai8yeFNya3F0ZHU3TWZVejZIQzAKTlBUN2wyYlRUeVl6Um1WL1U3ZWduZz09Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
  user.p12: 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
  user.password: T0VqdFlNZ0NVTVV5VDNpZ25kNHFxeU9IZ0RXRkhJa3E=
type: Opaque
```


Next test from another issuer

### Verifying existing secrets for compatability

We are starting from purely a cluster only point of view so we should create a new directory for our inspections that we need to make.

Making some directors

```
mkdir -p inspect/{cluster-ca,clients-ca}
```

Now loginto the cluster via `oc login`

My kafka resource is named `kafka` you can see that in the above steps.

My project is `kafka-byo-ca`

You can check by looking at `oc project -q`

Lets download our secrets

```
oc extract secret/kafka-cluster-ca-cert -n kafka-byo-ca --to=inspect/cluster-ca --confirm
oc extract secret/kafka-cluster-ca -n kafka-byo-ca --to=inspect/cluster-ca --confirm
```

```
oc extract secret/kafka-clients-ca-cert -n kafka-byo-ca --to=inspect/clients-ca --confirm
oc extract secret/kafka-clients-ca -n kafka-byo-ca --to=inspect/clients-ca --confirm
```

Overview of what we should have seen:

```
ls inspect/cluster-ca
```

Should give you:

```
ca.crt  ca.key  ca.p12  ca.password
```

Now for clients

```
ls inspect/clients-ca
```

Depending on your setup it must at least have

```
ca.crt
ca.key

ca.p12 --optional
ca.password --optional
```

Testing the cluster ca

```
openssl x509 -in inspect/cluster-ca/ca.crt -noout -text
```

You should have:

- Basic Constraings -> `CA:TRUE`
- KeyUsage: `Certificate Sign, CRL Sign`
- SAN should be empty
- Subject: Something valid
- Issuer: An intermediate or Root


Cert modulus matching

```
openssl rsa  -in inspect/cluster-ca/ca.key -noout -modulus | openssl md5
openssl x509 -in inspect/cluster-ca/ca.crt -noout -modulus | openssl md5
```

Shoudl return the same value.

Now to check the ordering:

```
openssl crl2pkcs7 -nocrl -certfile inspect/cluster-ca/ca.crt | openssl pkcs7 -print_certs -noout

```

For me we can see the order Cluster CA -> Intermediate CA -> Root CA

```
subject=C=US, O=Mikes Company, CN=Mikes-Cluster-CA
issuer=C=US, O=Mikes Company, CN=Mikes-Intermediate-CA

subject=C=US, O=Mikes Company, CN=Mikes-Intermediate-CA
issuer=C=US, O=Mikes Company, CN=Mikes-Root-CA

subject=C=US, O=Mikes Company, CN=Mikes-Root-CA
issuer=C=US, O=Mikes Company, CN=Mikes-Root-CA
```

Lets check the pkcs12

```
cat inspect/cluster-ca/ca.password
```

or we can do some script but don't relyon this

```
printf '%s\n' "$(cat inspect/cluster-ca/ca.password)" | openssl pkcs12 -info -in inspect/cluster-ca/ca.p12 -passin stdin
```

PKS keytool test

```
keytool -list -keystore inspect/cluster-ca/ca.p12 -storetype PKCS12 -storepass "$(cat inspect/cluster-ca/ca.password)"
```

Should return something other than an error.
