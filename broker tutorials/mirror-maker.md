oc get secret mirror-maker-user-west -n kafka-tutorial-kraft-west -o yaml > exported-mirror-maker-user-west.yaml

oc apply -f exported-mirror-maker-user-west.yaml


--- ca's

oc get secret my-cluster-kraft-cluster-ca-cert -n kafka-tutorial-kraft-west -o yaml > exported-west-cluster-ca-cert.yaml


## requirement
You need the user authentication and the CA of the broker listener, they need to be copied over and accessible in both namespaces.

```bash
clusters:
  - alias: "east-cluster"
    bootstrapServers: my-cluster-kraft-kafka-bootstrap-kafka-tutorial-kraft-east.apps.axolab.axodevelopment.dev:443
    tls:
      trustedCertificates:
        - secretName: my-cluster-kraft-cluster-ca-cert
          certificate: ca.crt
    authentication:
      type: tls
      certificateAndKey:
        secretName: mirror-maker-user-east
        certificate: user.crt
        key: user.key
  - alias: "west-cluster"
    bootstrapServers: my-cluster-kraft-kafka-bootstrap-kafka-tutorial-kraft-west.apps.axolab.axodevelopment.dev:443
    tls:
      trustedCertificates:
        - secretName: my-cluster-kraft-cluster-ca-cert-west
          certificate: ca.crt
    authentication:
      type: tls
      certificateAndKey:
        secretName: mirror-maker-user-west
        certificate: user.crt
        key: user.key
```

Meaning for each cluster I need the trustedCert (ca) for that connection to that cluster
We also need the user auth, in this case I am using tls.

---

Authentication on listeners need to be set, in my example im using tls on (`Kafka`) resource

```bash
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: external
        port: 9094
        type: route
        tls: true
        authentication: <-- here
          type: tls
```

If you don't set authentication then you auth as `ANONYMOUS` which does not have access to many things.

---

If you want to sync checkpoints / consumer group offsets etc you can set that here

```bash
mirrors:
  - sourceCluster: "east-cluster"
    targetCluster: "west-cluster"
    sourceConnector:
      tasksMax: 1
      config:
        replication.factor: -1
        offset-syncs.topic.replication.factor: -1
        sync.topic.acls.enabled: "false"
        refresh.topics.interval.seconds: 600
    checkpointConnector:
      tasksMax: 1
      config:
        checkpoints.topic.replication.factor: -1
        sync.group.offsets.enabled: "false" # <- set to true
        refresh.groups.interval.seconds: 600
    topicsPattern: ".*"
    groupsPattern: ".*"
```