# Lab 5.3 – Building Batch Pipelines in Cloud Data Fusion

**Course:** Google Cloud Professional Data Engineer – Day 5 (Build Batch Data Pipelines on Google Cloud)
**Official lab:** Building Batch Pipelines in Cloud Data Fusion
**Mode:** UI (no code), with CLI for setup and verification
**Region:** `us-central1`

---

## What we are trying to achieve

We build a **no-code ETL pipeline** in **Cloud Data Fusion**, Google's managed, visual data integration service built on open-source **CDAP**:

| Stage | What happens |
|---|---|
| **Extract** | Read a raw orders CSV from **Cloud Storage** |
| **Transform** | Use **Wrangler** to set data types, uppercase `status` (`complete` → `COMPLETE`) and remove invalid rows (negative amount) |
| **Load** | Write the clean rows to a **BigQuery** table |
| *Optional* | **Join** with a customers table from BigQuery to enrich each order |

```
GCS: orders.csv ──► Wrangler (types, uppercase, filter) ──► BigQuery: pde_d5.orders_df
                                     │
BigQuery: pde_d5.customers ──────────┴──► Joiner ──► BigQuery: pde_d5.orders_enriched   (optional)
```

**Why Data Fusion?** Teams can build, preview and schedule pipelines by drag and drop, with 150+ connectors, and no Spark or Beam code. When the pipeline runs, Data Fusion converts it to a **Spark job on an ephemeral Dataproc cluster**.

> **Exam angle:** choose **Data Fusion** for visual, low-code ETL and many connectors; **Dataflow** for code-based batch or streaming (Beam); **Dataproc** for existing Spark/Hadoop code; **Dataform** for SQL transformations inside BigQuery.

---

## Before you start

- You are logged in to the lab project (see the lab access README).
- Use **Cloud Shell** (top-right **>_** icon) for the CLI steps.
- Use **us-central1** for everything.
- Use the client-provided service account **`nl-sa-…`**. Never delete or modify it.
- A Data Fusion instance takes **15–25 minutes** to create. If your lab already has one, use it and skip Step 2.

---

## Step 0 – Set variables (Cloud Shell)

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d5"
export DATASET="pde_d5"

echo "Project : $PROJECT_ID"
echo "Bucket  : $BUCKET"
```

> If **Project** is empty, run `gcloud config set project <your-project-id>` and run the block again.

---

## Step 1 – Enable APIs, create the bucket, dataset and sample data

### 1.1 Enable APIs

**UI:** **APIs & Services → Library →** enable **Cloud Data Fusion API** (and **Cloud Dataproc API** for full runs).

**CLI:**
```bash
gcloud services enable datafusion.googleapis.com dataproc.googleapis.com
```

### 1.2 Create the bucket and dataset

**UI:**
1. **Cloud Storage → Buckets → + Create**: `<PROJECT_ID>-pde-d5`, Region **us-central1** → **Create**.
2. **BigQuery → ⋮ next to project → Create dataset**: `pde_d5`, Region **us-central1** → **Create dataset**.

**CLI:**
```bash
gcloud storage buckets create gs://$BUCKET --location=$REGION
bq --location=$REGION mk -d "$PROJECT_ID:$DATASET"
```

### 1.3 Upload the sample orders file

The data includes mixed-case status values and one invalid row (negative amount), so the transformations have work to do:

```bash
cat > orders.csv <<'EOF'
order_id,customer_id,amount,status
1,101,100,complete
2,102,200,COMPLETE
3,101,50,cancelled
4,103,300,complete
5,104,-20,complete
EOF

gcloud storage cp orders.csv gs://$BUCKET/datafusion/orders.csv
gcloud storage cat gs://$BUCKET/datafusion/orders.csv
```

**UI alternative:** **Bucket → Create folder `datafusion` → Upload files → `orders.csv`**.

### 1.4 Create the target table

The BigQuery sink writes into an existing table, so create it first (Console **BigQuery → query editor**):

```sql
CREATE TABLE IF NOT EXISTS pde_d5.orders_df (
  order_id    INT64,
  customer_id INT64,
  amount      INT64,
  status      STRING
);
```

**CLI:**
```bash
bq mk --table "$PROJECT_ID:$DATASET.orders_df" \
  order_id:INTEGER,customer_id:INTEGER,amount:INTEGER,status:STRING
```

---

## Step 2 – Create (or open) the Data Fusion instance

### 2.1 Check for an existing instance

**UI:** Search bar → **Data Fusion → Instances**. If an instance is listed with status **Running**, skip to Step 2.3.

**CLI:**
```bash
gcloud beta data-fusion instances list --location=$REGION
```

### 2.2 Create the instance (only if none exists)

**UI:** **Data Fusion → Instances → + Create instance**

| Field | Value |
|---|---|
| Instance name | `pde-datafusion` |
| Region | `us-central1` |
| Edition | **Basic** (or **Developer**, if offered, for the lowest cost) |
| Authorization | Grant the Data Fusion service account the requested permission if prompted (provider-side if denied) |

→ **Create**. The status goes **Creating → Running** in 15–25 minutes.

**CLI alternative:**
```bash
gcloud beta data-fusion instances create pde-datafusion \
  --location=$REGION --edition=basic
```

> Use the waiting time for the theory part: CDAP, plugins, Wrangler and compute profiles.

### 2.3 Open the Data Fusion UI

**Data Fusion → Instances →** on your instance, click **View instance**. The CDAP Studio opens in a new tab.

---

## Step 3 – Build the pipeline in Pipeline Studio

**Data Fusion UI → ☰ menu → Studio** (or **Integrate → Studio**). Make sure the pipeline type at the top-left is **Data Pipeline – Batch**.

### 3.1 Add the source: Cloud Storage (GCS)

1. Left panel **Source → GCS**. The node appears on the canvas.
2. Hover over the node → **Properties**:

| Property | Value |
|---|---|
| Reference name | `orders_gcs` |
| Use connection | Off |
| Project ID | `auto-detect` |
| Path | `gs://<PROJECT_ID>-pde-d5/datafusion/orders.csv` |
| Format | **csv** |
| Use First Row as Header | **True** |
| Output schema | click **Get Schema** |

3. After **Get Schema**, the output schema shows `order_id`, `customer_id`, `amount`, `status`.
4. Click **Validate** → *No errors found* → close the panel (**X**).

> If the schema shows `body_0, body_1 …` instead of the column names, **Use First Row as Header** is not set to True. Set it and click **Get Schema** again.

### 3.2 Add the transform: Wrangler

1. Left panel **Transform → Wrangler**. Drag an arrow from the **GCS** node to the **Wrangler** node.
2. Wrangler → **Properties**:
   - Input field name: `*`
   - **Directives (Recipe)**: paste:
     ```
     set-type :order_id integer
     set-type :customer_id integer
     set-type :amount integer
     uppercase :status
     filter-rows-on condition-true amount < 0
     ```
   - **Output schema** (right side): set the types to `order_id` int, `customer_id` int, `amount` int, `status` string. You can also click **Get Schema** if it's available.
3. **Validate** → close.

| Directive | What it does |
|---|---|
| `set-type :order_id integer` | Converts the column from string to integer |
| `uppercase :status` | `complete` → `COMPLETE` |
| `filter-rows-on condition-true amount < 0` | **Removes** rows where the condition is true, so the invalid order 5 is dropped |

> **Interactive Wrangler (optional):** In the Wrangler node, click **Wrangle**, pick `orders.csv` from the bucket, and apply the same steps from each column's ▼ menu (**Change data type**, **Format → Uppercase**, **Filter**). Wrangler writes the directives for you.

### 3.3 Add the sink: BigQuery

1. Left panel **Sink → BigQuery**. Drag an arrow from **Wrangler** to **BigQuery**.
2. BigQuery → **Properties**:

| Property | Value |
|---|---|
| Reference name | `orders_bq` |
| Project ID | `auto-detect` |
| Dataset | `pde_d5` |
| Table | `orders_df` |
| Operation | **Insert** |
| Truncate Table | **True** (each run replaces the data) |
| Location | `us-central1` |

3. **Validate** → close.

### 3.4 Name and save the pipeline

Top of the canvas: click **Name your pipeline** → `orders_gcs_to_bq` → **Save**.

---

## Step 4 – Preview the pipeline (test without deploying)

**Preview** runs the pipeline on a small sample **inside the Data Fusion instance**. No Dataproc cluster is needed and no data is written to BigQuery.

1. Click **Preview** (top bar) → **Run**.
2. When it finishes, click **Preview Data** on each node:
   - **GCS:** 5 input records
   - **Wrangler:** 4 output records, `status` uppercase, order 5 missing
   - **BigQuery:** the 4 records it would write

> Preview is the quickest way to debug transformations. Fix the directives and preview again until the output is correct.

---

## Step 5 – Deploy and run the pipeline

1. Click **Preview** again to leave preview mode → **Deploy**.
2. On the deployed pipeline page → **Run**.
3. The status goes **Provisioning → Starting → Running → Succeeded** (about 6–10 minutes; most of it is creating the ephemeral Dataproc cluster).
4. Click **Logs** to follow progress. After the run, each node shows its **In / Out** record counts.

> **Compute profile:** by default the pipeline runs on an **ephemeral Dataproc cluster** that is created for the run and deleted afterwards. You can see or change it under **Configure → Compute config**.

---

## Step 6 – Verify in BigQuery

**UI:** **BigQuery → pde_d5 → orders_df → Preview**, or run:

```sql
SELECT * FROM pde_d5.orders_df ORDER BY order_id;
```

**CLI:**
```bash
bq query --use_legacy_sql=false \
  "SELECT * FROM \`$PROJECT_ID.$DATASET.orders_df\` ORDER BY order_id"
```

**Expected:**

| order_id | customer_id | amount | status |
|---|---|---|---|
| 1 | 101 | 100 | COMPLETE |
| 2 | 102 | 200 | COMPLETE |
| 3 | 101 | 50 | CANCELLED |
| 4 | 103 | 300 | COMPLETE |

Order 5 (amount −20) was **filtered out** by Wrangler.

---

## Step 7 (optional) – Enrich with a Joiner

**Goal:** Join each order with customer details from BigQuery, which is the multi-source pattern used in the official lab.

### 7.1 Create the customers and target tables

```sql
CREATE OR REPLACE TABLE pde_d5.customers AS
SELECT * FROM UNNEST([
  STRUCT(101 AS customer_id, 'Asha'  AS name, 'India'     AS country),
  STRUCT(102, 'Liam',  'Canada'),
  STRUCT(103, 'Wei',   'Singapore')
]);

CREATE TABLE IF NOT EXISTS pde_d5.orders_enriched (
  order_id INT64, customer_id INT64, amount INT64, status STRING,
  name STRING, country STRING
);
```

### 7.2 Extend the pipeline

1. Open the pipeline in **Studio**. On a deployed pipeline, use **Actions → Duplicate** to get an editable copy.
2. **Source → BigQuery**: Reference name `customers_bq`, Dataset `pde_d5`, Table `customers` → **Get Schema** → Validate.
3. **Analytics → Joiner**. Connect **Wrangler → Joiner** and **customers_bq → Joiner**.
4. Joiner **Properties**:
   - **Fields:** keep all fields from Wrangler, plus `name` and `country` from `customers_bq`
   - **Join type:** **Inner**
   - **Join condition:** `Wrangler.customer_id = customers_bq.customer_id`
   - Click **Get Schema** → Validate
5. **Sink → BigQuery** (a second sink): Reference name `orders_enriched_bq`, Dataset `pde_d5`, Table `orders_enriched`, Truncate **True**. Connect **Joiner → this sink**.
6. Save as `orders_enriched_pipeline` → **Preview → Run** → check the data → **Deploy → Run**.

### 7.3 Verify

```sql
SELECT order_id, name, country, amount, status
FROM pde_d5.orders_enriched
ORDER BY order_id;
```

**Expected:** 4 rows. Asha / India for orders 1 and 3, Liam / Canada for order 2, Wei / Singapore for order 4.

---

## Step 8 (optional) – Schedule the pipeline

On the deployed pipeline page → **Schedule** → set, for example, **daily at 02:00** → **Save and start schedule**. Turn the schedule **off** again before cleanup.

> For production orchestration with dependencies across services, use **Cloud Composer**. Data Fusion pipelines can be triggered from Airflow.

---

## Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| Schema shows `body_0, body_1, …` | Header row not recognised | GCS source → **Use First Row as Header = True** → **Get Schema** |
| `Table … does not exist` at the sink | Target table not created | Run the DDL in Step 1.4 (or 7.1) |
| `Location US is restricted` / location mismatch | Dataset or sink not in us-central1 | Dataset in us-central1; sink **Location = us-central1** |
| Wrangler output still has `amount = -20` | Filter directive missing or mistyped | `filter-rows-on condition-true amount < 0` (after `set-type :amount integer`) |
| Type error at the sink (string vs int) | Wrangler output schema still says string | Set the output schema types in Wrangler, or add the `set-type` directives |
| Run stuck in **Provisioning**, or fails with `CustomVMrestriction` / `autoscalingPolicies.use` / `dataproc.*` permission | Ephemeral Dataproc cluster blocked or no permission | Provider-side policy or role. Use **Preview** (Step 4) to show the results |
| Instance creation fails with a permission error | Data Fusion service agent roles missing | Provider-side; use an existing instance if the lab has one |

---

## Cleanup (lab resources only)

> ⚠️ **A Data Fusion instance bills every hour it exists, even when idle.** If you created it in Step 2.2, delete it at the end.

### UI

1. **Data Fusion UI → List** → select the pipelines → **Delete**. Turn off any schedule first.
2. **Data Fusion → Instances →** select `pde-datafusion` → **Delete** (only if you created it).
3. **BigQuery** → delete tables `orders_df`, `customers`, `orders_enriched` (or the whole `pde_d5` dataset if no other lab needs it).
4. **Cloud Storage** → delete the `datafusion/` folder (or the whole bucket).

### CLI

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
export REGION="us-central1"
export BUCKET="${PROJECT_ID}-pde-d5"
export DATASET="pde_d5"

bq rm -f "$PROJECT_ID:$DATASET.orders_df"
bq rm -f "$PROJECT_ID:$DATASET.customers"
bq rm -f "$PROJECT_ID:$DATASET.orders_enriched"
gcloud storage rm -r "gs://$BUCKET/datafusion"

# Only if YOU created the instance in Step 2.2
gcloud beta data-fusion instances delete pde-datafusion --location=$REGION --quiet
```

> Never delete the `nl-sa-…` service account, the project, IAM bindings, a Data Fusion instance provided by the lab, or resources from other labs.

---

## Key takeaways

- **Cloud Data Fusion** is a visual, low-code ETL service based on CDAP. Pipelines are made of **source → transform → sink** plugins.
- **Wrangler** applies data preparation steps (type changes, formatting, filtering) as recorded **directives**.
- **Preview** tests a pipeline inside the instance. **Deploy + Run** executes it as Spark on an **ephemeral Dataproc cluster**.
- **Joiner** combines multiple sources. Pipelines can be **scheduled** or triggered by Composer.

## References

- Cloud Data Fusion overview: https://cloud.google.com/data-fusion/docs/concepts/overview
- Create a data pipeline (quickstart): https://cloud.google.com/data-fusion/docs/create-data-pipeline
- Wrangler directives: https://cdap.atlassian.net/wiki/spaces/DOCS/pages/382042117/Wrangler+Directives
- Compute profiles: https://cloud.google.com/data-fusion/docs/concepts/compute-profiles
- Building Batch Data Pipelines on Google Cloud (course): https://www.cloudskillsboost.google/course_templates/53

---
*Trainer: Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · deekshasharma191196@gmail.com*
