# strimzi-mirroring


Login into live cluster
```bash
KUBECONFIG=~/.kube/config_live; oc login --token=<token> --server=<url_api>
```

Login into dr cluster
```bash
KUBECONFIG=~/.kube/config_dr; oc login --token=<token> --server=<url_api>
```

## Add role
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n kafka-live
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n kafka-dr

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

# Test

## Produce and consume messages on live cluster

Export k8s context
```bash
export KUBECONFIG=~/.kube/config_live
```

Produce test messages to topic `my-topic`
```bash
oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
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
oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
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

## Produce messages on live cluster and consume messages on dr cluster

Produce test messages to topic `my-topic`
```bash
oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
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

## Produce and consume messages on dr cluster

Stop mirroring
```
yq e -i '.spec.template.spec.sources[0].targetRevision="switch-to-dr"' acm/application-set.yaml
```

Export k8s context
```bash
export KUBECONFIG=~/.kube/config_dr
```

Produce test messages to topic `my-topic`
```bash
oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-dr -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
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
oc run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka-live -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
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