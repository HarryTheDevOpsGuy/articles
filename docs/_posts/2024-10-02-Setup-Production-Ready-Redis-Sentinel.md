---
title: Setup Production-ready Redis Sentinel
key: 20241002
tags: howtos
modify_date: 2017-09-09
author: Harry - The DevOps Guy
show_author_profile: true
aside:
  toc: true
---

Setting up a production-ready 3-node Redis Sentinel cluster on EC2 involves configuring both the Redis servers and the Sentinel nodes across different EC2 instances. Redis Sentinel provides high availability for Redis by monitoring the master and replicas and performing failover when necessary.

### Steps to Set Up a Redis Sentinel Cluster on EC2:

### Prerequisites:
1. **EC2 Instances:** Ensure you have 3 EC2 instances (or more) ready. They should be in different availability zones for high availability.
2. **Security Groups:** Allow Redis (port 6379) and Sentinel (port 26379) traffic between instances.
3. **Private IPs:** Each EC2 instance should have a private IP that will be used for Redis and Sentinel communication.

### Step-by-Step Setup

#### 1. **Install Redis on all 3 EC2 instances**
- SSH into each EC2 instance.
- Install Redis on each node:

   ```bash
   sudo yum update -y  # For Amazon Linux/CentOS
   sudo yum install redis -y
   ```

#### 2. **Configure Redis on each node**
- Modify the Redis configuration file (`/etc/redis/redis.conf`) on each instance.

   - Set the IP address to the instance's private IP:
     ```bash
     bind <private_ip_of_instance>
     ```

   - Enable persistence by setting the `appendonly` option:
     ```bash
     appendonly yes
     ```

   - Set the master node on one of the EC2 instances (for the master node):
     ```bash
     port 6379
     protected-mode yes
     ```

   - For the other two Redis nodes (slave nodes), add:
     ```bash
     replicaof <master_ip> 6379
     ```

   - Set Redis password (optional but recommended for production):
     ```bash
     requirepass <redis_password>
     masterauth <redis_password>
     ```

   - Save and restart Redis:
     ```bash
     sudo systemctl restart redis
     sudo systemctl enable redis
     ```

#### 3. **Configure Redis Sentinel on each node**
- Edit the `sentinel.conf` file for each node (usually located at `/etc/redis/sentinel.conf`).

   - Add the following configuration:
     ```bash
     port 26379
     sentinel monitor mymaster <master_ip> 6379 2
     sentinel auth-pass mymaster <redis_password>  # Use the Redis password if authentication is enabled
     sentinel down-after-milliseconds mymaster 5000
     sentinel failover-timeout mymaster 15000
     sentinel parallel-syncs mymaster 1
     ```

   - Ensure Sentinel listens on the private IP of each EC2 instance:
     ```bash
     bind <private_ip_of_instance>
     ```

   - Start and enable Sentinel:
     ```bash
     sudo systemctl start redis-sentinel
     sudo systemctl enable redis-sentinel
     ```

#### 4. **Verify the Sentinel Configuration**
- On each EC2 instance, check the Sentinel status:
   ```bash
   redis-cli -p 26379
   SENTINEL get-master-addr-by-name mymaster
   ```

- This should return the master node's IP address.

#### 5. **Failover Testing**
- You can test the failover by stopping the master node:
   ```bash
   sudo systemctl stop redis
   ```

- After a few seconds, the Sentinel nodes should elect a new master. You can verify the new master by running the following command on any Sentinel node:
   ```bash
   redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
   ```

#### 6. **Automate with Ansible (Optional)**
For ease of deployment and management in production, you can use Ansible to automate the setup across all EC2 instances.

#### 7. **Set up Monitoring and Alerts**
- **CloudWatch**: Set up AWS CloudWatch to monitor Redis and Sentinel metrics like CPU, memory, and network usage.
- **Prometheus and Grafana**: You can configure Redis Exporter for Prometheus to gather Redis metrics and visualize them in Grafana.

#### 8. **Security Considerations**
- Use **Redis Authentication** (`requirepass` and `masterauth`) to secure Redis.
- Ensure that **Security Groups** only allow traffic from trusted IP addresses (like other EC2 instances or specific networks).
- For additional security, consider **TLS encryption** for Redis.

By following these steps, you'll have a 3-node Redis Sentinel cluster in production, capable of automatic failover and monitoring.