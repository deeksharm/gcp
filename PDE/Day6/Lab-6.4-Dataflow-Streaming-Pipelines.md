# Lab 6.4 – Dataflow Streaming Pipelines (UI-first)

**Course:** Google Cloud Professional Data Engineer – Day 6
**Mode:** UI steps first, with a CLI option for each step
**Region:** `us-central1`

| Part | Official source | Pipeline |
|---|---|---|
| **A** | Dataflow: Qwik Start – Templates (GSP192) | Pub/Sub (live taxi rides) → BigQuery |
| **B** | Google-provided streaming template | Pub/Sub (your own events) → Cloud Storage text files (windowed) |
| **C** | Monitoring, Logging and Error Reporting for Dataflow Jobs | Monitor the streaming jobs, then **Drain vs Cancel** |

---

## What we are trying to achieve

- Run **streaming** pipelines on Dataflow from Google-provided templates, with no code.
- See how **Pub/Sub + Dataflow** feeds both a **warehouse** (BigQuery) and a **data lake** (Cloud Storage).
- Understand **windowing**: how an endless stream becomes files.
- **Operate** streaming jobs: read metrics such as data freshness and system latency, and stop jobs safely.

```
Part A:  projects/pubsub-public-data/topics/taxirides-realtime ──► Dataflow ──► BigQuery pde_df_stream.taxi_realtime
Part B:  your topic pde-df-events (you publish) ──► Dataflow (5-min windows) ──► gs://<bucket>/events/*.txt
```

> **Streaming jobs never finish on their own, and they bill until you stop them.** Always do Part C (Drain or Cancel) before you leave.

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
export BUCKET="${PROJECT_ID}-pde-df-stream"
export DATASET="pde_df_stream"
export TOPIC="pde-df-events"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

echo "Project : $PROJECT_ID"
echo "Bucket  : $BUCKET"
echo "SA      : $SA_EMAIL"
```

> If **Project** is empty, run `gcloud config set project <your-project-id>` and run the block again.
> Open a new Cloud Shell tab? Run this block again.

### 0.2 Enable APIs

**UI:** **APIs & Services → Library →** enable **Dataflow API** and **Cloud Pub/Sub API**.

**CLI:**
```bash
gcloud services enable dataflow.googleapis.com pubsub.googleapis.com
```

### 0.3 Create the bucket, dataset and Pub/Sub topic

**UI:**
1. **Cloud Storage → Buckets → + Create**: `<PROJECT_ID>-pde-df-stream`, Region **us-central1** → **Create**.
2. **BigQuery → ⋮ next to project → Create dataset**: `pde_df_stream`, Region **us-central1** → **Create dataset**.
3. **Pub/Sub → Topics → + Create topic**: Topic ID `pde-df-events`. Untick **Add a default subscription** → **Create**.

**CLI:**
```bash
gcloud storage buckets create gs://$BUCKET --location=$REGION
bq --location=$REGION mk -d "$PROJECT_ID:$DATASET"
gcloud pubsub topics create $TOPIC
```

### 0.4 Check Private Google Access

**UI:** **VPC network → VPC networks → `default` → Subnets → us-central1** → Private Google Access should be **On**.

**CLI:**
```bash
gcloud compute networks subnets describe default --region=$REGION \
  --format="value(privateIpGoogleAccess)"        # expected: True
```

### 0.5 Common settings for every Dataflow job (UI → Optional parameters)

| Optional parameter | Value |
|---|---|
| Temporary location | `gs://<PROJECT_ID>-pde-df-stream/temp` |
| Service account email | `nl-sa-…@<PROJECT_ID>.iam.gserviceaccount.com` |
| Subnetwork | `regions/us-central1/subnetworks/default` |
| Worker IP address configuration | **Private** (or untick *Use public IPs*) |
| Max workers | `2` |

---

## Part A – Pub/Sub → BigQuery (Dataflow Qwik Start – Templates)

**Goal:** Stream live NYC taxi ride events from Google's **public Pub/Sub topic** into BigQuery in real time.

### A.1 Create the target table

**UI:** **BigQuery → query editor** → run:

```sql
CREATE TABLE pde_df_stream.taxi_realtime (
  ride_id STRING,
  point_idx INT64,
  latitude FLOAT64,
  longitude FLOAT64,
  timestamp TIMESTAMP,
  meter_reading FLOAT64,
  meter_increment FLOAT64,
  ride_status STRING,
  passenger_count INT64
);
```

**CLI:**
```bash
bq mk --table "$PROJECT_ID:$DATASET.taxi_realtime" \
  ride_id:STRING,point_idx:INTEGER,latitude:FLOAT,longitude:FLOAT,timestamp:TIMESTAMP,meter_reading:FLOAT,meter_increment:FLOAT,ride_status:STRING,passenger_count:INTEGER
```

### A.2 Run the streaming job (UI)

**Dataflow → Jobs → + Create job from template**

| Field | Value |
|---|---|
| Job name | `taxi-stream-01` |
| Regional endpoint | `us-central1` |
| Dataflow template | **Pub/Sub Topic to BigQuery** (search "Pub/Sub to BigQuery") |
| Input Pub/Sub topic | `projects/pubsub-public-data/topics/taxirides-realtime` |
| BigQuery output table | `<PROJECT_ID>:pde_df_stream.taxi_realtime` |
| Optional parameters | as in Step 0.5 |

→ **Run job**

> If the Console only offers **Pub/Sub Subscription to BigQuery**, pick the variant that takes a **topic**, or use the CLI.

### A.2 CLI alternative

```bash
gcloud dataflow jobs run "taxi-stream-$(date +%s)" \
  --gcs-location="gs://dataflow-templates-$REGION/latest/PubSub_to_BigQuery" \
  --region="$REGION" \
  --staging-location="gs://$BUCKET/temp" \
  --service-account-email="$SA_EMAIL" \
  --subnetwork="regions/$REGION/subnetworks/default" \
  --disable-public-ips \
  --max-workers=2 \
  --parameters="inputTopic=projects/pubsub-public-data/topics/taxirides-realtime,outputTableSpec=$PROJECT_ID:$DATASET.taxi_realtime"
```

### A.3 Confirm the job is running

**UI:** **Dataflow → Jobs** → `taxi-stream-…` shows **Running**, Type **Streaming**.

**CLI:**
```bash
gcloud dataflow jobs list --region=$REGION --status=active --format="table(id,name,type,state)"
```

### A.4 Verify in BigQuery (after about 3–5 minutes)

```sql
SELECT * FROM pde_df_stream.taxi_realtime LIMIT 10;

SELECT ride_status, COUNT(*) AS events
FROM pde_df_stream.taxi_realtime
GROUP BY ride_status;

-- run twice, a minute apart: the count keeps growing
SELECT COUNT(*) AS total_rows FROM pde_df_stream.taxi_realtime;
```

**Expected:** `ride_status` values `enroute`, `pickup` and `dropoff`, and **`total_rows` increases** between runs.

> **Leave this job running.** Part C monitors it, then drains it.

---

## Part B – Pub/Sub → Cloud Storage text files (streaming to a data lake)

**Goal:** Land streaming events as **windowed files** in Cloud Storage. You publish your own events, so you control the data.

> **Windowing:** a stream never ends, so Dataflow groups events into **fixed windows**. The classic template uses **5-minute windows** and writes one set of files per window, when that window closes.

### B.1 Run the streaming job (UI)

**Dataflow → Jobs → + Create job from template**

| Field | Value |
|---|---|
| Job name | `pubsub-to-gcs-01` |
| Regional endpoint | `us-central1` |
| Dataflow template | **Pub/Sub Topic to Text Files on Cloud Storage** (classic) |
| Input Pub/Sub topic | `projects/<PROJECT_ID>/topics/pde-df-events` |
| Output file directory in Cloud Storage | `gs://<PROJECT_ID>-pde-df-stream/events/` (must end with `/`) |
| Output filename prefix | `events-` |
| Optional parameters | as in Step 0.5. Leave **window duration** empty if the field appears |

→ **Run job**

### B.1 CLI alternative

```bash
gcloud dataflow jobs run "pubsub-to-gcs-$(date +%s)" \
  --gcs-location="gs://dataflow-templates-$REGION/latest/Cloud_PubSub_to_GCS_Text" \
  --region="$REGION" \
  --staging-location="gs://$BUCKET/temp" \
  --service-account-email="$SA_EMAIL" \
  --subnetwork="regions/$REGION/subnetworks/default" \
  --disable-public-ips \
  --max-workers=2 \
  --parameters="inputTopic=projects/$PROJECT_ID/topics/$TOPIC,outputDirectory=gs://$BUCKET/events/,outputFilenamePrefix=events-"
```

> Do **not** add `windowDuration`. The classic template rejects it (`Unrecognized parameter`). Only the Flex version supports it, and Flex needs a launcher VM that may be blocked in this lab.
> To list the parameters the template accepts:
> `gcloud storage cat gs://dataflow-templates-$REGION/latest/Cloud_PubSub_to_GCS_Text_metadata`

### B.2 Wait until the job is Running

**UI:** **Dataflow → Jobs** → `pubsub-to-gcs-…` shows **Running** (2–3 minutes).

**CLI:**
```bash
gcloud dataflow jobs list --region=$REGION --status=active --format="table(id,name,type,state)"
gcloud pubsub subscriptions list --filter="topic~$TOPIC" --format="value(name)"
```

> ⚠️ **Publish only after the job is Running and its subscription exists.** Messages published earlier are not delivered to the job.

### B.3 Publish events

**UI:** **Pub/Sub → Topics → `pde-df-events` → Messages tab → Publish message**. Publish these one at a time:

```
{"order_id": 1, "status": "COMPLETE", "amount": 100}
{"order_id": 2, "status": "CANCELLED", "amount": 50}
{"order_id": 3, "status": "COMPLETE", "amount": 300}
```

**CLI alternative** (10 events, one every 5 seconds):
```bash
for i in $(seq 1 10); do
  gcloud pubsub topics publish $TOPIC \
    --message="{\"order_id\": $i, \"status\": \"COMPLETE\", \"amount\": $((i*100))}"
  sleep 5
done
```

### B.4 Confirm the job is reading

**UI:** **Dataflow → Jobs → `pubsub-to-gcs-…` → Job graph** → click the first step (**Read PubSub Events**). **Elements added** should match the number of messages you published.

### B.5 Verify the files (wait about 7 minutes after publishing)

**UI:** **Cloud Storage → `<PROJECT_ID>-pde-df-stream` → events/**. Files are named like
`events-2026-10-27T13:45:00.000Z-2026-10-27T13:50:00.000Z-pane-0-last-00000-of-00001`

**CLI:**
```bash
gcloud storage ls -r gs://$BUCKET/events/
gcloud storage cat "gs://$BUCKET/events/events-*"
```

**Expected:** the JSON messages you published, one per line. The window start and end times are in the file name.

> `One or more URLs matched no objects` means the window hasn't closed yet. Wait a few more minutes and check again.

---

## Part C – Monitor and stop the streaming jobs

Do this while **both jobs are still Running**.

### C.1 Job list

**UI:** **Dataflow → Jobs**: note the **Type** column (Streaming) and **Status** (Running).

**CLI:**
```bash
gcloud dataflow jobs list --region=$REGION --limit=10 --format="table(id,name,type,state,creationTime)"
```

### C.2 Job graph and step details

Open **`taxi-stream-…`** → **Job graph**:
1. Click **ReadPubSubTopic** to see *Elements added* and *Throughput*.
2. Click **WriteSuccessfulRecords** to see the rows written to BigQuery.
3. Failed records go to an **error records** table (`taxi_realtime_error_records`) if conversion fails.

**Job info** panel (right): worker machine type, service account, region, SDK version.

### C.3 Job metrics

Open the **Job metrics** tab:

| Metric | What it tells you |
|---|---|
| **Autoscaling** | Current vs target workers (max 2 here) |
| **Throughput** | Elements per second for each step |
| **Data freshness** | How far behind real time the output is. Low is healthy |
| **System latency** | How long an element waits to be processed |
| **CPU utilization** | Worker load |

### C.4 Logs and diagnostics

- At the bottom of the job page, open **Logs**: **Job logs** (startup, autoscaling, errors) and **Worker logs** (processing).
- **Diagnostics** tab: grouped errors and recommendations.
- **Error Reporting** (search bar): errors grouped across jobs.

**CLI:**
```bash
JOB_ID=$(gcloud dataflow jobs list --region=$REGION --status=active \
  --filter="name~taxi-stream" --format="value(id)" | head -1)
echo "JOB_ID=$JOB_ID"

gcloud logging read "resource.type=\"dataflow_step\" AND resource.labels.job_id=\"$JOB_ID\" AND severity>=WARNING" \
  --limit=10 --format="table(timestamp,severity,textPayload)"
```

> If `JOB_ID` is empty, copy the ID from `gcloud dataflow jobs list` and set it manually.

### C.5 (Optional) Create an alert

**Job metrics → Data freshness → ⋮ → Create alerting policy**, threshold **> 300 seconds** → **Create**. Delete it at cleanup.

### C.6 Stop the jobs: Drain vs Cancel

| Option | What happens | Use when |
|---|---|---|
| **Drain** | Stops reading new data, **finishes in-flight data**, then stops | Normal shutdown or pipeline update, no data loss |
| **Cancel** | Stops **immediately**; in-flight data may be lost | Broken or stuck job, or you need to stop now |

**UI:**
1. **`taxi-stream-…` → Stop → Drain → Stop job**
2. **`pubsub-to-gcs-…` → Stop → Cancel → Stop job**

**CLI alternative:**
```bash
TAXI_ID=$(gcloud dataflow jobs list --region=$REGION --status=active --filter="name~taxi-stream" --format="value(id)")
GCS_ID=$(gcloud dataflow jobs list --region=$REGION --status=active --filter="name~pubsub-to-gcs" --format="value(id)")

gcloud dataflow jobs drain  "$TAXI_ID" --region=$REGION
gcloud dataflow jobs cancel "$GCS_ID"  --region=$REGION
```

**Verify** after 2–5 minutes (Drain takes longer than Cancel):
```bash
gcloud dataflow jobs list --region=$REGION --limit=5 --format="table(name,type,state)"
```

**Expected:** `taxi-stream-…` shows **Drained**, and `pubsub-to-gcs-…` shows **Cancelled**.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `Unrecognized parameter: windowDuration` | Classic template has no window parameter | Remove `windowDuration`; the default is 5-minute windows |
| No files in `events/` | Messages published before the job was Running, or the window hasn't closed | Publish again after Running; wait about 7 minutes |
| Part A: no rows after 5 minutes | Job still starting, or write errors | Check the job is Running; check `taxi_realtime_error_records` and worker logs |
| `CustomVMrestriction` in job logs | Provider org policy blocks worker VMs | Report it to the trainer. Use the parts as a walkthrough |
| `Subnetwork … does not have Private Google Access` | PGA is off | Step 0.4 |
| `permission … dataflow.worker` / `pubsub.subscriptions.create` | Wrong service account, or a missing role | Use `nl-sa-…`; role issues are provider-side |
| `Location US is restricted` | Job or dataset not in us-central1 | Regional endpoint **us-central1**; dataset in us-central1 |
| `Output directory must end with /` | Missing trailing slash | `gs://…/events/` |
| `Unparseable filter` in `gcloud logging read` | `JOB_ID` is empty | Set the job ID manually |

---

## Cleanup (lab resources only)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-df-stream"
export DATASET="pde_df_stream"
export TOPIC="pde-df-events"

# 1. Stop any streaming jobs still running from this lab
for j in $(gcloud dataflow jobs list --region=$REGION --status=active \
           --filter="name~taxi-stream OR name~pubsub-to-gcs" --format="value(id)"); do
  gcloud dataflow jobs cancel "$j" --region=$REGION
done

# 2. Delete subscriptions the templates created (own topic + public taxi topic)
for s in $(gcloud pubsub subscriptions list --format="value(name)" \
           --filter="topic~$TOPIC OR topic~taxirides-realtime"); do
  gcloud pubsub subscriptions delete "$s" --quiet
done

# 3. Delete the topic, dataset and bucket
gcloud pubsub topics delete $TOPIC --quiet
bq rm -r -f -d "$PROJECT_ID:$DATASET"
gcloud storage rm -r "gs://$BUCKET"
```

**UI alternative:**
- **Dataflow → Jobs**: make sure no lab job is **Running**.
- **Pub/Sub**: delete topic `pde-df-events` and the subscriptions the templates created.
- **BigQuery**: delete dataset `pde_df_stream`.
- **Cloud Storage**: delete bucket `<PROJECT_ID>-pde-df-stream`.
- **Monitoring → Alerting**: delete the policy from C.5, if you created one.

> Never delete the `nl-sa-…` service account, the project, IAM bindings, or resources from other labs.

---

## Key takeaways

- **Pub/Sub + Dataflow** is the standard Google Cloud streaming pattern. The same stream can feed BigQuery (analytics) and Cloud Storage (data lake).
- Streaming jobs process **unbounded** data and use **windows** to group events in time.
- Monitor streaming health with **data freshness** and **system latency**.
- **Drain** for a clean stop with no data loss; **Cancel** to stop immediately. Streaming jobs bill until stopped.

## References

- Dataflow: Qwik Start – Templates: https://www.cloudskillsboost.google/focuses/1101
- Pub/Sub to BigQuery template: https://cloud.google.com/dataflow/docs/guides/templates/provided/pubsub-to-bigquery
- Pub/Sub to Cloud Storage text template: https://cloud.google.com/dataflow/docs/guides/templates/provided/pubsub-to-text
- Windowing in Dataflow: https://cloud.google.com/dataflow/docs/concepts/streaming-pipelines
- Stopping a pipeline (Drain vs Cancel): https://cloud.google.com/dataflow/docs/guides/stopping-a-pipeline

---
*Trainer: Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · deekshasharma191196@gmail.com*
