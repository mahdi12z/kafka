# Kafka Deployment on Kubernetes
This repository provides step-by-step instructions for deploying Apache Kafka on a Kubernetes cluster. The deployment includes Zookeeper, Kafka Broker, and a Kafka UI (Kafdrop) for easy management.

Prerequisites
Ensure you have the following installed:

Kubernetes Cluster (Minikube, MicroK8s, or a cloud-based cluster)
kubectl (Kubernetes CLI)
Docker (for running Kafdrop)
kafkacat (for testing Kafka topics)

# 1. Defining the Kafka Namespace
First, create a Kubernetes namespace to deploy Kafka resources.

Create a namespace file (00-namespace.yaml):
```sh
apiVersion: v1
kind: Namespace
metadata:
  name: "kafka"
  labels:
    name: "kafka"
...

kubectl apply -f 00-namespace.yaml

# 2. Deploying Zookeeper
Kafka requires Zookeeper for managing broker metadata. We deploy a single Zookeeper instance.

Create a deployment file (01-zookeeper.yaml):
yaml
Copy
apiVersion: v1
kind: Service
metadata:
  labels:
    app: zookeeper-service
  name: zookeeper-service
  namespace: kafka
spec:
  type: NodePort
  ports:
    - name: zookeeper-port
      port: 2181
      nodePort: 30181
      targetPort: 2181
  selector:
    app: zookeeper
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: zookeeper
  name: zookeeper
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - image: wurstmeister/zookeeper
          imagePullPolicy: IfNotPresent
          name: zookeeper
          ports:
            - containerPort: 2181
Apply the deployment:
sh
Copy
kubectl apply -f 01-zookeeper.yaml
Verify Zookeeper service:
sh
Copy
kubectl get services -n kafka
# 3. Deploying a Kafka Broker
Now, deploy the Kafka broker and connect it to Zookeeper.

Create a deployment file (02-kafka.yaml):
Replace <ZOOKEEPER-INTERNAL-IP> with the ClusterIP of the Zookeeper service obtained in the previous step.

yaml
Copy
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kafka-broker
  name: kafka-service
  namespace: kafka
spec:
  ports:
    - port: 9092
  selector:
    app: kafka-broker
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kafka-broker
  name: kafka-broker
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-broker
  template:
    metadata:
      labels:
        app: kafka-broker
    spec:
      hostname: kafka-broker
      containers:
        - env:
            - name: KAFKA_BROKER_ID
              value: "1"
            - name: KAFKA_ZOOKEEPER_CONNECT
              value: "<ZOOKEEPER-INTERNAL-IP>:2181"
            - name: KAFKA_LISTENERS
              value: PLAINTEXT://:9092
            - name: KAFKA_ADVERTISED_LISTENERS
              value: PLAINTEXT://kafka-broker:9092
          image: wurstmeister/kafka
          imagePullPolicy: IfNotPresent
          name: kafka-broker
          ports:
            - containerPort: 9092
Apply the deployment:
sh
Copy
kubectl apply -f 02-kafka.yaml
Kafka Broker may take a minute to start. Check its status with:

sh
Copy
kubectl get pods -n kafka
# 4. Configuring Hostname for Kafka
To ensure Zookeeper and Kafka can communicate properly, add the following entry to your local /etc/hosts file:

Copy
127.0.0.1   kafka-broker
# 5. Testing Kafka Topics
To verify that Kafka is working correctly, follow these steps:

Expose Kafka Broker Port:
sh
Copy
kubectl port-forward kafka-broker-<POD_ID> 9092 -n kafka
Replace <POD_ID> with the actual Kafka broker pod ID (found using kubectl get pods -n kafka).

Produce a test message:
sh
Copy
echo "hello world!" | kafkacat -P -b localhost:9092 -t test
Consume the test message:
sh
Copy
kafkacat -C -b localhost:9092 -t test
If everything is set up correctly, the output will be:

shell
Copy
hello world!
% Reached end of topic test [0] at offset 1
# 6. Kafka UI (Kafdrop)
To monitor and manage Kafka topics visually, deploy Kafdrop using Docker.

Run Kafdrop using Docker:
sh
Copy
docker run -d -p 9000:9000 \
    --add-host kafka-broker:<KAFKA-INTERNAL-IP> \
    -e KAFKA_BROKERCONNECT=<KAFKA-INTERNAL-IP>:9092 \
    obsidiandynamics/kafdrop
Replace <KAFKA-INTERNAL-IP> with the ClusterIP of the Kafka broker.

Access Kafka UI:
Open a browser and navigate to:
 http://localhost:9000
Here, you can view topics, partitions, and messages.
