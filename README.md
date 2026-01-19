
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

### 1. System prep — Java & Hadoop
Open a **new Colab notebook** and execute each cell below.

```bash
# install java
!apt-get -qq update
!apt-get -qq install -y openjdk-8-jdk-headless > /dev/null

# Download Hadoop 3.3.1
!wget -q https://archive.apache.org/dist/hadoop/core/hadoop-3.3.1/hadoop-3.3.1.tar.gz

# Extract the zip file
!tar -xzf hadoop-3.3.1.tar.gz
```
### Step 2 — Set environment variables

Environment variables are key–value settings that many programs read at startup to find dependencies (like Java) and their own install paths (like Hadoop). In Colab, we set them for the current Python process and shell so that Hadoop can find Java and its own binaries/config.
You will run the following commands one by one.
```
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-8-openjdk-amd64"
os.environ["HADOOP_HOME"] = "/content/hadoop-3.3.1"
os.environ["HADOOP_CONF_DIR"] = "/content/hadoop-3.3.1/etc/hadoop"
os.environ["PATH"] += ":/content/hadoop-3.3.1/bin:/content/hadoop-3.3.1/sbin"
```
Below I explain what each of this enviroment variable is about.
- **1. JAVA_HOME**
  - Hadoop daemons (NameNode/DataNode) are Java processes.
  - If `JAVA_HOME` is not set correctly, Hadoop scripts can’t launch the JVM, and you’ll see errors like “JAVA_HOME is not set and could not be found.”
- **HADOOP_HOME**
  - This is the top folder where Hadoop is unpacked.
  -  Hadoop’s scripts use this to locate binaries (bin/) and configuration (etc/hadoop/).
  -  In Colab: we unpacked into /content/hadoop-3.3.1:
- **HADOOP_CONF_DIR**
  - It points to the directory containing Hadoop’s XML configs (core-site.xml, hdfs-site.xml, etc.).
  - When you run Hadoop commands (e.g., hdfs dfs -ls /), Hadoop reads these files to know the default filesystem URI (fs.defaultFS), replication factor, and local data directories. If this isn’t set (or points somewhere empty), commands may try to use default/standalone mode and won’t talk to your NameNode.
- **PATH**
  -  This adds the Hadoop binaries (hdfs, hadoop, yarn) and sbin (daemon helpers). So that you can run hdfs, hadoop jar, etc., without typing the full paths.
