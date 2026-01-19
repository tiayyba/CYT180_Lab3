
# CYT180 – Week 3 Lab  
## Intro to Hadoop & HDFS for Cybersecurity Data Analytics (Google Colab)

This lab introduces Hadoop in **pseudo‑distributed mode** on Google Colab and walks you through loading and analyzing **cybersecurity datasets** using HDFS + MapReduce/Streaming.  This environment allows you to experiment with Hadoop without installing anything locally.

The lab directly reinforces the following concepts from week 3:
- HDFS architecture
- Data locality
- Reads/writes
- MapReduce basics
- Cybersecurity analytics use cases
---

## Learning Objectives
- Install and configure Hadoop in a Linux environment (Colab).
- Create, inspect, and manipulate directories & files in HDFS.
- Understand how HDFS metadata and block storage work in practice.
- Run a simple MapReduce job using security‑relevant data.
---
## Prerequisites

- Google Colab
- Basic Linux CLI skills
- Understanding of logs (HTTP/SSH)
- Intro knowledge of distributed systems
---
### Step 1. System prep — Java & Hadoop
Open a **new Colab notebook** and execute each cell below.

```bash

# Update packages
!apt-get -qq update

# Install Java (Hadoop requires the JVM)
!apt-get -qq install -y openjdk-11-jdk-headless > /dev/null

# Check Java
!java -version
```
Next run the following code to download  Hadoop
```
# Download Hadoop 3.3.1
!wget -q https://archive.apache.org/dist/hadoop/core/hadoop-3.3.1/hadoop-3.3.1.tar.gz

# Extract the zip file
!tar -xzf hadoop-3.3.1.tar.gz

```
### Step 2 — Set environment variables

Environment variables are key–value settings that many programs read at startup to find dependencies (like Java) and their own install paths (like Hadoop). In Colab, we set them for the current Python process and shell so that Hadoop can find Java and its own binaries/config.

Run the following cell to set the environment variables.

```
import os

# 1. Java
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"

# 2. Hadoop home
os.environ["HADOOP_HOME"] = "/content/hadoop-3.3.1"

# 3. Hadoop config directory
os.environ["HADOOP_CONF_DIR"] = f'{os.environ["HADOOP_HOME"]}/etc/hadoop'

# 4. Add Hadoop's bin/ and sbin/ to PATH
os.environ["PATH"] += f':{os.environ["HADOOP_HOME"]}/bin:{os.environ["HADOOP_HOME"]}/sbin'

```
Below I explain what each of this enviroment variable is about.
- **JAVA_HOME** → Needed to launch JVM-based Hadoop daemons.
- **HADOOP_HOME** → Base folder for all Hadoop commands and configs.
- **HADOOP_CONF_DIR** → Tells Hadoop where to find core-site.xml and hdfs-site.xml.
- **PATH** → Lets you run hdfs and hadoop without typing long paths.

### Step 3 — Create Hadoop Configuration Files
Now that `JAVA_HOME`, `HADOOP_HOME`, `HADOOP_CONF_DIR`, and `PATH` are set, the next required task in a  HDFS setup is to create Hadoop configuration files. HDFS will not start unless `core-site.xml` and `hdfs-site.xml` exist.
These two files tell Hadoop:
- where your NameNode lives
- what filesystem to use
- where to store NameNode/DataNode data
- how many replicas to maintain
Run the following commands:
```

%%bash
cd /content/hadoop-3.3.1/etc/hadoop

# --- core-site.xml ---
cat > core-site.xml << 'EOF'
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
EOF

# --- hdfs-site.xml ---
cat > hdfs-site.xml << 'EOF'
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///content/hadoop_tmp/hdfs/namenode</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///content/hadoop_tmp/hdfs/datanode</value>
    </property>
</configuration>
EOF

```
- fs.defaultFS → The NameNode URI
- dfs.replication=1 → Required for single-node mode
- NameNode/DataNode dirs → Where metadata + data blocks are stored

### Step 4 — Format HDFS and Start Daemons
Run the following commands one by one. You may ignore warnings about “Unable to load native-hadoop library.” These are normal in Colab.
```
!hdfs namenode -format

# Start NameNode + DataNode
!hdfs --daemon start namenode
!hdfs --daemon start datanode

#Check they’re running
!jps
```
### Step 5 — Create HDFS Directories

```
!hdfs dfs -mkdir /cyt180
!hdfs dfs -ls /
```

### Step 6 — Load Cybersecurity Logs into HDFS
We will download a real `Zeek conn.log.gz` file from the public zed-sample-data repository on GitHub.
Run the following command one by one.

```
!wget -q https://raw.githubusercontent.com/brimdata/zed-sample-data/main/zeek-default/conn.log.gz

#Confirm it's downloaded:
!ls -lh conn.log.gz

# Unzip it
!gunzip -f conn.log.gz

```
After unzipping, the file conn.log will appear in your Colab directory.
**Upload to HDFS**

```
!hdfs dfs -mkdir -p /logs
!hdfs dfs -put conn.log /logs/
!hdfs dfs -ls /logs
```

### Step 7 — Inspect the Zeek Log in HDFS
Before doing MapReduce, you should see what the data looks like.
Run the following command:
```
!hdfs dfs -head /logs/conn.log
```
A typical conn.log line looks like:
```
1672531200.12345 Ckq8..... 192.168.1.10 12345 10.0.0.5 80 tcp ssl 1 52 104 ...
```
**Columns include:**

- Timestamp
- UID
- Origin IP / port
- Destination IP / port
- Protocol
- Connection state
- Bytes sent/received

A full description of fields can be found in Zeek documentation (not needed for the lab).

### Step 8 — MapReduce Example: 
Hadoop ships with several example jobs in the directory:
```
$HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar
```
The most classic one is WordCount.
Even though it's not cybersecurity‑specific, WordCount is perfect for:

- learning Map → shuffle → reduce
- validating that HDFS + MapReduce works
- running a real distributed job on the Zeek log

And Zeek logs are plaintext, so running WordCount on them works immediately.

**Run WordCount on the Zeek conn.log**
```
!hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar \
    wordcount \
    /logs/conn.log \
    /output_wordcount
```
**View the results**
```
!hdfs dfs -cat /output_wordcount/part-r-00000 | head
```

### Step 9 — Use Hadoop’s Built‑In Grep Example
Hadoop also has a built‑in grep job for filtering matching lines.
We'll use it to search for all TCP connections inside the Zeek log.
Run grep to extract lines containing `tcp`. Since `tcp` appears in the protocol field of most flows, so this will return lines containing TCP traffic.
```

!hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar \
    grep \
    /logs/conn.log \
    /output_grep_tcp \
    tcp
```
**View results**
```
!hdfs dfs -cat /output_grep_tcp/part-r-00000 | head -20
```

## Conclusion
In this lab, you successfully:

- Installed and configured Hadoop inside Google Colab
- Set up HDFS in pseudo‑distributed mode
- Downloaded a real Zeek conn.log file
- Uploaded it into HDFS
- Ran two essential built‑in Hadoop MapReduce jobs:
  - WordCount — to get high‑level insights from the log
  - Grep — to filter the dataset based on patterns


## Submission Instructions

Submit **3–5 screenshots** that clearly show the work you completed in this lab.

Your screenshots **must include** the following:

1. **HDFS Setup Verification**
   - Output of:
     ```bash
     !hdfs dfs -ls /
     ```
     showing the `/logs` and `/cyt180` directories.

2. **WordCount Job Output**
   - A screenshot showing the result of:
     ```bash
     !hdfs dfs -cat /output_wordcount/part-r-00000 | head
     ```

3. **Grep Job Output**
   - A screenshot showing the result of:
     ```bash
     !hdfs dfs -cat /output_grep_tcp/part-r-00000 | head -20
     ```

4. **(Optional but recommended)**
   - A screenshot of the Hadoop daemons running:
     ```bash
     !jps
     ```
     *or*
   - A screenshot showing the `conn.log` file after unzipping and before uploading.

5. Please ensure all screenshots are **clear, readable, and taken from your own Colab session**.
6. With each screenshot, include date and time as a proof from a separate terminal window.
7. Submit your screenshots as a **single PDF**.
