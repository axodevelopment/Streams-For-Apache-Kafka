# Streams For Apache Kafka - Console 

Tutorials around Streams For Apache Kafka running on OCP - How to manage clusters in gitops fashion

## Overview of tutorial

How can we manage a cluster through GitOps.  We will explore Openshift Gitops (ArgoCD) and how we can leverage that as well as Tekton to manage our Kafka Cluster.

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

Regarding ussers as code, we should distinguish what type of cluster a user is being created for because KRaft vs Zookeeper is relevant to the KafkaUser resource.

Quick recap on users and some operations you may need to take.

KRaft does not support ACL's in the KafkaUser resource directly.

For a our app we will just need the following:

- user.crt -> client encrypted cert
- user.key -> client key
- clusters-ca.crt -> cluster ca
- ca.crt -> client ca for mtls (More for mTLS support for the user)

I created a user called `test-kafka-user` and the broker is in `kafka-tutorial-kraft-east`

```bash
oc get secret test-kafka-user -n kafka-tutorial-kraft-east -o jsonpath='{.data.user\.crt}' | base64 -d > user.crt
oc get secret test-kafka-user -n kafka-tutorial-kraft-east -o jsonpath='{.data.user\.key}' | base64 -d > user.key
oc get secret my-cluster-kraft-cluster-ca-cert -n kafka-tutorial-kraft-east -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt

```

`oc get secret test-kafka-user -n kafka-tutorial-kraft-east -o jsonpath='{.data.ca\.crt}' | base64 -d > client-ca.crt` this is the client CA and not needed for client -> server connectivity, the broker should already be able to mtls by accessing that user

the ca.crt will now contain both the which you don't need but going to leave it in for now.  You really just need the ca on the 

we should be able to test the connectivity before running the app

```bash

openssl s_client -connect my-cluster-kraft-kafka-bootstrap-kafka-tutorial-kraft-east.apps.axolab.axodevelopment.dev:443 -cert user.crt -key user.key -CAfile ca.crt -verify_return_error
```

---

So this is all fine but how do we go about a resource-as-code type of solution.  The great thing about Strimzi is that there are CRD's/ CR's that describe items like topics and users.

Looking at the folder you can see each type in `./ResourcesAsCode` in the root of this repo.

- mytopic.yaml
- testuser.yaml

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: mytopic
  namespace: kafka-test
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    min.insync.replicas: 2
```

-and-

```bash
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: testuser
  namespace: kafka-tutorial-kraft-east
  labels:
    strimzi.io/cluster: my-cluster-kraft
spec:
  authentication:
    type: tls
```

So here we have a user and a topic that we want to manage for our cluster located at `kafka-tutorial-kraft-east` called `my-cluster-kraft`.

The gitops approach is fairly simple we can create a happy path to our solution by creating an Application yaml for our gitops to use (ArgoCD)

File lcoated at `./ref/resource-as-code/application.yaml

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: resources-kafka-mycluster
  namespace: openshift-gitops
spec:
  destination:
    namespace: kafka-tutorial-kraft-east
    server: https://kubernetes.default.svc
  project: default
  source:
    path: ResourcesAsCode
    repoURL: https://github.com/axodevelopment/Streams-For-Apache-Kafka.git
    targetRevision: HEAD
```

Argo depending on how you have created and manged the users may need access to the namespace `kafka-tutorial-kraft-east` to deploy users and topics.

Here are some bindings you may need first

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-kafka-manager
rules:
- apiGroups: ["kafka.strimzi.io"]
  resources: ["kafkatopics", "kafkausers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-kafka-manager
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
roleRef:
  kind: ClusterRole
  name: argocd-kafka-manager
  apiGroup: rbac.authorization.k8s.io

```

Depending on your account you may need to give the application-controllor access to your project namespsace:

```bash
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n kafka-tutorial-kraft-east
```

Ultimately there are many ways to approach this, you could have a repo that does an helm install against a repo directory something akin to:

An example set of Applications for users and topics which you could wrap up in some kustomize.

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kafka-topics-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/your-repo.git'
    targetRevision: HEAD
    path: charts/kafka-topics
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: kafka-tutorial-kraft-east
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

As for the charts you could do something like the following

```bash{{- range .Values.users }}
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: {{ .name }} 
  namespace: {{ .namespace }} 
  labels:
    strimzi.io/cluster: {{ .clustername }} 
spec:
  authentication:
    type: tls
{{- end }}
---
{{- range .Values.topics }}
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: {{ .name }} 
  namespace: {{ .namespace }} 
  labels:
    strimzi.io/cluster: {{ .clustername }} 
spec:
  partitions: {{ .partitions }}
  replicas: {{ .replicas }}
  config:
    min.insync.replicas: {{ .minisrs }}
{{- end }}
```

with some corresponding values that you could use to create lists of topics and users

```bash
users:
  - name: test-kafka-user
    namespace: kafka-tutorial-kraft-east
    clustername: my-cluster-kraft
  - name: other-kafka-user
    namespace: kafka-tutorial-kraft-east
    clustername: my-cluster-kraft
---
topics:
  - name: mytopic
    namespace: kafka-tutorial-kraft-east
    clustername: my-cluster-kraft
    partitions: 3
    replicas: 3
    minisrs: 2
  - name: anothertopic
    namespace: kafka-tutorial-kraft-east
    clustername: my-cluster-kraft
    partitions: 2
    replicas: 2
    minisrs: 1
```

Syncing and statuses here are important so you'll want to ensure some of the following:

Ensure keep is not set on your helm manifest

- `helm.sh/resource-policy: keep`

If you set this ArgoCD will not cleanup what should be removed, which means topics and users will stay on the cluster even though the resource isn't there.  The operator reacts only to the delete operation to remove the resource from the cluster.

For most cases I would suggest `prune:true` so that if you remove an item from one of the values.yaml it will prune / delete that resource that is no longer being generated by running the helm or straightup yaml.

---

There will be cases where you need to wait until resources are finished reconciling before an application can be deployed.

While init containers can work it may take more time to resolve say a scaledown then an init container maybe configured to support.

Lets take an example that your application will be deployed sometime after the KafkaUser is deployed.  What you can do is do (as a happy path) 3 layers in your sync waves in ArgoCD.  Or even a Presync job to do some evaluation.

Tke the following job

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: wait-for-kafka-crds
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: wait-foe
        image: quay.io/openshift/origin-cli:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            set -e
            oc wait --for=condition=Ready --timeout=60s kafkauser/test-kafka-user -n kafka-tutorial-kraft-east
      restartPolicy: Never
```

I can put this as a PreSync or even on the syncwave prior to having your application get deployed.  This will ensure that your app can properly collect the KafkaUser details like certs or whatever for your application.

---

Hopefully the above examples give you some insights on how to approach resources as code for say topics and users.