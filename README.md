
# CYT180 – Week 3 Lab  
## Intro to Hadoop & HDFS for Cybersecurity Data Analytics (Google Colab)

This lab introduces Hadoop in **pseudo‑distributed mode** on Google Colab and walks you through loading and analyzing **cybersecurity datasets** using HDFS + MapReduce/Streaming.  

**Concepts come from Week 3 slides:**
- HDFS blocks, replication, NameNode/DataNode roles, read/write paths, and data locality

Pseudo‑distributed Hadoop normally requires Java configuration and passwordless SSH as noted in the official Hadoop single‑node setup. (https://barrelsofdata.com/apache-hadoop-pseudo-distributed-mode)

---

## Learning Objectives
- Install Hadoop in pseudo‑distributed mode on Google Colab.  
- Create and interact with HDFS.  
- Load cybersecurity logs (Zeek, Windows EVTX → CSV, small NetFlow‑like).  
- Run HDFS commands and Hadoop MapReduce/Streaming jobs.  
- Explain NameNode (metadata-only) and DataNode (data blocks) responsibilities. [1](https://docs.zeek.org/en/master/logs/index.html)

---

## Step 1: Getting Started (Google Colab)

Open a **new Colab notebook** and execute each cell below.

---

## 0) Install Java + Hadoop

```bash
!apt-get -qq update
!apt-get -qq install -y openjdk-8-jdk-headless > /dev/null

# Download Hadoop 3.3.1
!wget -q https://archive.apache.org/dist/hadoop/core/hadoop-3.3.1/hadoop-3.3.1.tar.gz
!tar -xzf hadoop-3.3.1.tar.gz
