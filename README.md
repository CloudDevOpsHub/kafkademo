# kafkademo
# Kafka on Kubernetes using Strimzi Operator (KRaft Mode)

This guide demonstrates how to deploy Apache Kafka on Kubernetes using the Strimzi Operator with KRaft mode (ZooKeeper-less architecture).

## Prerequisites

* Kubernetes Cluster
* kubectl configured
* StorageClass available in the cluster

---

# Step 1: Create Namespace

```bash
kubectl create namespace kafka
```

---

# Step 2: Install Strimzi Operator

```bash
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Verify the operator is running:

```bash
kubectl get pods -n kafka
```

Expected output:

```bash
strimzi-cluster-operator-xxxxx   1/1 Running
```

---

# Step 3: Deploy Kafka Cluster

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: true
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 4.2.0
    metadataVersion: 4.2-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

---

# Step 4: Verify Kafka Deployment

Watch the pods:

```bash
kubectl get pods -n kafka -w
```

Check Kafka status:

```bash
kubectl get kafka -n kafka
```

Expected:

```bash
NAME         READY
my-cluster   True
```

---

# Step 5: Create Kafka Topic

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
EOF
```

Verify topic:

```bash
kubectl get kafkatopic -n kafka
```

---

# Step 6: Start Kafka Producer

```bash
kubectl -n kafka run kafka-producer \
  -ti --image=quay.io/strimzi/kafka:latest-kafka-4.2.0 \
  --rm=true --restart=Never \
  -- bin/kafka-console-producer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic
```

Send sample messages:

```text
Hello Kafka
Kafka on Kubernetes
Strimzi Demo
```

---

# Step 7: Start Kafka Consumer

Open another terminal and run:

```bash
kubectl -n kafka run kafka-consumer \
  -ti --image=quay.io/strimzi/kafka:latest-kafka-4.2.0 \
  --rm=true --restart=Never \
  -- bin/kafka-console-consumer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic \
  --from-beginning
```

You should see all producer messages displayed in real time.

---

# Step 8: List Kafka Topics

```bash
kubectl -n kafka run kafka-client \
  -ti --image=quay.io/strimzi/kafka:latest-kafka-4.2.0 \
  --rm=true --restart=Never \
  -- bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --list
```

---

# Step 9: Describe Kafka Topic

```bash
kubectl -n kafka run kafka-client \
  -ti --image=quay.io/strimzi/kafka:latest-kafka-4.2.0 \
  --rm=true --restart=Never \
  -- bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe \
  --topic my-topic
```

---

# Cleanup

Delete Kafka Cluster:

```bash
kubectl delete kafka my-cluster -n kafka
```

Delete Namespace:

```bash
kubectl delete namespace kafka
```

---

## Architecture

* Kubernetes
* Strimzi Operator 1.0.0
* Apache Kafka 4.2.0
* KRaft Mode (No ZooKeeper)
* KafkaNodePool
* Internal Listener
* 3 Broker Cluster
