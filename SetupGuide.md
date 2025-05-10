# Data Engineering Home Project Setup Guide

This guide outlines how to set up a complete, hands-on data engineering environment using virtual machines. 
It includes instructions for installing and configuring tools like PostgreSQL, Apache Kafka, Apache Spark, Airflow, and Superset. 
It also includes solutions to various troubleshooting issues encountered during setup.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [VM Setup](#vm-setup)
- [Networking](#networking)
- [PostgreSQL Setup](#postgresql-setup)
- [Kafka Setup](#kafka-setup)
- [Kafka Service Integration](#kafka-service-integration)
- [Testing Kafka Topics](#testing-kafka-topics)
- [Troubleshooting Summary](#troubleshooting-summary)

---

## Overview
This project emulates a simplified data pipeline project.
Using a Windows 10 client and Hyper-V with four virtual machines (VMs) running AlmaLinux 9, each responsible for a different function:

- **Database VM**: PostgreSQL and MinIO
- **Processing VM**: Apache Spark and Airflow
- **Streaming VM**: Apache Kafka
- **Visualization VM**: Superset and Metabase

### Why not Windows for the VMs?

I chose Linux  for the following reasons: 
1. Linux could have better resource efficiency in a VM (even on a windows host).
2. All the tools (Posgresql, Spark, Airflow, Superset) will probably run better on a Linux OS.
3. I wanted to delve into Linux CLI automation and possibly set up some cron jobs.
4. I didn't want to have any software licensing concerns.
5. I wanted to mirror some real-world data engineering pipelines. As best I could.

### Why use these tools?

1. PostgreSQL & MinIO – Used for structured & object storage in production.
2. Apache Spark & Airflow – Common in large-scale ETL & data processing.
3. Apache Superset – A lightweight, open-source alternative to Tableau.
4. Linux-based Infrastructure – Standard for cloud & enterprise environments. 

## Architecture
```
+--------------+     +-------------+     +---------------+     +----------------+
| Visualization|<--->| Processing  |<--->|   Streaming   |<--->|    Database    |
|    (Superset)|     |  (Spark)    |     |   (Kafka)     |     |  (PostgreSQL)  |
+--------------+     +-------------+     +---------------+     +----------------+
```

## VM Setup
1. Create 4 VMs in Hyper-V all with AlmaLinux as the OSß:
   - VM1: `database` (2 vCPU, 4 GB RAM)
   - VM2: `processing` (4 vCPU, 8 GB RAM)
   - VM3: `streaming` (2 vCPU, 4 GB RAM)
   - VM4: `visualization` (2 vCPU, 4 GB RAM)

2. Update hostnames on each VM accordingly:
```
bash

sudo hostnamectl set-hostname database  # Repeat for each VM
```

3. Ensure all VMs are on the same internal network and can `ping` each other by hostname.
   a. Create an internal virtual swich
      1. In Hyper-V Manager
      2. Click Virtual Switch Manager
      3. Select internal and click create virtual switch
      4. Name it something compelling
      5. Click Apply->Ok

   b. Assign the virtual switch to each vm
      1. In each VM
      2. Go to settings
      3. Network Adapter
      4. Select the name of the virtual switch created

   c. Assign static IPs to each VM
      1. Edit the network config on each VM: 
      ```
      bash

      nmcli con show
      ```
      2. Look for an interface name like eno1
      3. Set a static IP address(replace eth0 with your interface name):
      ```
      bash

      sudo nmcli con mod eth0 ipv4.addresses 10.0.0.100/24
      sudo ncmli con mod eth0 ipv4.gateway 10.0.0.1
      sudo nmcli con mod eth0 ipv4.dns 8.8.8.8
      sudo nmcli con mod eth0 ipv4.method manual
      ```
      4. Apply changes
      ```
      bash

      sudo nmcli con up eth0
      ```
      5. Verify the new IP
      ```
      bash

      ip a
      ```

4. Fix DNS resolution:
```
bash

sudo vi /etc/resolv.conf
# Add:
nameserver 8.8.8.8
```

## PostgreSQL Setup
1. Install PostgreSQL 16 on the `database` VM.
```bash
sudo dnf install -y postgresql16-server postgresql16
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```

2. Configure PostgreSQL to accept connections:
- Edit `postgresql.conf` and `pg_hba.conf` for listen addresses and authentication.

3. Install PostgreSQL client tools on all other VMs:
```bash
sudo dnf install -y postgresql
```

## Kafka Setup (Streaming VM)
1. Download and extract Kafka:
```bash
wget https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
mkdir ~/kafka && tar -xzf kafka_2.13-4.0.0.tgz -C ~/kafka --strip-components 1
```

2. Generate and format storage UUID:
```bash
~/kafka/bin/kafka-storage.sh random-uuid
sudo ~/kafka/bin/kafka-storage.sh format -t <UUID> -c ~/kafka/config/server.properties --standalone
```

3. Start Kafka:
```bash
sudo ~/kafka/bin/kafka-server-start.sh ~/kafka/config/server.properties
```

## Kafka Service Integration
Ensure port `9092` is open and accessible from the `processing` VM:
- If firewall is enabled, open the port:
```bash
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --reload
```

- Test from `processing`:
```bash
nc -zv streaming 9092
```

## Testing Kafka Topics
1. On the Streaming VM, create a topic:
```bash
~/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

2. Start a producer:
```bash
~/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic
```

3. Start a consumer on the Processing VM:
   - Requires Kafka client tools installed or copied over.
   - Test connection and consumption from topic:
```bash
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server streaming:9092 --topic test-topic --from-beginning
```

## Troubleshooting Summary
- **DNS not resolving**: Manually edit `/etc/resolv.conf` and disable IPv6 if needed.
- **404 when downloading Kafka**: Use [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads) to get correct mirror.
- **Kafka storage formatting error**: Must generate UUID and format with `kafka-storage.sh`.
- **Kafka start error**: Ensure the `meta.properties` exists and logs directory is writable.
- **Connection refused from Processing to Streaming**: Open port 9092 and confirm hostname resolves.

---

## Next Steps
- Set up Spark and Airflow on the Processing VM.
- Connect Airflow DAGs to read from Kafka and write to PostgreSQL.
- Visualize the results using Superset on the Visualization VM.

---

This guide forms a foundational hands-on data engineering environment that emulates real-world workflows on a local machine using open-source tools.
