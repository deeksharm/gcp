# Lab 6.2 – Running Spark Jobs on Dataproc

**Course:** Google Cloud Professional Data Engineer – Day 6
**Mode:** UI steps first, with a CLI option for each step
**Region:** `us-central1`

This lab merges four official labs into one hands-on session:

| Part | Official lab | What you do |
|---|---|---|
| **A** | Dataproc: Qwik Start – Console (GSP103) | Submit the SparkPi job from the UI |
| **B** | Dataproc: Qwik Start – Command Line (GSP104) | Submit the same job with `gcloud` |
| **C** | Prepare Data for ML APIs on Google Cloud – *Run a simple Dataproc job* (GSP323) | Run SparkPageRank on a data file in Cloud Storage |
| **D** | Building Batch Data Pipelines – *Running Apache Spark jobs on Dataproc* | Lift and shift a Hadoop Spark job: HDFS → Cloud Storage |

> **Lab environment note:** The official labs create a Dataproc **cluster**. In this environment the org policy `CustomVMrestriction` blocks cluster creation, so every part runs as a **Dataproc Serverless batch**. The Spark code, jobs and concepts are the same. Cluster topics (sizing, scaling, ephemeral clusters) are covered in the discussion at the end.

> In the new Console, Dataproc is labelled **Managed Apache Spark**.

---

## What we are trying to achieve

- Understand the **Dataproc job lifecycle**: submit → run → monitor → read output.
- Run both kinds of Spark job: a **Spark jar job** (Java/Scala, Parts A–C) and a **PySpark job** (Part D).
- Learn the core Dataproc migration pattern: **keep data in Cloud Storage instead of HDFS**, so compute can be job-scoped and storage outlives the job.

---

## Before you start

- You are logged in to the lab project (see the lab access README).
- Use **Cloud Shell** (top-right **>_** icon) for the CLI steps.
- Use **us-central1** for everything.
- Use the client-provided service account **`nl-sa-…`**. Never delete or modify it.

---

## Step 0 – Setup (once for the whole lab)

### 0.1 Set variables (Cloud Shell)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d6"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

echo "Project : $PROJECT_ID"
echo "Bucket  : $BUCKET"
echo "SA      : $SA_EMAIL"
```

> If **Project** is empty, run `gcloud config set project <your-project-id>` and run the block again.
> Open a new Cloud Shell tab? Run this block again.

### 0.2 Create the lab bucket

**UI:** **Cloud Storage → Buckets → + Create** → Name `<PROJECT_ID>-pde-d6` → Location type **Region** → **us-central1** → **Create**

**CLI alternative:**
```bash
gcloud storage buckets create gs://$BUCKET --location=$REGION
```

### 0.3 Check Private Google Access

Serverless Spark workers have no external IP, so they need Private Google Access to reach Cloud Storage.

**UI:** **VPC network → VPC networks → `default` → Subnets → us-central1 → Edit → Private Google Access: On → Save**

**CLI alternative:**
```bash
gcloud compute networks subnets describe default --region=$REGION \
  --format="value(privateIpGoogleAccess)"      # expected: True
# if False:
gcloud compute networks subnets update default --region=$REGION --enable-private-ip-google-access
```

---

## Part A – Dataproc Qwik Start (Console): SparkPi

**Goal:** Submit Google's built-in Spark example, which estimates π with a Monte Carlo method, and read the result.

### A.1 Create the batch (UI)

**Managed Apache Spark → Serverless → Batches → + Create**

| Field | Value |
|---|---|
| Batch ID | `sparkpi-ui-01` (must be new each run) |
| Region | `us-central1` |
| Batch type | **Spark** |
| Runtime version | **2.3 LTS** |
| Main class or jar | `org.apache.spark.examples.SparkPi` |
| Jar files | `file:///usr/lib/spark/examples/jars/spark-examples.jar` |
| Arguments | `1000` |
| Service account | `nl-sa-…@<PROJECT_ID>.iam.gserviceaccount.com` |
| Network / Subnetwork | `default` / `default` |

→ **Submit**

> - `file:///usr/lib/...` is a jar already inside the Spark runtime image, so you don't need to upload anything.
> - `1000` is the number of tasks (partitions). More tasks give a more accurate estimate.

### A.2 Monitor and verify

1. **Batches** → click `sparkpi-ui-01` → status goes **Pending → Running → Succeeded** (about 1–3 minutes).
2. In the **Output** panel, find:
   ```
   Pi is roughly 3.14...
   ```
3. Click **View Spark UI** to see the job, its stage and the 1000 tasks.
4. Open the **Details** tab to see the runtime, service account and network.

---

## Part B – Dataproc Qwik Start (Command Line): SparkPi with gcloud

**Goal:** Run the same job from the command line. This is how jobs are automated in scripts, CI/CD pipelines and Composer.

### B.1 Submit

```bash
gcloud dataproc batches submit spark \
  --region="$REGION" \
  --batch="sparkpi-cli-$(date +%s)" \
  --version=2.3 \
  --class=org.apache.spark.examples.SparkPi \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  --service-account="$SA_EMAIL" \
  --subnet=default \
  -- 1000
```

> `gcloud` streams the driver output into Cloud Shell. Look for `Pi is roughly 3.14…`.

### B.2 List and inspect batches

```bash
gcloud dataproc batches list --region=$REGION --limit=5

# replace <BATCH_ID> with an ID from the list
gcloud dataproc batches describe <BATCH_ID> --region=$REGION \
  --format="value(state,runtimeConfig.version,environmentConfig.executionConfig.serviceAccount)"
```

**UI check:** The CLI batch also appears in **Managed Apache Spark → Serverless → Batches**.

| Console field | gcloud flag |
|---|---|
| Batch type = Spark | `batches submit spark` |
| Main class | `--class` |
| Jar files | `--jars` |
| Runtime version | `--version` |
| Arguments | everything after `--` |

---

## Part C – Prepare Data for ML APIs: Run a Spark job on a data file

**Goal:** In the official skill badge, a Dataproc Spark job processes a data file as one step of a data-preparation pipeline. We run the built-in **SparkPageRank** example, which ranks pages by their incoming links, on a link file stored in Cloud Storage.

> In the official lab, the file is copied into the cluster's **HDFS**. With Serverless there is no HDFS, so the job reads it straight from **Cloud Storage** (`gs://`).

### C.1 Create the input file

Each line is `<page> <page it links to>`:

```bash
cat > data.txt <<'EOF'
url_1 url_4
url_2 url_1
url_3 url_2
url_3 url_1
url_4 url_3
url_4 url_1
EOF

gcloud storage cp data.txt gs://$BUCKET/pagerank/data.txt
gcloud storage cat gs://$BUCKET/pagerank/data.txt
```

> **Optional, official file:** `gcloud storage cp gs://cloud-training/gsp323/data.txt gs://$BUCKET/pagerank/data.txt`. If you get access denied, use the sample above.

**UI alternative:** Create `data.txt` on your laptop with the lines above, then go to **Bucket → Create folder `pagerank` → Upload files**.

### C.2 Submit the job (UI)

**Managed Apache Spark → Serverless → Batches → + Create**

| Field | Value |
|---|---|
| Batch ID | `pagerank-01` |
| Region | `us-central1` |
| Batch type | **Spark** |
| Runtime version | **2.3 LTS** |
| Main class or jar | `org.apache.spark.examples.SparkPageRank` |
| Jar files | `file:///usr/lib/spark/examples/jars/spark-examples.jar` |
| Arguments (2 chips) | `gs://<PROJECT_ID>-pde-d6/pagerank/data.txt` → `10` |
| Service account | `nl-sa-…` |
| Network / Subnetwork | `default` / `default` |

→ **Submit**

> Argument 1 is the input file. Argument 2 is the number of PageRank iterations.

### C.2 CLI alternative

```bash
gcloud dataproc batches submit spark \
  --region="$REGION" \
  --batch="pagerank-$(date +%s)" \
  --version=2.3 \
  --class=org.apache.spark.examples.SparkPageRank \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  --service-account="$SA_EMAIL" \
  --subnet=default \
  -- "gs://$BUCKET/pagerank/data.txt" 10
```

### C.3 Verify

In the batch **Output** (or the Cloud Shell output), look for lines like:

```
url_1 has rank: 1.7...
url_4 has rank: 0.7...
url_2 has rank: 0.7...
url_3 has rank: 0.7...
```

**`url_1` gets the highest rank** because three other pages link to it. Exact decimals can vary slightly.

---

## Part D – Running Apache Spark jobs on Dataproc: lift and shift (HDFS → Cloud Storage)

**Goal:** Follow the official migration story. A company runs a Spark analysis on on-prem Hadoop that reads from **HDFS**. To move it to Google Cloud:

1. Move the data to **Cloud Storage**, which replaces HDFS.
2. Change **only the file path** in the code: `hdfs://…` → `gs://…`.
3. Run the same Spark code as a **Dataproc job** and write the report to Cloud Storage.

**Dataset:** KDD Cup 99 network connection logs (10% sample, 494,021 rows). We analyse connections per protocol and attack vs normal traffic.

### D.1 Copy the data into Cloud Storage (the "migrate from HDFS" step)

```bash
wget https://storage.googleapis.com/cloud-training/dataengineering/lab_assets/sparklab/kddcup.data_10_percent.gz
# if the URL above fails:
# wget http://kdd.ics.uci.edu/databases/kddcup99/kddcup.data_10_percent.gz

gcloud storage cp kddcup.data_10_percent.gz gs://$BUCKET/kdd/
gcloud storage ls -l gs://$BUCKET/kdd/
```

**UI alternative:** Download the file to your laptop, then go to **Bucket → Create folder `kdd` → Upload files**.

### D.2 Create the Spark job

On-prem, this code read `hdfs:///kddcup.data_10_percent.gz`. The only migration change is that the bucket is passed in and the path becomes `gs://`.

```bash
cat > spark_analysis.py <<'EOF'
import argparse
from pyspark.sql import SparkSession, Row

p = argparse.ArgumentParser()
p.add_argument("--bucket", required=True)          # bucket name only (no gs://)
a = p.parse_args()

spark = SparkSession.builder.appName("kdd-lift-and-shift").getOrCreate()
sc = spark.sparkContext

# On-prem:  data_file = "hdfs:///kddcup.data_10_percent.gz"
data_file = f"gs://{a.bucket}/kdd/kddcup.data_10_percent.gz"

raw = sc.textFile(data_file).map(lambda line: line.split(","))
rows = raw.map(lambda r: Row(
    duration=int(r[0]), protocol_type=r[1], service=r[2], flag=r[3],
    src_bytes=int(r[4]), dst_bytes=int(r[5]),
    num_failed_logins=int(r[10]), num_compromised=int(r[12]),
    label=r[-1]))
df = spark.createDataFrame(rows)
df.createOrReplaceTempView("connections")

# Analysis 1 – connections per protocol
by_protocol = spark.sql("""
  SELECT protocol_type, COUNT(*) AS connections
  FROM connections
  GROUP BY protocol_type
  ORDER BY connections DESC""")
by_protocol.show()

# Analysis 2 – attack vs normal traffic per protocol
attack_stats = spark.sql("""
  SELECT protocol_type,
         CASE label WHEN 'normal.' THEN 'no attack' ELSE 'attack' END AS state,
         COUNT(*)                 AS total_freq,
         ROUND(AVG(src_bytes), 2) AS mean_src_bytes,
         ROUND(AVG(dst_bytes), 2) AS mean_dst_bytes,
         ROUND(AVG(duration), 2)  AS mean_duration,
         SUM(num_failed_logins)   AS total_failed_logins,
         SUM(num_compromised)     AS total_compromised
  FROM connections
  GROUP BY protocol_type, state
  ORDER BY total_freq DESC""")
attack_stats.show()

# Write reports to Cloud Storage (replaces writing back to HDFS)
out = f"gs://{a.bucket}/sparktodp"
by_protocol.coalesce(1).write.mode("overwrite").option("header", True).csv(f"{out}/connections_by_protocol")
attack_stats.coalesce(1).write.mode("overwrite").option("header", True).csv(f"{out}/attack_stats")

spark.stop()
EOF

gcloud storage cp spark_analysis.py gs://$BUCKET/scripts/spark_analysis.py
```

> This script uses a **named argument** (`--bucket`), so it can't get mixed up by argument order.

### D.3 Submit the job (UI)

**Managed Apache Spark → Serverless → Batches → + Create**

| Field | Value |
|---|---|
| Batch ID | `kdd-analysis-01` |
| Region | `us-central1` |
| Batch type | **PySpark** |
| Runtime version | **2.3 LTS** |
| Main python file | `gs://<PROJECT_ID>-pde-d6/scripts/spark_analysis.py` |
| Jar files | *(leave empty)* |
| Arguments (2 chips) | `--bucket` → `<PROJECT_ID>-pde-d6` |
| Service account | `nl-sa-…` |
| Network / Subnetwork | `default` / `default` |

→ **Submit**

> You can also enter the argument as a single chip: `--bucket=<PROJECT_ID>-pde-d6`.

### D.3 CLI alternative

```bash
gcloud dataproc batches submit pyspark "gs://$BUCKET/scripts/spark_analysis.py" \
  --region="$REGION" \
  --batch="kdd-analysis-$(date +%s)" \
  --version=2.3 \
  --service-account="$SA_EMAIL" \
  --subnet=default \
  -- --bucket="$BUCKET"
```

### D.4 Verify

**Batch Output panel:** The two `show()` tables are printed in the driver log.

**Cloud Storage (UI):** **Bucket → `sparktodp/`** contains `connections_by_protocol/` and `attack_stats/`, each with a `part-….csv` file.

**CLI:**
```bash
gcloud storage ls -r gs://$BUCKET/sparktodp/
gcloud storage cat "gs://$BUCKET/sparktodp/connections_by_protocol/part-*.csv"
gcloud storage cat "gs://$BUCKET/sparktodp/attack_stats/part-*.csv"
```

**Expected `connections_by_protocol`:**

| protocol_type | connections |
|---|---|
| icmp | 283602 |
| tcp | 190065 |
| udp | 20354 |

In **attack_stats**, most **ICMP** traffic is labelled `attack` (smurf DoS), and most **UDP** traffic is `no attack`.

---

## Discussion – Dataproc concepts (from the official labs)

| On-prem Hadoop | Dataproc on Google Cloud |
|---|---|
| Data in **HDFS** on cluster disks | Data in **Cloud Storage**: outlives the job and is shared by Spark, BigQuery and Dataflow |
| Cluster runs 24×7 | **Job-scoped compute**: Serverless batches, or **ephemeral clusters** deleted after the job |
| Code reads `hdfs://` | Code reads `gs://`, and nothing else changes |
| Manual cluster sizing | Serverless autoscaling; on clusters, autoscaling policies and **secondary (Spot/preemptible) workers** |

**Serverless vs cluster: when to use which (PDE exam angle)**

| Use **Dataproc Serverless** | Use a **Dataproc cluster** |
|---|---|
| Batch Spark jobs, no ops overhead | Need Hadoop ecosystem tools (Hive, Pig, HBase, Presto) |
| Unpredictable or occasional workloads | Long-running or interactive workloads that need fine-grained tuning |
| Pay only while the job runs | Custom images, init actions, specific machine types/GPUs |

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| Field shows *Main python file* instead of *Main class* | Batch type is PySpark | Parts A–C need **Spark**; Part D needs **PySpark** |
| `ClassNotFoundException: org.apache.spark.examples…` | Jar path is wrong | `file:///usr/lib/spark/examples/jars/spark-examples.jar` (three slashes) |
| `Batch ID already exists` | IDs must be unique | Use a new Batch ID |
| `FileNotFoundException … data.txt` / `kddcup…` | File isn't in the bucket | Re-run the upload step; check with `gcloud storage ls -r gs://$BUCKET` |
| `the following arguments are required: --bucket` | Argument missing | Add `--bucket` and the bucket name (or `--bucket=<name>`) |
| `404 The specified bucket does not exist` | Bucket name contains `gs://`, or the bucket was deleted | Use the name only; re-create it with Step 0.2 |
| `Subnetwork … Private Google Access` | PGA is off | Step 0.3 |
| `Permission … dataproc.worker` | The default compute service account was used | Select the `nl-sa-…` service account |
| `CustomVMrestriction` | Provider org policy blocks the VMs | Report it to the trainer (provider-side policy) |

---

## Cleanup (lab resources only)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d6"

# Lab bucket and its contents (data.txt, KDD data, scripts, reports)
gcloud storage rm -r "gs://$BUCKET"

# Local files in Cloud Shell
rm -f data.txt kddcup.data_10_percent.gz spark_analysis.py

# Optional – delete batch records (finished batches use no compute)
for b in $(gcloud dataproc batches list --region=$REGION --format="value(name.basename())" \
           --filter="name~sparkpi OR name~pagerank OR name~kdd-analysis"); do
  gcloud dataproc batches delete "$b" --region=$REGION --quiet
done
```

**UI alternative:** **Cloud Storage → tick `<PROJECT_ID>-pde-d6` → Delete**. Under **Managed Apache Spark → Batches**, tick the lab batches and click **Delete**.

> Do **not** delete the `nl-sa-…` service account, the project, IAM bindings, `dataproc-staging-*` / `dataproc-temp-*` buckets, or resources from other labs.

---

## Key takeaways

- Dataproc runs **Spark jar jobs** (main class + jar) and **PySpark jobs** (main Python file), from the Console or `gcloud`.
- **Cloud Storage replaces HDFS**. Migrating a Spark job is mostly a path change (`hdfs://` → `gs://`).
- **Serverless batches** give job-scoped compute with no cluster to manage. Clusters remain the choice for the full Hadoop ecosystem and fine-grained control.
- Monitor jobs with the batch **Output**, **Spark UI**, **Details/Efficiency** tabs and **Cloud Logging**.

## References

- Dataproc: Qwik Start – Console: https://www.cloudskillsboost.google/focuses/585
- Dataproc: Qwik Start – Command Line: https://www.cloudskillsboost.google/focuses/583
- Prepare Data for ML APIs on Google Cloud (skill badge): https://www.cloudskillsboost.google/course_templates/631
- Building Batch Data Pipelines on Google Cloud: https://www.cloudskillsboost.google/course_templates/53
- Dataproc Serverless – submit a Spark batch: https://cloud.google.com/dataproc-serverless/docs/quickstarts/spark-batch
- Migrating HDFS data to Cloud Storage: https://cloud.google.com/architecture/hadoop/hadoop-gcp-migration-data

---
*Trainer: Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · deekshasharma191196@gmail.com*
