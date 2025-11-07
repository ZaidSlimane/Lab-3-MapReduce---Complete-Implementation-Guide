# Lab 3: MapReduce - Complete Implementation Guide

## Lab Information
- **Course**: M2 ILSI - Big Data 2025–2026
- **University**: University of Constantine 2 - Abdelhamid Mehri
- **Duration**: 1 session

---

## Section 2.1: File Creation and Upload

### Step 1: Access Master Node Container Shell
```bash
# If using Docker
docker exec -it <master-container-name> /bin/bash

# Or if using direct SSH
ssh <master-node>
```

### Step 2: Verify Hadoop Version
```bash
hadoop version
```

### Step 3: Start Hadoop and Verify JVM Processes
```bash
# Start Hadoop services
start-dfs.sh
start-yarn.sh

# Verify running processes
jps
```

**Expected JPS Output:**
- NameNode
- SecondaryNameNode
- ResourceManager
- Jps

### Step 4: Create testfile2.txt
```bash
cat > testfile2.txt << 'EOF'
Consistency means that all the nodes ( databases ) inside a network will have the same copies of a replicated data item visible for various transactions . It guarantees that every node in a distributed cluster returns the same , most recent , and successful write . It refers to every client having the same view of the data . There are various types of consistency models . Consistency in CAP refers to sequential consistency , a very strong form of consistency . Note that the concept of Consistency in ACID and CAP are slightly different
EOF
```

### Step 5: Upload File to HDFS
```bash
# Create directory if it doesn't exist
hdfs dfs -mkdir -p /data/bigdata_ilsi

# Upload the file
hdfs dfs -put testfile2.txt /data/bigdata_ilsi/

# Verify upload
hdfs dfs -ls /data/bigdata_ilsi/
```

### Step 6: Display Contents of Input Directory
```bash
hdfs dfs -ls /data/bigdata_ilsi/
hdfs dfs -cat /data/bigdata_ilsi/testfile2.txt
```

---

## Section 2.2: Prepare Python Mapper and Reducer Code

### Step 1: Create mapper.py
```bash
cat > mapper.py << 'EOF'
#!/usr/bin/env python3
import sys
import re

def main():
    for line in sys.stdin:
        # Remove leading/trailing whitespace
        line = line.strip()
        # Split the line into words
        words = re.findall(r'\w+', line.lower())
        # Output each word with count 1
        for word in words:
            print(f"{word}\t1")

if __name__ == "__main__":
    main()
EOF
```

### Step 2: Create reducer.py
```bash
cat > reducer.py << 'EOF'
#!/usr/bin/env python3
import sys

def main():
    current_word = None
    current_count = 0
    
    for line in sys.stdin:
        # Remove leading/trailing whitespace
        line = line.strip()
        # Parse the input from mapper
        word, count = line.split('\t', 1)
        
        # Convert count to int
        try:
            count = int(count)
        except ValueError:
            continue
        
        # If same word, increment count
        if current_word == word:
            current_count += count
        else:
            # If new word, output the previous word
            if current_word:
                print(f"{current_word}\t{current_count}")
            current_word = word
            current_count = count
    
    # Output the last word
    if current_word:
        print(f"{current_word}\t{current_count}")

if __name__ == "__main__":
    main()
EOF
```

### Step 3: Make Scripts Executable
```bash
chmod +x mapper.py
chmod +x reducer.py
```

### Step 4: Test Scripts Locally
```bash
echo "hello world hello hadoop" | ./mapper.py | sort | ./reducer.py
```

**Expected Output:**
```
hadoop	1
hello	2
world	1
```

---

## Section 2.3: Run Hadoop Streaming Job

### Step 1: Execute Hadoop Streaming Job
```bash
hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-3.3.2.jar \
  -files mapper.py,reducer.py \
  -input /data/bigdata_ilsi/testfile2.txt \
  -output /data/bigdata_ilsi/output \
  -mapper mapper.py \
  -reducer reducer.py
```

**Note**: If output directory exists, remove it first:
```bash
hdfs dfs -rm -r /data/bigdata_ilsi/output
```

### Step 2: List Output Files
```bash
hdfs dfs -ls /data/bigdata_ilsi/output
```

**Expected Output:**
- `_SUCCESS` (indicates successful completion)
- `part-00000` (output file)

### Step 3: View Results (First 20 Lines)
```bash
hdfs dfs -cat /data/bigdata_ilsi/output/part-00000 | head -20
```

### Step 4: Get Top 10 Most Frequent Words
```bash
hdfs dfs -cat /data/bigdata_ilsi/output/part-00000 | sort -k2 -nr | head -10
```

### Step 5: Create Directory for Large Input
```bash
hdfs dfs -mkdir -p /input-large
```

### Step 6: Create Multiple Input Files
```bash
for i in {1..25}; do
  hdfs dfs -put testfile2.txt /input-large/file$i.txt
done

# Verify
hdfs dfs -ls /input-large/ | wc -l
```

### Step 7: Launch Long-Running MapReduce Job with Multiple Reducers
```bash
hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-3.3.2.jar \
  -files mapper.py,reducer.py \
  -input /input-large \
  -output /output-large \
  -mapper mapper.py \
  -reducer reducer.py \
  -numReduceTasks 2
```

### Step 8: Monitor HDFS Cluster Health (In Another Terminal)
```bash
# Open another terminal and access master node
docker exec -it <master-container-name> /bin/bash

# Monitor cluster health
hdfs dfsadmin -report | grep -E "(Live datanodes|Dead datanodes|Under replicated|Missing blocks)"

# Or watch continuously
watch -n 2 'hdfs dfsadmin -report | grep -E "(Live datanodes|Dead datanodes|Under replicated|Missing blocks)"'
```

### Step 9: Shutdown One Worker Node and Monitor
```bash
# In another terminal, access worker node
docker stop <worker-container-name>

# Or if SSH
ssh <worker-node>
sudo systemctl stop hadoop-datanode
```

**Continue monitoring in the master terminal:**
```bash
hdfs dfsadmin -report | grep -E "(Live datanodes|Dead datanodes|Under replicated|Missing blocks)"
```

---

## Question 10: Analysis

**Question**: Based on the HDFS health status you observed after stopping the worker node, what does the behavior of the "Under replicated blocks" metric reveal about Hadoop's fundamental approach to data reliability, and how might this design philosophy impact real-world enterprise data management decisions?

**Answer**:

When a worker node is shutdown, you'll observe the following HDFS behavior:

1. **Under-replicated Blocks Increase**: The "Under replicated blocks" metric increases immediately after a DataNode goes down, indicating that some data blocks no longer meet the configured replication factor (default is 3).

2. **Hadoop's Fundamental Approach to Data Reliability**:
   - **Proactive Replication**: HDFS detects under-replicated blocks and automatically initiates re-replication to restore the target replication factor
   - **Fault Tolerance by Design**: The system assumes node failures are normal and expected, not exceptional
   - **No Data Loss**: As long as at least one replica remains, no data is lost
   - **Self-Healing**: The NameNode monitors block replication and coordinates automatic recovery

3. **Impact on Enterprise Data Management Decisions**:
   
   **Advantages**:
   - **High Availability**: Data remains accessible even during hardware failures
   - **Reduced Downtime**: No immediate intervention required for node failures
   - **Cost-Effective**: Uses commodity hardware instead of expensive fault-tolerant systems
   - **Scalability**: Easy to add nodes without complex configuration
   
   **Considerations**:
   - **Storage Overhead**: 3x replication means 300% storage cost
   - **Network Traffic**: Re-replication generates significant network load
   - **Recovery Time**: Large datasets take time to re-replicate
   - **Resource Planning**: Must account for replication factor in capacity planning
   
   **Real-World Implications**:
   - Organizations must balance replication factor (reliability) vs. storage costs
   - Network bandwidth becomes critical for cluster health
   - Monitoring and alerting systems are essential for proactive management
   - Disaster recovery planning should consider geographic distribution of replicas

---

## Additional Monitoring Commands

### Check Cluster Status
```bash
# Detailed cluster report
hdfs dfsadmin -report

# File system check
hdfs fsck / -files -blocks -locations

# List dead nodes
hdfs dfsadmin -printTopology
```

### YARN Monitoring
```bash
# Check YARN applications
yarn application -list

# Application status
yarn application -status <application_id>

# Node status
yarn node -list
```

### Web UI Access
- **HDFS NameNode**: http://<master-node>:9870
- **YARN ResourceManager**: http://<master-node>:8088
- **MapReduce JobHistory**: http://<master-node>:19888

---

## Key Concepts

### MapReduce Components
- **ResourceManager**: Manages cluster resources and schedules tasks
- **NodeManager**: Runs on worker nodes, executes tasks
- **ApplicationMaster**: Manages a single MapReduce job

### HDFS Replication
- **Default Replication Factor**: 3
- **Rack Awareness**: Places replicas on different racks for fault tolerance
- **Block Size**: Default 128MB (configurable)

### Word Count Flow
1. **Input**: Text file in HDFS
2. **Map Phase**: Tokenize words and emit (word, 1) pairs
3. **Shuffle & Sort**: Group all values by key (word)
4. **Reduce Phase**: Sum counts for each word
5. **Output**: Word frequency list in HDFS

---

## Troubleshooting

### Common Issues

1. **Output directory already exists**
   ```bash
   hdfs dfs -rm -r /data/bigdata_ilsi/output
   ```

2. **Permission denied on scripts**
   ```bash
   chmod +x mapper.py reducer.py
   ```

3. **Python not found**
   ```bash
   which python3
   # Update shebang if needed
   ```

4. **HDFS in safe mode**
   ```bash
   hdfs dfsadmin -safemode leave
   ```

5. **DataNode not starting**
   ```bash
   # Check logs
   tail -f $HADOOP_HOME/logs/hadoop-*-datanode-*.log
   ```

---

## Expected Results

After running the word count on testfile2.txt, the most frequent words should be:
- "consistency" (appears multiple times)
- "the" (appears multiple times)
- "a" (appears multiple times)
- Various other terms related to distributed systems

The exact counts depend on the tokenization rules in the mapper.

---

## Submission Checklist

- [ ] Screenshots of JPS output showing running services
- [ ] Screenshot of HDFS directory listing
- [ ] MapReduce job completion output
- [ ] Top 10 words from word count
- [ ] HDFS health report before and after node shutdown
- [ ] Answer to Question 10 with analysis

---

## Resources

- [Hadoop Documentation](https://hadoop.apache.org/docs/current/)
- [Hadoop Streaming Guide](https://hadoop.apache.org/docs/current/hadoop-streaming/HadoopStreaming.html)
- [HDFS Architecture](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)

---

**Lab Completed Successfully! 🎉**
