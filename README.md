# strimzi-mirroring

# Setup

## Login to clusters

Login into hub cluster
```bash
KUBECONFIG=~/.kube/config_hub; oc login --token=<token> --server=<url_api>
```

Login into live cluster
```bash
KUBECONFIG=~/.kube/config_live; oc login --token=<token> --server=<url_api>
```

Login into dr cluster
```bash
KUBECONFIG=~/.kube/config_dr; oc login --token=<token> --server=<url_api>
```

## Add role binding for ArgoCD SA
```bash
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n kafka-live
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n kafka-dr
```

## Install ArgoCD ApplicationSet

Apply ApplicationSet (App of Apps):
```bash
KUBECONFIG=~/.kube/config_hub oc apply -f acm/application-set.yaml -n openshift-gitops
```

ArgoCD CLI login:
```bash
export LIVE_ARGOCD_PWD=$(KUBECONFIG=~/.kube/config_live oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
export LIVE_ARGOCD_HOST=$(KUBECONFIG=~/.kube/config_live oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.status.ingress[0].host}')

argocd login --name live --username admin --password ${LIVE_ARGOCD_PWD} ${LIVE_ARGOCD_HOST} --insecure

export DR_ARGOCD_PWD=$(KUBECONFIG=~/.kube/config_dr oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
export DR_ARGOCD_HOST=$(KUBECONFIG=~/.kube/config_dr oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.status.ingress[0].host}')

argocd login --name dr --username admin --password ${DR_ARGOCD_PWD} ${DR_ARGOCD_HOST} --insecure
```

Sync apps:
```bash
argocd --server-name live app sync streams-app-cluster-live
argocd --server-name dr app sync streams-app-cluster-dr
```

Wait for all kafka clusters to be ready
```bash
KUBECONFIG=~/.kube/config_live oc wait kafka/streams-cluster -n kafka-live --for=condition=Ready --timeout=500s
KUBECONFIG=~/.kube/config_dr oc wait kafka/streams-cluster -n kafka-dr --for=condition=Ready --timeout=500s
```

## Create MM2 secrets

Create secret for live mirroring user on dr
```bash
export LIVE_MIRRORING_USER_PASSWORD=$(KUBECONFIG=~/.kube/config_live oc get secret mirroring-user -n kafka-live -o jsonpath='{.data.password}' | base64 -d)

KUBECONFIG=~/.kube/config_dr oc create secret generic live-mirroring-user -n kafka-dr --from-literal=password=$LIVE_MIRRORING_USER_PASSWORD
```

Create secret for dr mirroring user on live
```bash
export DR_MIRRORING_USER_PASSWORD=$(KUBECONFIG=~/.kube/config_dr oc get secret mirroring-user -n kafka-dr -o jsonpath='{.data.password}' | base64 -d)

KUBECONFIG=~/.kube/config_live oc create secret generic dr-mirroring-user -n kafka-live --from-literal=password=$DR_MIRRORING_USER_PASSWORD
```

Wait for mm2 to be ready
```bash
KUBECONFIG=~/.kube/config_dr oc wait kafkamirrormaker2/streams-mm2 -n kafka-dr --for=condition=Ready --timeout=500s
```

# Test

## 1 - Produce and consume messages on live cluster

Produce test messages to topic `my-topic`
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --producer.config=/tmp/producer.properties"
```

Input messages
```
>test msg 1 live
>test msg 2 live
>test msg 3 live
```

Consume test messages from topic `my-topic` and create consumer group `app-group`
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --group app-group --from-beginning --consumer.config=/tmp/consumer.properties"
```

Output
```
test msg 1 live
test msg 2 live
test msg 3 live
```

Produce test messages to topic `my-topic`
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --producer.config=/tmp/producer.properties"
```

Input
```
>test msg 4 live
>test msg 5 live
>test msg 6 live
```

## 2 - Switch to dr cluster

### 2.1 - Stop mirroring from live -> dr

Change git branch to `stop_mm2_live-to-dr`:
```bash
yq '.spec.template.spec.sources[0].targetRevision = "stop_mm2_live-to-dr"' acm/application-set.yaml | \
    KUBECONFIG=~/.kube/config_hub oc replace -f -
```

Sync app:
```bash
argocd --server-name dr app sync streams-app-cluster-dr
```

### 2.2 - Consume and produce messages on dr cluster

Consume test messages from topic `my-topic` and create consumer group `app-group`
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --group app-group --from-beginning --consumer.config=/tmp/consumer.properties"
```

Output
```
test msg 4 live
test msg 5 live
test msg 6 live
```

> [!IMPORTANT]  
> If the checkpoint connector has not yet performed the sync the consumer offsets, the client (consumer) may consume already consumed messages!

Produce test messages to topic `my-topic`
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --producer.config=/tmp/producer.properties"
```

Input messages
```
>test msg 7 dr
>test msg 8 dr
>test msg 9 dr
```

Consume test messages from topic `my-topic` and create consumer group `app-group`
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --group app-group --from-beginning --consumer.config=/tmp/consumer.properties"
```

Output
```
test msg 7 dr
test msg 8 dr
test msg 9 dr
```

Produce test messages to topic `my-topic`
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --producer.config=/tmp/producer.properties"
```

Input messages
```
>test msg 10 dr
>test msg 11 dr
>test msg 12 dr
```

### 2.3 - Start mirroring from dr to live cluster

Delete consumer group:
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-topics -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-consumer-groups.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --delete --group app-group --command-config=/tmp/client.properties"
```

Delete topic:
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-topics -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-topics.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --delete --topic my-topic --command-config=/tmp/client.properties"
```

Change git branch to `start_mm2_dr-to-live`:
```bash
yq '.spec.template.spec.sources[0].targetRevision = "start_mm2_dr-to-live"' acm/application-set.yaml | \
    KUBECONFIG=~/.kube/config_hub oc replace -f -
```

Sync app:
```bash
argocd --server-name live app sync streams-app-cluster-live
argocd --server-name dr app sync streams-app-cluster-dr
```

Wait for mm2 to be ready
```bash
KUBECONFIG=~/.kube/config_live oc wait kafkamirrormaker2/streams-mm2 -n kafka-live --for=condition=Ready --timeout=500s
```

## 3 - Back to live cluster

### 3.1 - Stop mirroring from dr -> live
TODO

### 3.2 - Consume and produce messages on live cluster

Consume test messages from topic `my-topic` and create consumer group `app-group`
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --group app-group --from-beginning --consumer.config=/tmp/consumer.properties"
```

Output
```
test msg 10 dr
test msg 11 dr
test msg 12 dr
```

> [!IMPORTANT]  
> If the checkpoint connector has not yet performed the sync the consumer offsets, the client (consumer) may consume already consumed messages!

### 3.3 - Start mirroring from live to dr cluster

TODO:
1. Cleanup topics and cg on dr

```bash
git checkout main
KUBECONFIG=~/.kube/config_hub oc replace -f acm/application-set.yaml -n openshift-gitops
```

Wait for mm2 to be ready
```bash
KUBECONFIG=~/.kube/config_dr oc wait kafkamirrormaker2/streams-mm2 -n kafka-dr --for=condition=Ready --timeout=500s
```

# Debug

## List topics

Live cluster:
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-topics -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-topics.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --list --command-config=/tmp/client.properties"
```

Dr cluster:
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-topics -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-topics.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --list --command-config=/tmp/client.properties"
```

## Describe consumer groups

Live cluster:
```bash
KUBECONFIG=~/.kube/config_live oc run kafka-topics -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$LIVE_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-consumer-groups.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --describe --all-groups --command-config=/tmp/client.properties"
```

## Dump MM2 internal topics:

mirrormaker cluster configs:
```bash
KUBECONFIG=~/.kube/config_dr oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=mirroring-user password=$DR_MIRRORING_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic mirrormaker2-cluster-configs --from-beginning --consumer.config=/tmp/consumer.properties"
```

## Log mirror maker

Sync consumer group offset:
```
INFO [cluster-source->cluster-target.MirrorCheckpointConnector|task-0] sync idle consumer group offset from source to target took 6 ms
```

# Cleanup all (delete all data also)
```bash
KUBECONFIG=~/.kube/config_hub oc delete ApplicationSet/streams-app-set -n openshift-gitops

KUBECONFIG=~/.kube/config_live oc delete pvc/data-0-streams-cluster-kafka-0 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete pvc/data-0-streams-cluster-kafka-1 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete pvc/data-0-streams-cluster-kafka-2 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete pvc/data-streams-cluster-zookeeper-0 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete pvc/data-streams-cluster-zookeeper-1 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete pvc/data-streams-cluster-zookeeper-2 -n kafka-live
KUBECONFIG=~/.kube/config_live oc delete secret/dr-mirroring-user -n kafka-live

KUBECONFIG=~/.kube/config_dr oc delete pvc/data-0-streams-cluster-kafka-0 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete pvc/data-0-streams-cluster-kafka-1 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete pvc/data-0-streams-cluster-kafka-2 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete pvc/data-streams-cluster-zookeeper-0 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete pvc/data-streams-cluster-zookeeper-1 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete pvc/data-streams-cluster-zookeeper-2 -n kafka-dr
KUBECONFIG=~/.kube/config_dr oc delete secret/live-mirroring-user -n kafka-dr
```