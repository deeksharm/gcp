# Lab 6.1 – Dataproc Serverless (Spark): Load GCS CSV into BigQuery

**Course:** Google Cloud Professional Data Engineer – Day 6
**Mode:** UI steps first, with a CLI option for each step
**Region:** `us-central1`

---

## What we are trying to achieve

We build a small **ETL pipeline using Apache Spark on Dataproc Serverless**:

| Stage | What happens |
|---|---|
| **Extract** | Spark reads a raw CSV file (`orders_1.csv`) from Cloud Storage |
| **Transform** | It casts columns to integers, uppercases `status` (`complete` → `COMPLETE`), and adds a `load_ts` timestamp |
| **Load** | It writes two BigQuery tables: `orders_spark` (clean rows) and `orders_status_summary` (orders and total amount per status) |

**Why Dataproc Serverless?** You run Spark jobs without creating, sizing or deleting a cluster. Google provisions the compute, runs the job and releases the resources when it finishes.

> In the new Console, Dataproc is labelled **Managed Apache Spark**.

```
gs://<PROJECT_ID>-pde-d5/orders/orders_1.csv
              │
              ▼
   Dataproc Serverless (PySpark: orders_etl.py)
              │   (temp files → gs://<PROJECT_ID>-pde-d5)
              ▼
   BigQuery: pde_d5.orders_spark
             pde_d5.orders_status_summary
```

---

## Before you start

- You are logged in to the lab project (see the lab access README).
- Use **Cloud Shell** for the CLI steps (top-right **>_** icon in the Console).
- Use **us-central1** for every resource. The `US` multi-region is blocked in this lab.
- Use the client-provided service account **`nl-sa-…`**. Never delete or modify it.

---

## Step 1 – Set variables (Cloud Shell)

Every participant's values are picked up automatically:

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d5"
export DATASET="pde_d5"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

echo "Project : $PROJECT_ID"
echo "Bucket  : $BUCKET"
echo "SA      : $SA_EMAIL"
```

> If **Project** is empty, run `gcloud config set project <your-project-id>` and run the block again.
> Open a new Cloud Shell tab? Run this block again, because variables don't carry over.

---

## Step 2 – Create the Cloud Storage bucket and BigQuery dataset

### UI

1. **Cloud Storage → Buckets → + Create**
   - Name: `<PROJECT_ID>-pde-d5`
   - Location type: **Region** → **us-central1**
   - Leave the other settings as default → **Create** (if prompted about public access prevention, keep it **enforced**).
2. **BigQuery → Explorer → ⋮ next to your project → Create dataset**
   - Dataset ID: `pde_d5`
   - Location type: **Region** → **us-central1**
   - → **Create dataset**

### CLI alternative

```bash
gcloud storage buckets create gs://$BUCKET --location=$REGION
bq --location=$REGION mk -d "$PROJECT_ID:$DATASET"
```

---

## Step 3 – Create the sample data and the PySpark script (Cloud Shell)

```bash
cat > orders_1.csv <<'EOF'
order_id,customer_id,amount,status
1,101,100,complete
2,102,200,COMPLETE
3,101,50,cancelled
4,103,300,complete
EOF

cat > orders_etl.py <<'EOF'
import sys
from pyspark.sql import SparkSession, functions as F

# Arguments (by position):
#   1) input CSV path       e.g. gs://<PROJECT_ID>-pde-d5/orders/orders_1.csv
#   2) output dataset       e.g. <PROJECT_ID>:pde_d5
#   3) temp bucket (name)   e.g. <PROJECT_ID>-pde-d5
input_path, output_dataset, temp_bucket = sys.argv[1], sys.argv[2], sys.argv[3]

spark = SparkSession.builder.appName("orders-gcs-to-bq").getOrCreate()

# Extract
df = spark.read.option("header", True).option("inferSchema", True).csv(input_path)

# Transform
orders = (df.select(F.col("order_id").cast("int"),
                    F.col("customer_id").cast("int"),
                    F.col("amount").cast("int"),
                    F.upper(F.col("status")).alias("status"))
            .withColumn("load_ts", F.current_timestamp()))

summary = orders.groupBy("status").agg(F.count("*").alias("orders"),
                                       F.sum("amount").alias("total_amount"))

# Load
for table, data in [("orders_spark", orders), ("orders_status_summary", summary)]:
    (data.write.format("bigquery")
         .option("table", f"{output_dataset}.{table}")
         .option("temporaryGcsBucket", temp_bucket)
         .mode("overwrite").save())

spark.stop()
EOF
```

> The mixed-case `status` values are deliberate, so you can see the Spark transform at work.

---

## Step 4 – Upload the files to Cloud Storage

### UI

1. **Cloud Storage → Buckets → `<PROJECT_ID>-pde-d5`**
2. **Create folder** `orders` → open it → **Upload files** → `orders_1.csv`
3. Go back to the bucket root → **Create folder** `scripts` → open it → **Upload files** → `orders_etl.py`

> To upload from your laptop, first download the files from Cloud Shell (**⋮ → Download**). Or use the CLI below.

### CLI alternative

```bash
gcloud storage cp orders_1.csv  gs://$BUCKET/orders/orders_1.csv
gcloud storage cp orders_etl.py gs://$BUCKET/scripts/orders_etl.py
gcloud storage ls -r gs://$BUCKET
```

---

## Step 5 – Check Private Google Access on the subnet

Serverless Spark workers have no external IP, so they reach Cloud Storage and BigQuery through **Private Google Access**.

### UI

**VPC network → VPC networks → `default` → Subnets** tab → **us-central1** → **Edit** → **Private Google Access: On** → **Save**

### CLI alternative

```bash
gcloud compute networks subnets update default --region=$REGION --enable-private-ip-google-access

gcloud compute networks subnets describe default --region=$REGION \
  --format="value(privateIpGoogleAccess)"      # expected: True
```

> If **update** returns *permission denied*, run only **describe**. If it prints `True`, continue.

---

## Step 6 – Submit the Spark batch

### Option A – UI

**Managed Apache Spark → Serverless → Batches → + Create**

**Batch info**

| Field | Value |
|---|---|
| Batch ID | `orders-etl-01` (must be **new** for every run: `-02`, `-03` …) |
| Region | `us-central1` |

**Container**

| Field | Value |
|---|---|
| Batch type | **PySpark** ← select this **first**; the next field changes from *Main class* to *Main python file* |
| Runtime version | **2.1** |
| Main python file | `gs://<PROJECT_ID>-pde-d5/scripts/orders_etl.py` |
| Additional python files | *(leave empty)* |
| Jar files | `gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar` |
| Files / Archives | *(leave empty)* |
| Arguments | add the **3 values below, in this order**, pressing **Enter** after each |

**Arguments (order matters):**

```
gs://<PROJECT_ID>-pde-d5/orders/orders_1.csv
<PROJECT_ID>:pde_d5
<PROJECT_ID>-pde-d5
```

| # | Used by the script as | Format rule |
|---|---|---|
| 1 | Input CSV | Full `gs://` path |
| 2 | Output dataset | `project:dataset`, with a **colon** and an **underscore** (`pde_d5`) |
| 3 | Temp bucket | Bucket **name only**: no `gs://`, **hyphens** (`pde-d5`) |

**Execution configuration**

| Field | Value |
|---|---|
| Service account | `nl-sa-…@<PROJECT_ID>.iam.gserviceaccount.com` |

**Network configuration**

| Field | Value |
|---|---|
| Network | `default` |
| Subnetwork | `default` (us-central1) |

→ **Submit**

> **Jar file and runtime must match:**
>
> | Runtime | Jar files |
> |---|---|
> | 2.1 (Scala 2.12) | `gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar` |
> | 2.2 / 2.3 (Scala 2.13) | `gs://spark-lib/bigquery/spark-bigquery-latest_2.13.jar` |
>
> Runtimes 2.1 and later already include the BigQuery connector, so you can also leave **Jar files** empty.

### Option B – CLI alternative

```bash
gcloud dataproc batches submit pyspark "gs://$BUCKET/scripts/orders_etl.py" \
  --region="$REGION" \
  --batch="orders-etl-$(date +%s)" \
  --version=2.1 \
  --jars=gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar \
  --service-account="$SA_EMAIL" \
  --subnet=default \
  -- "gs://$BUCKET/orders/orders_1.csv" "$PROJECT_ID:$DATASET" "$BUCKET"
```

> Everything after `--` is passed to the script as arguments 1, 2, 3.

---

## Step 7 – Monitor the job

### UI

**Managed Apache Spark → Serverless → Batches** → click your batch

- Status goes **Pending → Running → Succeeded** (about 2–4 minutes; Spark needs ~60 s to get resources).
- The **Output** panel shows the Spark driver log.
- **View Logs** opens Cloud Logging for the batch.

### CLI alternative

```bash
gcloud dataproc batches list --region=$REGION --limit=5
gcloud dataproc batches describe <BATCH_ID> --region=$REGION --format="value(state,stateMessage)"
```

---

## Step 8 – Verify in BigQuery

### UI

**BigQuery → Explorer → `pde_d5`** → open `orders_spark` and `orders_status_summary` → **Preview**.
Or run in the query editor:

```sql
SELECT * FROM pde_d5.orders_spark ORDER BY order_id;

SELECT status, orders, total_amount
FROM pde_d5.orders_status_summary
ORDER BY status;
```

### CLI alternative

```bash
bq --location=$REGION query --use_legacy_sql=false \
  "SELECT status, orders, total_amount FROM $DATASET.orders_status_summary ORDER BY status"
```

### Expected result

`orders_spark`: **4 rows**, `status` all uppercase, `load_ts` filled in.

| status | orders | total_amount |
|---|---|---|
| CANCELLED | 1 | 50 |
| COMPLETE | 3 | 600 |

---

## Step 9 – Fallback if the batch is blocked by `CustomVMrestriction`

If the batch fails with **`customConstraints/custom.CustomVMrestriction`**, the lab provider's org policy is blocking the Spark VMs. Your steps are not the problem. This produces the same tables without Spark:

```bash
bq --location=$REGION load --source_format=CSV --skip_leading_rows=1 --replace \
  "$DATASET.orders_spark" "gs://$BUCKET/orders/orders_1.csv" \
  order_id:INT64,customer_id:INT64,amount:INT64,status:STRING

bq --location=$REGION query --use_legacy_sql=false "
CREATE OR REPLACE TABLE $DATASET.orders_status_summary AS
SELECT UPPER(status) AS status, COUNT(*) AS orders, SUM(amount) AS total_amount
FROM $DATASET.orders_spark
GROUP BY 1"
```

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| Only **Main class** is shown, no *Main python file* | Batch type is **Spark** | Change Batch type to **PySpark** |
| `404 The specified bucket does not exist` on an upload to `…/b/<PROJECT_ID>.pde_d5/…` | Arguments are in the wrong order; the dataset was used as the temp bucket | Click **Clone**, set a new Batch ID, and re-enter the arguments in the order shown in Step 6 |
| `404 bucket does not exist` for `<PROJECT_ID>-pde-d5` | Bucket was deleted or never created | Repeat Steps 2–4 |
| `Batch ID already exists` | Batch IDs must be unique | Use a new Batch ID |
| `ClassNotFoundException` / `NoClassDefFoundError … scala` | The jar's Scala version doesn't match the runtime | Runtime 2.1 → `_2.12` jar; 2.2/2.3 → `_2.13` jar (or leave the jar empty) |
| `Dataset … not found` / location error | Argument 2 is wrong, or the dataset isn't in us-central1 | Use `PROJECT_ID:pde_d5`; create the dataset in us-central1 |
| `Invalid bucket name gs://…` | Argument 3 includes `gs://` | Give the bucket name only |
| `Subnetwork … does not have Private Google Access` | PGA is off | Step 5 |
| `Permission … dataproc.worker` | The default compute service account was used | Select the `nl-sa-…` service account |
| `CustomVMrestriction` | Provider org policy | Step 9 |

---

## Cleanup (lab resources only)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d5"
export DATASET="pde_d5"

# BigQuery tables
bq rm -f "$PROJECT_ID:$DATASET.orders_spark"
bq rm -f "$PROJECT_ID:$DATASET.orders_status_summary"

# Lab files in Cloud Storage
gcloud storage rm -r "gs://$BUCKET/orders" "gs://$BUCKET/scripts"

# Optional – remove the bucket and dataset if no later lab needs them
# gcloud storage rm -r "gs://$BUCKET"
# bq rm -r -f -d "$PROJECT_ID:$DATASET"
```

**UI alternative:** In **BigQuery**, open each table → **Delete**. In **Cloud Storage**, tick the folders or bucket → **Delete**.

> Do **not** delete or modify the `nl-sa-…` service account, the project, IAM bindings, `dataproc-staging-*` / `dataproc-temp-*` buckets created by the provider, or datasets from other labs.

---

## Key takeaways

- **Dataproc Serverless** runs Spark batches without managing clusters: you submit code and Google handles the compute.
- A PySpark batch needs the **main Python file**, the **runtime**, optional **jars**, and **arguments** passed by position.
- The **Spark BigQuery connector** writes DataFrames to BigQuery using a **temporary GCS bucket**.
- Serverless Spark needs **Private Google Access** on its subnet and a service account with **Dataproc Worker** permissions.

## References

- Dataproc Serverless overview: https://cloud.google.com/dataproc-serverless/docs/overview
- Submit a Spark batch workload: https://cloud.google.com/dataproc-serverless/docs/quickstarts/spark-batch
- Serverless runtime versions: https://cloud.google.com/dataproc-serverless/docs/concepts/versions/spark-runtime-versions
- Spark BigQuery connector: https://github.com/GoogleCloudDataproc/spark-bigquery-connector

---
*Trainer: Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · deekshasharma191196@gmail.com*
