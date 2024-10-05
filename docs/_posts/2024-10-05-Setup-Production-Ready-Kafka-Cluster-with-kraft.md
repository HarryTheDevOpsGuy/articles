---
title: Setup Production-ready Kafka Cluster with kraft
key: 20241005
tags: howtos
author: Harry - The DevOps Guy
#show_author_profile: true
aside:
  toc: true
---

Setting up a production-ready Kafka cluster using Kafka with **KRaft (Kafka Raft Metadata)**, which eliminates the need for Zookeeper, is a more modern approach. Here's a step-by-step guide for configuring a production-ready Kafka cluster using Kafka + KRaft:

### Step 1: **Planning the Kafka Cluster**

1. **Cluster Size**: Decide on the number of brokers (at least 3 for high availability).
2. **Storage Requirements**: Plan storage based on the amount of data and retention policy. Use SSDs for high I/O performance.
3. **Partitions**: Choose the number of partitions carefully based on the workload and throughput needs.
4. **Network**: Use a high-speed network (e.g., 10Gbps) to handle Kafka’s data flow efficiently.
5. **Replication Factor**: Set the replication factor to at least 3 for durability.

### Step 2: **Download and Install Kafka**
Download the latest Kafka release that supports KRaft mode from the [Apache Kafka website](https://kafka.apache.org/downloads).

```bash
wget https://downloads.apache.org/kafka/3.x.x/kafka_2.13-3.x.x.tgz
tar -xvf kafka_2.13-3.x.x.tgz
cd kafka_2.13-3.x.x
```

### Step 3: **Initial Cluster Configuration**

1. **Configure KRaft Mode**:
   Kafka's KRaft mode uses its built-in Raft consensus protocol for metadata management.

   Edit the `server.properties` file:

   ```bash
   vi config/kraft/server.properties
   ```

   Key configurations for KRaft:

   ```properties
   process.roles=broker,controller
   node.id=1
   controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
   listeners=PLAINTEXT://broker1:9092,CONTROLLER://broker1:9093
   log.dirs=/var/lib/kafka/data
   num.partitions=3
   offsets.topic.replication.factor=3
   transaction.state.log.replication.factor=3
   transaction.state.log.min.isr=2
   ```

   - **process.roles**: Both broker and controller roles in KRaft.
   - **controller.quorum.voters**: This defines the quorum nodes for metadata management.
   - **node.id**: Unique node ID for each broker.
   - **listeners**: Defines both broker and controller listener ports.

2. **Storage Directory Setup**:
   Create the directories for Kafka logs and data:

   ```bash
   mkdir -p /var/lib/kafka/data
   ```

3. **Initialize the KRaft Metadata Quorum**:
   Kafka’s KRaft mode requires initialization for the metadata quorum. Run the following command on one of the nodes to bootstrap the cluster:

   ```bash
   ./bin/kafka-storage.sh format --config config/kraft/server.properties --cluster-id $(./bin/kafka-storage.sh random-uuid)
   ```

4. **Set the Same Cluster ID on All Brokers**:
   Make sure you use the same `cluster-id` for all the brokers.

### Step 4: **Start the Kafka Brokers**

Once you have initialized the storage and metadata for the cluster, start each Kafka broker:

```bash
./bin/kafka-server-start.sh config/kraft/server.properties
```

Do this on all the broker nodes in your cluster.

### Step 5: **Create Topics and Test**

1. **Create Kafka Topics**:
   Now that your cluster is up, you can create Kafka topics with desired replication and partition settings.

   ```bash
   ./bin/kafka-topics.sh --create --bootstrap-server broker1:9092 --replication-factor 3 --partitions 3 --topic my-topic
   ```

2. **Send and Consume Messages**:
   Test the Kafka setup by producing and consuming messages.

   - Produce messages:
     ```bash
     ./bin/kafka-console-producer.sh --broker-list broker1:9092 --topic my-topic
     ```

   - Consume messages:
     ```bash
     ./bin/kafka-console-consumer.sh --bootstrap-server broker1:9092 --topic my-topic --from-beginning
     ```

### Step 6: **Configure Production-Ready Security**

1. **TLS Encryption**:
   Enable SSL/TLS to secure communication between brokers and clients.

   ```properties
   listeners=SSL://broker1:9092
   ssl.keystore.location=/path/to/keystore.jks
   ssl.keystore.password=keystore-password
   ssl.truststore.location=/path/to/truststore.jks
   ssl.truststore.password=truststore-password
   ```

2. **Authentication with SASL**:
   For secure client-broker communication, configure SASL authentication. Example SASL configuration for broker:

   ```properties
   sasl.enabled.mechanisms=PLAIN
   security.inter.broker.protocol=SASL_SSL
   ```

3. **Authorization with ACLs**:
   Implement ACLs to control access to topics and consumer groups.

   ```bash
   ./bin/kafka-acls.sh --add --allow-principal User:user1 --operation Read --topic my-topic
   ```

### Step 7: **Configure Monitoring and Metrics**

1. **Enable JMX Metrics**:
   Kafka provides JMX for exposing metrics. Set the following environment variable in the startup script:

   ```bash
   export JMX_PORT=9999
   ```

2. **Use Prometheus for Monitoring**:
   Install [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter) for monitoring Kafka brokers and expose metrics to Prometheus.

   Example configuration:

   ```bash
   - javaagent:/path/to/jmx_prometheus_javaagent.jar=7071:/path/to/kafka.yml
   ```

3. **Set Up Grafana Dashboards**:
   Create Grafana dashboards to monitor Kafka metrics such as broker throughput, under-replicated partitions, consumer lag, etc.

### Step 8: **Log Retention and Data Management**

1. **Log Retention**:
   Configure log retention policies based on your storage capacity and data retention needs.

   ```properties
   log.retention.hours=168  # Retain data for 7 days
   log.retention.bytes=10000000000  # Max log size before deletion
   ```

2. **Log Compaction**:
   Enable log compaction for topics where you need to retain only the latest value for each key.

   ```properties
   log.cleanup.policy=compact
   ```

### Step 9: **Backup and Disaster Recovery**

- Consider setting up **MirrorMaker** for cross-cluster replication if you need geo-redundancy or failover across Kafka clusters.

### Step 10: **Optimize Performance**

- **Heap Size**: Adjust the Kafka broker’s heap size (`KAFKA_HEAP_OPTS`) based on your available memory. Start with around 8GB:
  
  ```bash
  export KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
  ```

- **Increase I/O Threads**:
  
  ```properties
  num.network.threads=3
  num.io.threads=8
  ```

### Conclusion
With the above steps, you’ll have a Kafka cluster running with KRaft mode in production. This configuration ensures high availability, scalability, and better resource management, removing the need for Zookeeper.

---

### Areas to Improve:
1. Adding a section on **capacity planning** for the Kafka cluster.
2. More detailed instructions on **security hardening** for production.
3. Including a brief explanation on how to scale the cluster after setup.

