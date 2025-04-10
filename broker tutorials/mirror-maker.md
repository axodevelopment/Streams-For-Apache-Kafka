oc get secret mirror-maker-user-west -n kafka-tutorial-kraft-west -o yaml > exported-mirror-maker-user-west.yaml

oc apply -f exported-mirror-maker-user-west.yaml


--- ca's

oc get secret my-cluster-kraft-cluster-ca-cert -n kafka-tutorial-kraft-west -o yaml > exported-west-cluster-ca-cert.yaml