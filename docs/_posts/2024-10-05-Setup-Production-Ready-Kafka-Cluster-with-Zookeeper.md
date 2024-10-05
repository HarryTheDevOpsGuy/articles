---
title: Setup Production-ready Kafka Cluster with Zookeeper
key: 20241005
tags: howtos
author: Harry - The DevOps Guy
#show_author_profile: true
aside:
  toc: true
---

Configuring a production-ready Apache Kafka cluster requires careful planning and setup to ensure reliability, scalability, and fault tolerance. Here is a step-by-step guide to configure a production-ready Kafka cluster:

### Step 1: **Planning the Kafka Cluster**
Before installing Kafka, you need to plan the cluster size, resources, and other key parameters.

1. **Number of Brokers**: Start with at least 3 brokers to ensure high availability and fault tolerance.
2. **Zookeeper**: Kafka relies on Zookeeper for managing brokers and metadata. Set up a Zookeeper cluster with an odd number of nodes (at least 3 nodes).
3. **Storage Requirements**: Estimate how much data will be retained in Kafka, and provision adequate storage on each broker. Use high-throughput SSDs for optimal performance.
4. **Network and Bandwidth**: Kafka is a network-intensive application. Ensure your network can handle the expected throughput, preferably using 10Gbps or higher.
5. **Replication Factor**: Decide the replication factor for topics (at least 3 for production).
6. **Topic Partitions**: Plan the number of partitions per topic based on the expected data throughput and parallelism.

### Step 2: **Set Up Zookeeper Cluster**
Kafka requires Zookeeper for cluster management. Here’s how to set up a Zookeeper cluster:

1. **Download Zookeeper**:  
   Download Zookeeper from [Apache Zookeeper](https://zookeeper.apache.org/releases.html) and extract the tarball.

   ```bash
   wget https://apache.org/dyn/closer.cgi/zookeeper/zookeeper-3.x.x/apache-zookeeper-3.x.x-bin.tar.gz
   tar -xvf apache-zookeeper-3.x.x-bin.tar.gz
   ```

2. **Configure Zookeeper**:
   Create a Zookeeper configuration file (`zoo.cfg`) under the `conf` directory.

   ```bash
   vi /path/to/zookeeper/conf/zoo.cfg
   ```

   Example configuration:
   ```properties
   tickTime=2000
   dataDir=/var/lib/zookeeper
   clientPort=2181
   initLimit=10
   syncLimit=5
   server.1=zk1:2888:3888
   server.2=zk2:2888:3888
   server.3=zk3:2888:3888
   ```

3. **Start Zookeeper**:
   Run Zookeeper on each server.

   ```bash
   ./bin/zkServer.sh start
   ```

   Ensure Zookeeper is running on all nodes, and the quorum is formed.

### Step 3: **Set Up Kafka Brokers**
Now that Zookeeper is running, set up the Kafka brokers.

1. **Download Kafka**:
   Download Kafka from [Apache Kafka](https://kafka.apache.org/downloads).

   ```bash
   wget https://downloads.apache.org/kafka/3.x.x/kafka_2.13-3.x.x.tgz
   tar -xvf kafka_2.13-3.x.x.tgz
   ```

2. **Configure Kafka Broker**:
   Update the `server.properties` file for each broker.

   ```bash
   vi /path/to/kafka/config/server.properties
   ```

   Key configuration options:
   ```properties
   broker.id=1  # Unique broker ID for each broker (e.g., 1, 2, 3)
   listeners=PLAINTEXT://broker1:9092
   log.dirs=/var/lib/kafka/logs  # Specify the log directory
   zookeeper.connect=zk1:2181,zk2:2181,zk3:2181  # Zookeeper connection string
   num.partitions=3  # Number of default partitions
   log.retention.hours=168  # Retain logs for 7 days
   log.segment.bytes=1073741824  # Size of each log segment (1GB)
   log.retention.bytes=10000000000  # Maximum log size before deletion
   num.network.threads=3  # Adjust based on your server's CPU
   num.io.threads=8  # Adjust based on your server's CPU
   socket.send.buffer.bytes=102400
   socket.receive.buffer.bytes=102400
   socket.request.max.bytes=104857600  # 100 MB max message size
   offsets.topic.replication.factor=3  # Ensure high availability for offset topic
   transaction.state.log.replication.factor=3  # Replicate transaction logs
   transaction.state.log.min.isr=2  # Ensure minimum ISR for transaction logs
   ```

   **Important settings**:
   - `broker.id`: Each Kafka broker should have a unique ID.
   - `log.retention.hours`: How long Kafka retains logs.
   - `log.segment.bytes`: The maximum size of a segment before Kafka rolls it.
   - `zookeeper.connect`: Zookeeper nodes the broker will connect to.
   - `replication.factor`: Set to 3 for high availability.

3. **Start Kafka Brokers**:
   Start the Kafka broker on each server:

   ```bash
   ./bin/kafka-server-start.sh /path/to/kafka/config/server.properties
   ```

### Step 4: **Enable Replication and Fault Tolerance**
1. **Replication**: Set the replication factor to at least 3 in your topics to ensure high availability and fault tolerance.
2. **ISR (In-Sync Replicas)**: Configure the minimum number of in-sync replicas (`min.insync.replicas=2`). This ensures that at least two brokers have a copy of the data before an acknowledgment is sent to the producer.

### Step 5: **Configure Security (Optional but Recommended)**
For production environments, it’s highly recommended to configure security:

1. **TLS Encryption**:
   Enable TLS for communication between Kafka clients and brokers.

   Add the following to `server.properties`:
   ```properties
   listeners=SSL://broker1:9093
   ssl.keystore.location=/path/to/keystore.jks
   ssl.keystore.password=keystore-password
   ssl.key.password=key-password
   ssl.truststore.location=/path/to/truststore.jks
   ssl.truststore.password=truststore-password
   ```

2. **Authentication and Authorization**:
   - **SASL Authentication**: Use SASL to authenticate clients and brokers.
   - **ACLs**: Set up Access Control Lists (ACLs) to limit access to topics and consumer groups.

### Step 6: **Configure Monitoring and Metrics**
Kafka provides JMX metrics, which you can monitor using tools like Prometheus, Grafana, or Datadog.

1. **Enable JMX**:
   Set the following in `server.properties` to expose JMX metrics:

   ```properties
   JMX_PORT=9999
   ```

2. **Install Prometheus JMX Exporter** (optional):
   Use the [JMX exporter](https://github.com/prometheus/jmx_exporter) to scrape metrics from Kafka and expose them to Prometheus.

3. **Setup Monitoring Dashboards**:
   Create dashboards in Grafana for monitoring key Kafka metrics such as:
   - Broker throughput (bytes in/out)
   - Partition replica lag
   - Under-replicated partitions
   - Consumer group lag
   - Broker CPU, memory, and disk usage

### Step 7: **Configure Log Retention and Compaction**
Kafka topics can be configured for either log retention or log compaction, depending on the use case.

1. **Log Retention**:
   Retain data for a specific period (e.g., 7 days) using the following settings in `server.properties`:

   ```properties
   log.retention.hours=168  # Retain logs for 7 days
   log.retention.bytes=10737418240  # Retain logs until the total size reaches 10GB
   ```

2. **Log Compaction**:
   Enable log compaction for topics that need to retain the latest value for each key (e.g., in the case of user profile updates).

   ```properties
   log.cleanup.policy=compact
   ```

### Step 8: **Optimize for Performance**
For a production environment, optimize Kafka’s performance by tuning the following:

1. **Disk Throughput**: Use SSDs and configure RAID for high IOPS.
2. **Memory**: Increase Kafka’s heap size in the Kafka startup script (`KAFKA_HEAP_OPTS`).

   ```bash
   export KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
   ```

3. **Network Tuning**: Ensure enough network buffers are available, and consider tuning Linux kernel settings like `net.core.somaxconn` and `net.ipv4.tcp_tw_recycle`.

4. **I/O Threads**: Configure `num.io.threads` based on the number of available CPU cores.

### Step 9: **Test Failover and Recovery**
Simulate broker failures by shutting down brokers and observing how Kafka automatically rebalances partitions and resumes processing. This will help you ensure your replication and recovery strategies are working as expected.

### Step 10: **Set Up Backups (Optional)**
While Kafka is fault-tolerant, it's good practice to back up important topic data. You can back up Kafka logs or use a tool like `MirrorMaker` to replicate data across Kafka clusters.

---

By following these steps, you’ll have a production-ready Kafka cluster with high availability, security, monitoring, and optimized performance.