# HDFS & MapReduce — Complete Reference Notes
> Personal study notes covering history, theory, storage/retrieval internals, fault tolerance, Python coding, and real-world examples.

---

## Table of Contents
1. [History & Motivation](#1-history--motivation)
2. [HDFS Architecture](#2-hdfs-architecture)
3. [How Data is Stored — Block Storage Internals](#3-how-data-is-stored--block-storage-internals)
4. [How Data is Retrieved Efficiently](#4-how-data-is-retrieved-efficiently)
5. [Fault Tolerance in HDFS](#5-fault-tolerance-in-hdfs)
6. [MapReduce — Theory](#6-mapreduce--theory)
7. [MapReduce — Execution Lifecycle](#7-mapreduce--execution-lifecycle)
8. [Fault Tolerance in MapReduce](#8-fault-tolerance-in-mapreduce)
9. [Python Code Examples](#9-python-code-examples)
10. [Real-World Examples](#10-real-world-examples)
11. [Quick-Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. History & Motivation

### The Problem (Early 2000s)
- Web was exploding — Google was crawling **billions of web pages**
- A single machine **could not** store or process this data
- Existing distributed systems (like MPI) required careful programming and were brittle

### Google's Papers (The Foundation)
| Year | Paper | What It Gave Us |
|------|-------|-----------------|
| 2003 | **Google File System (GFS)** | Distributed, fault-tolerant storage on cheap hardware |
| 2004 | **MapReduce** | Simple programming model for parallel data processing |
| 2006 | **Bigtable** | Structured storage on top of GFS |

### Hadoop Birth
- Doug Cutting & Mike Cafarella were building **Nutch** (open-source web crawler)
- GFS and MapReduce papers inspired them to build open-source equivalents
- 2006: Hadoop spun out of Nutch into its own Apache project
- **HDFS** = open-source GFS | **Hadoop MapReduce** = open-source MapReduce
- Yahoo! was the first major adopter (2008), used it to sort 1 TB in 209 seconds

### Why It Was Revolutionary
- Run on **commodity hardware** (no expensive servers)
- **"Move compute to data"** — don't move huge data over network; run code where data lives
- Horizontal scaling — add more cheap nodes instead of bigger expensive nodes

---

## 2. HDFS Architecture

```
         ┌─────────────────────────────────────────────────────────┐
         │                     HDFS CLUSTER                        │
         │                                                         │
         │   ┌──────────────┐         ┌────────────────────────┐  │
         │   │  NameNode    │         │  Secondary NameNode     │  │
         │   │  (Master)    │◄───────►│  (Checkpoint Helper)   │  │
         │   │              │         └────────────────────────┘  │
         │   │ - FsImage    │                                      │
         │   │ - EditLog    │                                      │
         │   │ - Namespace  │                                      │
         │   └──────┬───────┘                                      │
         │          │  heartbeat + block reports                   │
         │    ┌─────┼─────────────────────┐                       │
         │    ▼     ▼                     ▼                       │
         │ ┌──────┐ ┌──────┐         ┌──────┐                    │
         │ │ DN-1 │ │ DN-2 │   ...   │ DN-N │   (DataNodes)      │
         │ │Blk A │ │Blk A │         │Blk B │                    │
         │ │Blk C │ │Blk D │         │Blk A │                    │
         │ └──────┘ └──────┘         └──────┘                    │
         └─────────────────────────────────────────────────────────┘
```

### NameNode (The Master / Brain)
- Stores the **filesystem namespace** (directory tree, file metadata)
- Maintains **block-to-DataNode mapping** in memory (very fast lookups)
- Persists state via:
  - `FsImage` — snapshot of the entire filesystem namespace
  - `EditLog` — write-ahead log of every change since last snapshot
- Does **NOT** store actual data — only metadata
- Single point of failure → mitigated with HA NameNode in Hadoop 2+

### DataNode (The Workers / Storage)
- Actually stores **blocks of data** on local disk
- Sends **heartbeats** to NameNode every 3 seconds (alive signal)
- Sends **block reports** every 6 hours (full inventory of blocks it holds)
- Serves read/write requests from clients directly
- Performs **block replication** on instruction from NameNode

### Secondary NameNode (Misnomer — NOT a backup)
- Periodically merges `FsImage` + `EditLog` → new `FsImage` (checkpointing)
- Prevents `EditLog` from growing too large
- In Hadoop HA mode, replaced by **Standby NameNode** (true failover)

---

## 3. How Data is Stored — Block Storage Internals

### Block-Based Storage
- HDFS splits every file into fixed-size **blocks** (default: **128 MB** in Hadoop 2+, was 64 MB in v1)
- Each block is stored as a **separate file** on the DataNode's local filesystem
- A 500 MB file → 4 blocks: [128MB][128MB][128MB][116MB]

```
File: /data/sales.csv  (500 MB)
         │
         ├── Block 0 (blk_1073741825) → 128 MB  → DN-1, DN-3, DN-5
         ├── Block 1 (blk_1073741826) → 128 MB  → DN-2, DN-4, DN-1
         ├── Block 2 (blk_1073741827) → 128 MB  → DN-3, DN-5, DN-2
         └── Block 3 (blk_1073741828) → 116 MB  → DN-4, DN-1, DN-3
```

### Replication
- Default **replication factor = 3** (configurable per file)
- NameNode uses a **Rack-Aware Placement Policy**:
  - Replica 1 → local DataNode (where client writes)
  - Replica 2 → different DataNode in a **different rack**
  - Replica 3 → different DataNode in same rack as Replica 2
- Why? Rack failure is more common than individual node failure

```
Rack 1                     Rack 2
┌────────────────┐         ┌────────────────┐
│  DN-1 [R1]     │         │  DN-3 [R2, R3] │
│  DN-2          │         │  DN-4          │
└────────────────┘         └────────────────┘
         Client writes → DN-1 → DN-3 → DN-4 (pipeline)
```

### Write Pipeline
1. Client asks NameNode: "I want to write file X"
2. NameNode returns list of DataNodes for each block
3. Client sends data to **DN-1** in a streaming pipeline
4. DN-1 forwards to **DN-2**, DN-2 forwards to **DN-3** (simultaneously)
5. Acknowledgement flows back: DN-3 → DN-2 → DN-1 → Client
6. Client notifies NameNode: "Block written successfully"

### Why 128 MB Blocks?
- Reduces metadata overhead (fewer blocks = less NameNode memory)
- Optimized for **sequential reads** (good for MapReduce batch jobs)
- Disk seek time is negligible compared to transfer time at this size
- Trade-off: bad for small files (a 1 KB file still uses a full block entry in NameNode)

---

## 4. How Data is Retrieved Efficiently

### Read Path
```
Client                 NameNode              DataNode(s)
  │                       │                      │
  │── getBlockLocations ──►│                      │
  │◄── [{blk1: [DN2, DN4, DN1]},                 │
  │      {blk2: [DN3, DN1, DN5]}] ───────────────│
  │                       │                      │
  │──────── read(blk1) ──────────────────────────►│ (closest DN)
  │◄──────── data stream ────────────────────────│
  │                       │                      │
```

1. Client calls `open()` on HDFS file
2. **DistributedFileSystem** contacts NameNode → gets block locations (sorted by network proximity)
3. Client opens **FSDataInputStream** → directly contacts the **closest DataNode** for each block
4. Reads blocks sequentially; if a DataNode fails mid-read, transparently switches to the next replica
5. NameNode is **not in the data path** — client reads directly from DataNodes (scales well)

### Data Locality Optimization
- MapReduce scheduler tries to run tasks **on the same node where data lives**
- Rack-local if not node-local; any node if neither is possible
- This is the famous **"move compute to data"** principle

### HDFS Caching (Hadoop 2.3+)
- Frequently accessed data can be **pinned in DataNode memory** (off-heap)
- NameNode manages a centralized cache directive system
- MapReduce and Hive jobs can request cached blocks for near-memory-speed reads

---

## 5. Fault Tolerance in HDFS

### DataNode Failure
| Event | Detection | Recovery |
|-------|-----------|----------|
| DataNode dies | NameNode stops receiving heartbeats (10 min timeout) | NameNode marks blocks under-replicated; triggers re-replication to other DataNodes |
| Disk corruption | DataNode computes checksum on read; mismatch detected | Reports bad block to NameNode; NameNode fetches good replica from another DataNode |
| Network partition | Heartbeats time out | Same as node failure |

**Under-replication trigger:**
```
Block A should have 3 replicas
DN-2 dies → Block A now has 2 replicas
NameNode: re-replicate Block A from DN-1 to DN-5
```

### NameNode Failure (The Critical One)
- NameNode is the **SPOF** in Hadoop 1.x
- Hadoop 2.x introduced **HA NameNode**:
  - **Active NameNode** handles all client requests
  - **Standby NameNode** maintains identical state via shared edit log (NFS or QJM — Quorum Journal Manager)
  - **ZooKeeper** manages failover (detects active NN failure, promotes standby)
  - Failover in < 30 seconds (automatic)

### Data Integrity — Checksums
- Every block has a **CRC-32C checksum** stored alongside it
- On write: client computes checksum, DataNode verifies and stores
- On read: DataNode recomputes checksum; if mismatch → bad data → fetch from replica
- Protects against **bit rot** and disk errors

### Balancer
- Over time, some DataNodes fill up faster than others (hot spots)
- HDFS **Balancer** runs as a background process to redistribute blocks
- Command: `hdfs balancer -threshold 10` (equalize to within 10%)

---

## 6. MapReduce — Theory

### Core Idea
Process data in two functional steps inspired by Lisp/Haskell:
- **Map**: Transform each input record → emit key-value pairs
- **Reduce**: Aggregate all values for each key → produce final output

### Formal Definition
```
map(k1, v1)       → list(k2, v2)
reduce(k2, list(v2)) → list(k3, v3)
```

### Word Count Walkthrough (Classic Example)
```
Input: "the cat sat on the mat the cat"

MAP phase:
  "the"  → (the, 1)
  "cat"  → (cat, 1)
  "sat"  → (sat, 1)
  "on"   → (on, 1)
  "the"  → (the, 1)
  "mat"  → (mat, 1)
  "the"  → (the, 1)
  "cat"  → (cat, 1)

SHUFFLE & SORT (framework does this automatically):
  cat → [1, 1]
  mat → [1]
  on  → [1]
  sat → [1]
  the → [1, 1, 1]

REDUCE phase:
  cat → sum([1,1]) → (cat, 2)
  mat → sum([1])   → (mat, 1)
  sat → sum([1])   → (sat, 1)
  the → sum([1,1,1]) → (the, 3)

Output written to HDFS
```

---

## 7. MapReduce — Execution Lifecycle

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MapReduce Job Execution                           │
│                                                                      │
│  HDFS Input     Map Phase         Shuffle/Sort     Reduce Phase      │
│  ─────────      ─────────         ────────────     ────────────      │
│                                                                      │
│  [Block 1] ──► [Mapper 1] ──►│                                      │
│  [Block 2] ──► [Mapper 2] ──►│ Sort by key  ──►  [Reducer 1] ──► HDFS│
│  [Block 3] ──► [Mapper 3] ──►│ Partition   ──►  [Reducer 2] ──► HDFS│
│  [Block 4] ──► [Mapper 4] ──►│             ──►  [Reducer N] ──► HDFS│
│                               │                                      │
│                          [Combiner]                                  │
│                       (optional mini-reduce)                         │
└──────────────────────────────────────────────────────────────────────┘
```

### Detailed Steps

**Step 1 — Input Split**
- Input data divided into **InputSplits** (usually = 1 HDFS block)
- One Mapper per split
- More splits = more parallelism (but overhead per task)

**Step 2 — Map**
- Each mapper processes its split line by line (for TextInputFormat)
- Outputs key-value pairs to **in-memory circular buffer** (default 100 MB)
- When buffer hits 80% → **spill to local disk** (sorted + partitioned)

**Step 3 — Combiner (Optional)**
- A "local reducer" that runs on mapper output before network transfer
- Reduces data shuffled over the network dramatically
- Must be commutative and associative (e.g., sum, count — not average)

**Step 4 — Shuffle & Sort**
- Framework transfers mapper output to reducers across the network
- Data for each reducer key is **sorted** and **merged** (external merge sort)
- This is usually the most expensive phase

**Step 5 — Reduce**
- Each reducer receives all values for its assigned keys
- Runs user's reduce function
- Output written to HDFS

**Step 6 — Output**
- Results stored in HDFS output directory
- One output file per reducer (`part-r-00000`, `part-r-00001`, etc.)

### Key Configurations
```bash
# Number of reducers (default: 1)
mapreduce.job.reduces=10

# Map output compression (reduces shuffle traffic)
mapreduce.map.output.compress=true

# Combiner class
mapreduce.job.combiner.class=WordCountReducer
```

---

## 8. Fault Tolerance in MapReduce

### Task Failure
- If a mapper/reducer fails (exception, timeout, node crash):
  - ApplicationMaster (YARN) detects via heartbeat timeout
  - Task is **automatically re-run** on a different node
  - Attempted up to 4 times (configurable) before job fails

### ApplicationMaster Failure
- YARN ResourceManager restarts the AM
- AM recovers job state from history and re-runs incomplete tasks
- Configurable: `yarn.app.mapreduce.am.max-attempts=2`

### Node Manager Failure
- ResourceManager detects via missed heartbeats
- All tasks on that node are rescheduled elsewhere
- Any in-progress reduce tasks that were reading from that node's mappers retry

### Speculative Execution
- Problem: one slow node (straggler) delays the whole job
- Solution: if a task runs much slower than peers, launch a **duplicate speculative task** on another node
- First one to finish wins; the other is killed
- Enabled by default: `mapreduce.map.speculative=true`

### Data Already in HDFS
- MapReduce reads from HDFS — data is already replicated 3x
- Even if a DataNode dies mid-job, the task simply reads from another replica

---

## 9. Python Code Examples

### 9.1 Hadoop Streaming — Word Count (Simplest)
Hadoop Streaming lets you write mappers/reducers as stdin/stdout scripts in any language.

**mapper.py**
```python
#!/usr/bin/env python3
"""Mapper: reads lines from stdin, emits (word, 1) pairs."""
import sys

for line in sys.stdin:
    line = line.strip()
    words = line.split()
    for word in words:
        # Emit tab-separated key-value pair
        print(f"{word.lower()}\t1")
```

**reducer.py**
```python
#!/usr/bin/env python3
"""Reducer: reads sorted (word, count) from stdin, sums counts per word."""
import sys
from itertools import groupby

def read_mapper_output(stream):
    for line in stream:
        line = line.strip()
        if line:
            key, value = line.split('\t', 1)
            yield key, int(value)

# Input is already sorted by key (Hadoop guarantees this)
for key, group in groupby(read_mapper_output(sys.stdin), key=lambda x: x[0]):
    total = sum(count for _, count in group)
    print(f"{key}\t{total}")
```

**Run on Hadoop:**
```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input  /user/data/input.txt \
  -output /user/data/wordcount_output \
  -mapper  mapper.py \
  -reducer reducer.py \
  -file    mapper.py \
  -file    reducer.py
```

---

### 9.2 MRJob — Pythonic MapReduce (mrjob library)
`mrjob` runs the same code locally, on Hadoop, or on AWS EMR.

```bash
pip install mrjob
```

**word_count.py**
```python
from mrjob.job import MRJob
from mrjob.step import MRStep
import re

WORD_RE = re.compile(r"[\w']+")

class MRWordCount(MRJob):

    def steps(self):
        return [
            MRStep(mapper=self.mapper_get_words,
                   combiner=self.combiner_count_words,   # local mini-reduce
                   reducer=self.reducer_count_words)
        ]

    def mapper_get_words(self, _, line):
        """Emit (word, 1) for each word in the line."""
        for word in WORD_RE.findall(line.lower()):
            yield word, 1

    def combiner_count_words(self, word, counts):
        """Local sum to reduce shuffle traffic."""
        yield word, sum(counts)

    def reducer_count_words(self, word, counts):
        """Final sum per word."""
        yield word, sum(counts)


if __name__ == '__main__':
    MRWordCount.run()
```

**Run locally (for testing):**
```bash
python word_count.py input.txt
```

**Run on Hadoop:**
```bash
python word_count.py -r hadoop hdfs:///user/data/input.txt
```

**Run on AWS EMR:**
```bash
python word_count.py -r emr s3://mybucket/input/ --output-dir s3://mybucket/output/
```

---

### 9.3 Multi-Step MapReduce — Top N Words
Two chained MapReduce jobs: first count, then find top-N.

```python
from mrjob.job import MRJob
from mrjob.step import MRStep

class MRTopNWords(MRJob):

    TOP_N = 10

    def steps(self):
        return [
            MRStep(
                mapper=self.mapper_count,
                combiner=self.combiner_count,
                reducer=self.reducer_count
            ),
            MRStep(
                mapper=self.mapper_prepare_sort,
                reducer=self.reducer_find_top_n
            )
        ]

    # ── Step 1: Count words ──────────────────────────────────────────
    def mapper_count(self, _, line):
        for word in line.lower().split():
            word = word.strip('.,!?";:')
            if word:
                yield word, 1

    def combiner_count(self, word, counts):
        yield word, sum(counts)

    def reducer_count(self, word, counts):
        yield word, sum(counts)

    # ── Step 2: Sort & pick top N ────────────────────────────────────
    def mapper_prepare_sort(self, word, count):
        # Emit with negative count so highest sorts first
        # Use None as key so all go to a single reducer
        yield None, (count, word)

    def reducer_find_top_n(self, _, word_count_pairs):
        # Sort descending by count; take top N
        top = sorted(word_count_pairs, key=lambda x: -x[0])[:self.TOP_N]
        for count, word in top:
            yield word, count


if __name__ == '__main__':
    MRTopNWords.run()
```

---

### 9.4 Custom Partitioner Example (Conceptual)
By default, Hadoop partitions by `hash(key) % numReducers`. Custom partitioner routes specific keys to specific reducers.

```python
from mrjob.job import MRJob
from mrjob.step import MRStep

class MRSalesAnalysis(MRJob):
    """Route sales data by region to specific reducers."""

    REGION_MAP = {"north": 0, "south": 1, "east": 2, "west": 3}

    def mapper(self, _, line):
        """
        Input line format: region,product,amount
        e.g.:  north,laptop,1200
        """
        parts = line.strip().split(',')
        if len(parts) == 3:
            region, product, amount = parts
            yield (region, product), float(amount)

    def reducer(self, key, amounts):
        region, product = key
        yield region, {product: sum(amounts)}

    def steps(self):
        return [MRStep(mapper=self.mapper, reducer=self.reducer)]

if __name__ == '__main__':
    MRSalesAnalysis.run()
```

---

### 9.5 Simulating MapReduce Logic Locally in Pure Python
Useful for understanding internals without a Hadoop cluster.

```python
"""
Simulate the full MapReduce pipeline in pure Python.
Demonstrates: Map → Shuffle/Sort → Reduce
"""
from collections import defaultdict
from itertools import groupby

# ── Sample data (simulating HDFS blocks) ────────────────────────────
documents = [
    "the quick brown fox",
    "the fox jumped over the lazy dog",
    "the dog barked at the fox",
]

# ── MAP phase ────────────────────────────────────────────────────────
def mapper(doc_id, text):
    """Emit (word, 1) for each word."""
    for word in text.lower().split():
        yield word, 1

print("=== MAP OUTPUT ===")
mapped = []
for doc_id, doc in enumerate(documents):
    for kv in mapper(doc_id, doc):
        mapped.append(kv)
        print(f"  {kv}")

# ── SHUFFLE & SORT phase (framework does this) ────────────────────────
print("\n=== SHUFFLE & SORT ===")
mapped.sort(key=lambda x: x[0])  # Sort by key
for k, v in mapped:
    print(f"  {k}: {v}")

# ── REDUCE phase ─────────────────────────────────────────────────────
def reducer(key, values):
    """Sum all counts for a word."""
    return key, sum(values)

print("\n=== REDUCE OUTPUT ===")
results = {}
for key, group in groupby(mapped, key=lambda x: x[0]):
    values = [v for _, v in group]
    out_key, out_val = reducer(key, values)
    results[out_key] = out_val
    print(f"  {out_key}: {out_val}")

print("\n=== FINAL WORD COUNTS ===")
for word, count in sorted(results.items(), key=lambda x: -x[1]):
    print(f"  {word:15} {count}")
```

**Output:**
```
=== FINAL WORD COUNTS ===
  the             6
  fox             3
  dog             2
  ...
```

---

### 9.6 Interacting with HDFS via Python (PyArrow / hdfs3)

```python
# Option A: PyArrow (modern, recommended)
import pyarrow.fs as pafs

hdfs = pafs.HadoopFileSystem(host="namenode-host", port=8020)

# List directory
entries = hdfs.get_file_info(pafs.FileSelector('/user/data'))
for e in entries:
    print(e.path, e.size)

# Read a file
with hdfs.open_input_stream('/user/data/sales.csv') as f:
    content = f.read().decode('utf-8')
    print(content[:500])

# Write a file
with hdfs.open_output_stream('/user/data/output.txt') as f:
    f.write(b"Hello HDFS from Python!\n")
```

```python
# Option B: hdfs (hdfscli) library
from hdfs import InsecureClient

client = InsecureClient('http://namenode-host:9870', user='hdfs')

# Upload local file to HDFS
client.upload('/user/data/myfile.csv', '/local/path/myfile.csv')

# Download from HDFS
client.download('/user/data/output/', '/local/output/', overwrite=True)

# Read directly
with client.read('/user/data/sales.csv', encoding='utf-8') as reader:
    for line in reader:
        print(line.strip())
```

---

## 10. Real-World Examples

### 10.1 Facebook — Log Analysis
- **Problem**: 500 TB of server logs generated daily; need ad click analysis in hours
- **HDFS**: Logs streamed to HDFS via Flume; each server's logs = one HDFS file
- **MapReduce Job**:
  - Map: Parse log line → emit `(user_id, ad_id, 1)`
  - Reduce: `(user_id, ad_id) → total_clicks`
- **Scale**: 4,000+ nodes, processed 1 PB/day at peak

### 10.2 Netflix — Recommendation Engine Batch Jobs
- **Problem**: Generate personalized recommendations for 200M+ users nightly
- **HDFS**: Stores viewing history, ratings, interaction events
- **MapReduce**:
  - Job 1 (Map): User → list of watched titles
  - Job 2 (Reduce): Co-occurrence matrix computation (users who watched X also watched Y)
  - Job 3: Score and rank recommendations per user
- **Result**: Batch results fed into real-time serving layer

### 10.3 Financial Services — Fraud Detection
- **Problem**: Detect fraudulent transactions in 500M daily transaction records
- **HDFS storage**: Transaction data partitioned by date/region
- **MapReduce pipeline**:
  - Map: Transaction → `(account_id, transaction_detail)`
  - Reduce: For each account, analyze pattern: velocity, geo-anomaly, amount spike
  - Output: Risk scores written back to HDFS, loaded into fraud alert system
- **Fault tolerance value**: A single corrupt transaction record doesn't fail the job — checksum catches it, reducer just skips that record

### 10.4 NYT — Digitizing 11 Million Articles (Famous Hadoop Story, 2008)
- New York Times converted 11 million scanned images to PDFs
- Used 100 EC2 instances with Hadoop Streaming
- Completed in **24 hours for ~$240** (would've taken weeks on a single machine)
- Mapper: Convert each TIFF → PDF (independent, embarrassingly parallel)
- Reducer: None needed (map-only job)

### 10.5 E-Commerce — Inventory & Sales Aggregation
```
Input (HDFS): /data/sales/2024-01-01/transactions.csv
  txn_id, product_id, qty, price, store_id

MapReduce Job:
  Mapper:  (product_id, revenue = qty × price)
  Reducer: SUM(revenue) per product_id

Output: /data/reports/2024-01-01/product_revenue/
  product_id → total_revenue

Secondary Job:
  Mapper:  (store_id, product_id, revenue)
  Reducer: TOP-5 products per store
```

---

## 11. Quick-Reference Cheat Sheet

### HDFS Key Numbers
| Parameter | Default | Notes |
|-----------|---------|-------|
| Block size | 128 MB | Configurable per file |
| Replication factor | 3 | `dfs.replication` |
| Heartbeat interval | 3 seconds | DataNode → NameNode |
| Block report interval | 6 hours | Full block inventory |
| DataNode timeout | 10 minutes | Before marked dead |
| Checksum | CRC-32C | Per block |

### MapReduce Key Concepts
| Concept | Purpose |
|---------|---------|
| InputSplit | Logical division of input data (≈ 1 block) |
| Mapper | Transforms records → key-value pairs |
| Combiner | Local mini-reduce, reduces shuffle data |
| Partitioner | Routes mapper output to correct reducer |
| Shuffle & Sort | Transfers + sorts data between Map and Reduce |
| Reducer | Aggregates all values per key |
| Speculative Execution | Duplicate slow tasks to avoid stragglers |

### HDFS CLI Quick Reference
```bash
# List files
hdfs dfs -ls /user/data/

# Copy local → HDFS
hdfs dfs -put localfile.txt /user/data/

# Copy HDFS → local
hdfs dfs -get /user/data/output.txt ./

# View file (cat)
hdfs dfs -cat /user/data/file.txt | head -20

# Check disk usage
hdfs dfs -du -h /user/data/

# Change replication factor
hdfs dfs -setrep -w 5 /user/data/important_file.csv

# Check HDFS health
hdfs fsck / -files -blocks -locations

# Run balancer
hdfs balancer -threshold 5
```

### Python Libraries for Hadoop Ecosystem
| Library | Use Case | Install |
|---------|----------|---------|
| `mrjob` | Write MapReduce in Python, run on Hadoop/EMR | `pip install mrjob` |
| `pyarrow` | Read/write HDFS files | `pip install pyarrow` |
| `hdfscli` | WebHDFS REST API client | `pip install hdfs` |
| `pyspark` | Spark (modern successor to MapReduce) | `pip install pyspark` |
| `snakebite` | Pure Python HDFS client | `pip install snakebite-py3` |

---

## Key Takeaways

1. **HDFS splits files into 128 MB blocks and replicates them 3× across racks** — this gives you both storage scalability and fault tolerance with no single point of data loss.

2. **The NameNode is the brain** — it holds metadata only; actual data flows directly between clients and DataNodes, so it scales.

3. **"Move compute to data"** — MapReduce tasks are scheduled where data already lives, minimizing network I/O on petabyte-scale datasets.

4. **MapReduce = Map + Shuffle/Sort + Reduce** — the framework handles the hard parts (parallelism, fault recovery, sorting); you just write business logic.

5. **Fault tolerance is built-in everywhere** — block replication, task retries, speculative execution, NameNode HA, and checksums make the system resilient to commodity hardware failures.

6. **MapReduce is batch-oriented** — for real-time/streaming use **Apache Spark** (in-memory) or **Apache Flink**. Hadoop/MapReduce is still the foundation for many data lakes.

---
*Notes compiled from Google GFS/MapReduce papers, Apache Hadoop documentation, and practical examples.*