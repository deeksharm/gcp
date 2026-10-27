# Lab 6.3 – Dataflow Batch Pipelines (UI-first)

**Course:** Google Cloud Professional Data Engineer – Day 6
**Mode:** UI steps first, with a CLI option for each step
**Region:** `us-central1`

| Part | Official source | Pipeline |
|---|---|---|
| **A** | Dataflow: Qwik Start – Python (UI template version) | Word Count: GCS text → GCS files |
| **B** | Monitoring, Logging and Error Reporting for Dataflow Jobs | Job graph, metrics and logs of a batch job |
| **C** *(optional)* | Dataflow Job Builder | GCS CSV → Filter → BigQuery (visual, no code) |

---

## What we are trying to achieve

- Run a **batch** pipeline on Dataflow without writing code, using a Google-provided template.
- Understand the Apache Beam flow: **Read → Transform → Write**.
- Learn how a batch job behaves: workers start, process a **bounded** dataset, scale down, and the job ends as **Succeeded**.
- Read the **job graph, metrics and logs** of a Dataflow job.

> **Dataflow in one line:** a fully managed, autoscaling service that runs **Apache Beam** pipelines. A batch job processes a fixed (bounded) dataset and then stops.

---

## Before you start

- You are logged in to the lab project (see the lab access README).
- Use **Cloud Shell** (top-right **>_** icon) for the CLI steps.
- Use **us-central1** for everything.
- Use the client-provided service account **`nl-sa-…`**. Never delete or modify it.

---

## Step 0 – Setup

### 0.1 Variables (Cloud Shell)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-df-batch"
export DATASET="pde_df_batch"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

echo "Project : $PROJECT_ID"
echo "Bucket  : $BUCKET"
echo "SA      : $SA_EMAIL"
```

> If **Project** is empty, run `gcloud config set project <your-project-id>` and run the block again.

### 0.2 Enable the Dataflow API

**UI:** **APIs & Services → Library →** search **Dataflow API** → **Enable** (skip this if it's already enabled).

**CLI:**
```bash
gcloud services enable dataflow.googleapis.com
```

### 0.3 Create the bucket and dataset

**UI:**
1. **Cloud Storage → Buckets → + Create**: `<PROJECT_ID>-pde-df-batch`, Location type **Region**, **us-central1** → **Create**.
2. **BigQuery → ⋮ next to project → Create dataset**: `pde_df_batch`, Region **us-central1** → **Create dataset**.

**CLI:**
```bash
gcloud storage buckets create gs://$BUCKET --location=$REGION
bq --location=$REGION mk -d "$PROJECT_ID:$DATASET"
```

### 0.4 Check Private Google Access

Dataflow workers run with private IPs, so they need Private Google Access to reach Cloud Storage and BigQuery.

**UI:** **VPC network → VPC networks → `default` → Subnets → us-central1** → Private Google Access should be **On**.

**CLI:**
```bash
gcloud compute networks subnets describe default --region=$REGION \
  --format="value(privateIpGoogleAccess)"        # expected: True
```

### 0.5 Common settings for every Dataflow job (UI → Optional parameters)

| Optional parameter | Value |
|---|---|
| Temporary location | `gs://<PROJECT_ID>-pde-df-batch/temp` |
| Service account email | `nl-sa-…@<PROJECT_ID>.iam.gserviceaccount.com` |
| Subnetwork | `regions/us-central1/subnetworks/default` |
| Worker IP address configuration | **Private** (or untick *Use public IPs*) |
| Max workers | `2` |

---

## Part A – Word Count template (GCS → GCS)

**Goal:** Count how often each word appears in Shakespeare's *King Lear* (public file `gs://dataflow-samples/shakespeare/kinglear.txt`).

```
gs://dataflow-samples/shakespeare/kinglear.txt
        │  ReadLines
        ▼
   Split into words  →  Count per word  →  Format "word: count"
        │  WriteCounts
        ▼
gs://<PROJECT_ID>-pde-df-batch/wordcount/output-*
```

### A.1 Run the job (UI)

**Dataflow → Jobs → + Create job from template**

| Field | Value |
|---|---|
| Job name | `wordcount-01` |
| Regional endpoint | `us-central1` |
| Dataflow template | **Word Count** |
| Input file(s) in Cloud Storage | `gs://dataflow-samples/shakespeare/kinglear.txt` |
| Output Cloud Storage file prefix | `gs://<PROJECT_ID>-pde-df-batch/wordcount/output` |
| Optional parameters | as in Step 0.5 |

→ **Run job**

### A.1 CLI alternative

```bash
gcloud dataflow jobs run "wordcount-$(date +%s)" \
  --gcs-location="gs://dataflow-templates-$REGION/latest/Word_Count" \
  --region="$REGION" \
  --staging-location="gs://$BUCKET/temp" \
  --service-account-email="$SA_EMAIL" \
  --subnetwork="regions/$REGION/subnetworks/default" \
  --disable-public-ips \
  --max-workers=2 \
  --parameters="inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,output=gs://$BUCKET/wordcount/output"
```

### A.2 Watch it run

**Dataflow → Jobs →** `wordcount-…`

- Status goes **Starting → Running → Succeeded** (about 4–6 minutes, mostly worker startup).
- In the CLI list the final state shows as **Done**.

**CLI:**
```bash
gcloud dataflow jobs list --region=$REGION --limit=5 --format="table(name,type,state,creationTime)"
```

### A.3 Verify the output

**UI:** **Cloud Storage → `<PROJECT_ID>-pde-df-batch` → wordcount/** contains `output-00000-of-0000N` files. Open one and **Download**.

**CLI:**
```bash
gcloud storage ls gs://$BUCKET/wordcount/
gcloud storage cat "gs://$BUCKET/wordcount/output-*" | head -20
```

**Expected:** one `word: count` per line, e.g. `KING: …`, `LEAR: …`.

---

## Part B – Monitor the batch job

Open **Dataflow → Jobs →** `wordcount-…`.

### B.1 Job graph

- Steps: **ReadLines → WordCount.CountWords → MapElements → WriteCounts**.
- Click a step. The side panel shows **Elements added**, **Estimated size** and **Wall time**.
- **Job info** panel (right): region, worker machine type, service account, SDK version, start/end time.

### B.2 Job metrics

Open the **Job metrics** tab:

| Metric | What to observe in a batch job |
|---|---|
| **Autoscaling** | Workers scale up to process the data, then down to 0 at the end |
| **Throughput** | Elements per second for each step |
| **CPU utilization** | Worker load while the job runs |

### B.3 Logs

- At the bottom of the job page, open **Logs → Job logs**: worker startup, autoscaling decisions, "Workflow completed".
- **Worker logs**: messages from the processing workers.
- **Diagnostics** tab: grouped errors and recommendations (empty for a healthy job).

**CLI:**
```bash
JOB_ID=$(gcloud dataflow jobs list --region=$REGION --limit=1 \
  --filter="name~wordcount" --format="value(id)")
echo "JOB_ID=$JOB_ID"

gcloud logging read "resource.type=\"dataflow_step\" AND resource.labels.job_id=\"$JOB_ID\"" \
  --limit=10 --format="table(timestamp,severity,textPayload)"
```

> If `JOB_ID` is empty, run `gcloud dataflow jobs list --region=$REGION` and paste the ID manually.

### B.4 Batch vs streaming (discussion)

| | Batch | Streaming |
|---|---|---|
| Input | Bounded (files, tables) | Unbounded (Pub/Sub) |
| Job ends | Automatically, **Succeeded / Done** | Never on its own; you **Drain** or **Cancel** |
| Typical use | Nightly ETL, backfills, reports | Real-time dashboards, alerts, CDC |

---

## Part C (optional) – Dataflow Job Builder: visual pipeline

**Goal:** Build a pipeline visually, without code: **orders CSV in GCS → keep only COMPLETE orders → BigQuery**.

> ⚠️ Job Builder starts a **launcher VM** first. In this lab environment the launcher VM can be blocked by the org policy `CustomVMrestriction`. If no job appears or it fails with that error, treat this part as a walkthrough.

### C.1 Upload the input data and create the target table

```bash
cat > orders.csv <<'EOF'
order_id,customer_id,amount,status
1,101,100,COMPLETE
2,102,200,COMPLETE
3,101,50,CANCELLED
4,103,300,COMPLETE
EOF
gcloud storage cp orders.csv gs://$BUCKET/builder/orders.csv
```

**BigQuery query editor:**
```sql
CREATE TABLE IF NOT EXISTS pde_df_batch.orders_complete (
  order_id INT64, customer_id INT64, amount INT64, status STRING
);
```

### C.2 Build and run the job (UI)

**Dataflow → Jobs → + Create job from builder**

1. **Job name:** `builder-orders-01`. **Job type:** **Batch**.
2. **Sources → Add a source**
   - Source type: **Cloud Storage**, format **CSV**
   - File location: `gs://<PROJECT_ID>-pde-df-batch/builder/orders.csv`
   - Schema: `order_id` INT64, `customer_id` INT64, `amount` INT64, `status` STRING (skip the header row)
3. **Transforms → Add a transform**: **Filter**, condition `status == 'COMPLETE'`
4. **Sinks → Add a sink**: **BigQuery**
   - Table: `<PROJECT_ID>:pde_df_batch.orders_complete`
   - Write disposition: **Append**
5. **Optional parameters:** as in Step 0.5
6. Check that no fields are marked red → **Run job**. The page opens the new job, so note its name.

### C.3 Verify

```sql
SELECT * FROM pde_df_batch.orders_complete ORDER BY order_id;
```

**Expected:** 3 rows (order_id **1, 2, 4**), all `COMPLETE`.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `CustomVMrestriction` in job logs | Provider org policy blocks the worker or launcher VM | Report it to the trainer. Use that part as a walkthrough |
| `Subnetwork … does not have Private Google Access` | PGA is off | Step 0.4 |
| `permission … dataflow.worker` | The default compute service account was used | Set **Service account email** = `nl-sa-…` |
| `Location US is restricted` | Job or dataset not in us-central1 | Regional endpoint **us-central1**; dataset in us-central1 |
| No `wordcount/output-*` files | Job still running or failed | Wait for **Succeeded**; check **Logs → Job logs** |
| `orders_complete was not found` | The Job Builder job never ran or failed | `gcloud dataflow jobs list --region=$REGION`; check the job's logs |
| `Unparseable filter` in `gcloud logging read` | `JOB_ID` is empty | Paste the job ID manually |

---

## Cleanup (lab resources only)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-df-batch"
export DATASET="pde_df_batch"

bq rm -r -f -d "$PROJECT_ID:$DATASET"
gcloud storage rm -r "gs://$BUCKET"
rm -f orders.csv
```

**UI alternative:** **BigQuery** → delete dataset `pde_df_batch`. **Cloud Storage** → delete the bucket `<PROJECT_ID>-pde-df-batch`.

> Batch jobs stop on their own, so there are no jobs to stop. Never delete the `nl-sa-…` service account, the project, IAM bindings, or resources from other labs.

---

## Key takeaways

- Google-provided **templates** let you run Dataflow pipelines with only parameters, no code.
- A **batch** job reads a bounded dataset, autoscales, and finishes on its own.
- The **job graph, metrics and logs** are the first places to look when you troubleshoot a pipeline.
- **Job Builder** creates Beam pipelines visually from sources, transforms and sinks.

## References

- Dataflow overview: https://cloud.google.com/dataflow/docs/overview
- Google-provided templates (Word Count): https://cloud.google.com/dataflow/docs/guides/templates/provided-templates
- Dataflow monitoring interface: https://cloud.google.com/dataflow/docs/guides/monitoring-overview
- Dataflow Job Builder: https://cloud.google.com/dataflow/docs/guides/job-builder

---
*Trainer: Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · deekshasharma191196@gmail.com*
