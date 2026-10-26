**DAY 4  /  LAB 4.1**

# Cloud Storage to BigQuery: Extract, Load and Transform

Hands-on batch ETL: create source CSV, stage into BigQuery, apply quality filters and validate totals.

> **SAFETY: Never disable, delete, or change the client-provided service account or unrelated IAM bindings, project resources, datasets, or buckets. Console uses the signed-in user, not Cloud Shell impersonation.**

## Lab configuration

| Field | Value |
|---|---|
| Project | alchemyso-1727076968159 |
| Region | us-central1 |
| Authentication | nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com |
| Objective | 3 curated orders; total amount 600 |

## Prerequisites and ownership

Requires storage.buckets.create, storage.objects.create, BigQuery dataset/table and job permissions. Bucket names are global; confirm availability. This lab is independent of Day 5 managed services.

These are custom instructor labs aligned to ETL/pipeline skills in the Google learning path; the publicly accessible course overview does not enumerate these four exact exercises. UI labels may vary. Choose either UI or CLI for each action, not both when the operation creates a resource.


## Exact values to enter in Console

| Setting | Enter this value |
|---|---|
| Project ID | `alchemyso-1727076968159` |
| Region (not US multi-region) | `us-central1` |
| CLI impersonated identity | `nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com` |
| Dataset ID | `pde_d4_etl_l1` |
| Bucket name | `alchemyso-1727076968159-d4etl-l1-8159` |
| Object | `orders.csv` |
| Raw table | `raw_orders` |
| Curated table | `curated_orders` |

**Console path is a numbered UI procedure; Cloud Shell CLI is the alternate way to perform the same action. Do not create each resource twice.**

## 01  |  Set up identity and lab variables

### CONSOLE UI — numbered instructions

1. Open **https://console.cloud.google.com/** and sign in with the authorized training user.
2. Choose **Select a project** and select **`alchemyso-1727076968159`**.
3. Click **Activate Cloud Shell** (terminal icon), then use the CLI block below to verify the service account can be impersonated.
4. **CLI identity:** `nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com`. **Location:** `us-central1`. The browser Console does **not** inherit Cloud Shell impersonation.
5. If token verification fails or the service account is disabled, **stop** and contact the client administrator. Do not create keys or change IAM bindings.

### CLOUD SHELL / CLI ALTERNATIVE

```bash
export PROJECT_ID="alchemyso-1727076968159"
export REGION="us-central1"
export SA_EMAIL="nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com"
gcloud config set project "$PROJECT_ID"
gcloud auth list
gcloud auth print-access-token --impersonate-service-account="$SA_EMAIL" >/dev/null && echo 'IMPERSONATION_OK'
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
export DATASET="pde_d4_etl_l1"
export BUCKET="${PROJECT_ID}-d4etl-l1-${PROJECT_ID: -4}"
export FILE="orders.csv"
```

**VERIFY / PASS: Token test returns IMPERSONATION_OK. If it fails, stop.**

## 02  |  Confirm APIs and resource names

### CONSOLE UI — numbered instructions

1. Go to **APIs & Services → Enabled APIs & services**. Confirm **BigQuery API** and **Cloud Storage** are usable; ask the administrator if service access is restricted.
2. Go to **BigQuery Studio → Explorer**; expand project **`alchemyso-1727076968159`**. Verify dataset **`pde_d4_etl_l1`** does not exist.
3. Go to **Cloud Storage → Buckets** and search for exact bucket **`alchemyso-1727076968159-d4etl-l1-8159`**. If the name already exists, stop; do not overwrite someone else's bucket.
4. Record the project **`alchemyso-1727076968159`** and region **`us-central1`**.

### CLOUD SHELL / CLI ALTERNATIVE

```bash
gcloud services list --enabled --project="$PROJECT_ID" --format='value(config.name)' | grep -E 'bigquery|storage' || true
bq --project_id="$PROJECT_ID" ls --datasets
gcloud storage buckets list --project="$PROJECT_ID" --format='value(name)' | grep -F "$BUCKET" || true
```

**VERIFY / PASS: Do not proceed if the exact bucket or dataset is already used.**

## 03  |  Create regional bucket and dataset

### CONSOLE UI — numbered instructions

1. Navigate to **Cloud Storage → Buckets → Create**.
2. **Name:** `alchemyso-1727076968159-d4etl-l1-8159`.
3. **Location type:** `Region`; **Region:** `us-central1`.
4. **Access control:** `Uniform`; **Public access prevention:** `Enforced`; retain approved encryption/default settings.
5. Click **Create**.
6. Navigate to **BigQuery Studio → Explorer → alchemyso-1727076968159 → ⋮ → Create dataset**.
7. **Dataset ID:** `pde_d4_etl_l1`; **Location type:** `Region`; **Region:** `us-central1`.
8. Click **Create dataset**; confirm both newly created resources appear.

### CLOUD SHELL / CLI ALTERNATIVE

```bash
gcloud storage buckets create "gs://$BUCKET" --project="$PROJECT_ID" --location="$REGION" --uniform-bucket-level-access --public-access-prevention
bq --project_id="$PROJECT_ID" --location="$REGION" mk --dataset "$PROJECT_ID:$DATASET"
```

**VERIFY / PASS: Both resources created in us-central1.**

## 04  |  Prepare deterministic raw CSV and upload

### CONSOLE UI — numbered instructions

1. Create a UTF-8 file called **`orders.csv`** locally with the exact CSV contents in the block below.
2. Open **Cloud Storage → Buckets → `alchemyso-1727076968159-d4etl-l1-8159`**.
3. Click **Upload files** and choose **`orders.csv`**.
4. Verify the object name **`orders.csv`**, full URI **`gs://alchemyso-1727076968159-d4etl-l1-8159/orders.csv`**, and **4 data records** (5 lines including header).
5. Do not upload into another client's bucket.

**Exact file contents (`orders.csv`):**
```csv
order_id,customer_id,amount,status
1,101,100,COMPLETE
2,102,200,COMPLETE
3,101,50,CANCELLED
4,103,300,COMPLETE
```

### CLOUD SHELL / CLI ALTERNATIVE

```bash
mkdir -p "$HOME/day4etl-l1"
cat > "$HOME/day4etl-l1/orders.csv" <<'CSV'
order_id,customer_id,amount,status
1,101,100,COMPLETE
2,102,200,COMPLETE
3,101,50,CANCELLED
4,103,300,COMPLETE
CSV
gcloud storage cp "$HOME/day4etl-l1/orders.csv" "gs://$BUCKET/$FILE"
gcloud storage ls "gs://$BUCKET/$FILE"
```

**VERIFY / PASS: Object exists with four data records.**

## 05  |  Load raw CSV to BigQuery staging table

### CONSOLE UI — numbered instructions

1. Open **BigQuery Studio → Explorer → `pde_d4_etl_l1` → Create table**.
2. **Create table from:** `Google Cloud Storage`; **Select file:** `gs://alchemyso-1727076968159-d4etl-l1-8159/orders.csv`.
3. **File format:** `CSV`; **Destination project:** `alchemyso-1727076968159`; **Dataset:** `pde_d4_etl_l1`; **Table:** `raw_orders`.
4. Select **Edit as text** or manually define the schema: `order_id INTEGER`, `customer_id INTEGER`, `amount INTEGER`, `status STRING`.
5. Under **Advanced options**, set **Header rows to skip:** `1`; use **Write if empty** (not overwrite unrelated tables).
6. Click **Create table**. Open **raw_orders → Preview** and confirm four records.

### CLOUD SHELL / CLI ALTERNATIVE

```bash
bq --project_id="$PROJECT_ID" --location="$REGION" load --replace --source_format=CSV --skip_leading_rows=1 "$PROJECT_ID:$DATASET.raw_orders" "gs://$BUCKET/$FILE" 'order_id:INTEGER,customer_id:INTEGER,amount:INTEGER,status:STRING'
```

**VERIFY / PASS: Raw row count = 4.**

## 06  |  Transform raw rows into curated completed orders

### CONSOLE UI — numbered instructions

1. Open **BigQuery Studio → SQL query** (not Cloud SQL Studio).
2. Set **Query settings → Processing location → us-central1**.
3. Paste the SQL below, with project `alchemyso-1727076968159` and dataset `pde_d4_etl_l1`.
4. Click **Run** and confirm table **`curated_orders`** is created.
5. Verify the `CANCELLED` order (order_id `3`) is excluded.

**SQL to paste in BigQuery Studio:**
```sql
CREATE OR REPLACE TABLE `alchemyso-1727076968159.pde_d4_etl_l1.curated_orders` AS
SELECT order_id, customer_id, amount
FROM `alchemyso-1727076968159.pde_d4_etl_l1.raw_orders`
WHERE status = 'COMPLETE' AND amount > 0;
```

### CLOUD SHELL / CLI ALTERNATIVE

```bash
bq --project_id="$PROJECT_ID" --location="$REGION" query --use_legacy_sql=false <<SQL
CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.curated_orders\` AS
SELECT order_id,customer_id,amount
FROM \`$PROJECT_ID.$DATASET.raw_orders\`
WHERE status='COMPLETE' AND amount>0;
SQL
```

**VERIFY / PASS: Curated table excludes CANCELLED order.**

## 07  |  Validate ETL totals and quality

### CONSOLE UI — numbered instructions

1. In **BigQuery Studio → SQL query**, select **us-central1**.
2. Paste the validation SQL below; click **Run**.
3. Check **Results:** `orders=3`, `total_amount=600`, `distinct_orders=3`.
4. Capture query results and the `raw_orders` / `curated_orders` tables.

**SQL to paste in BigQuery Studio:**
```sql
SELECT COUNT(*) AS orders, SUM(amount) AS total_amount, COUNT(DISTINCT order_id) AS distinct_orders
FROM `alchemyso-1727076968159.pde_d4_etl_l1.curated_orders`;
```

### CLOUD SHELL / CLI ALTERNATIVE

```bash
bq --project_id="$PROJECT_ID" --location="$REGION" query --use_legacy_sql=false <<SQL
SELECT COUNT(*) AS orders,SUM(amount) AS total_amount,COUNT(DISTINCT order_id) AS distinct_orders
FROM \`$PROJECT_ID.$DATASET.curated_orders\`;
SQL
```

**VERIFY / PASS: orders=3, total_amount=600, distinct_orders=3.**

## 08  |  Cleanup only resources owned by this lab

### CONSOLE UI — numbered instructions

1. **Only after saving evidence**, go to **BigQuery Studio → `pde_d4_etl_l1` → Delete dataset**. Confirm the exact dataset name.
2. Go to **Cloud Storage → Buckets → `alchemyso-1727076968159-d4etl-l1-8159`**; delete only its lab files and then that exact bucket.
3. Do **not** delete the project, service account, IAM roles, other datasets, or any bucket you did not create for this lab.
4. Reopen dataset/bucket lists and verify the two lab resources are absent.

### CLOUD SHELL / CLI ALTERNATIVE

```bash
# Run ONLY after proof captured; destructive for the EXACT lab bucket/dataset.
bq --project_id="$PROJECT_ID" rm -r -f -d "$PROJECT_ID:$DATASET"
gcloud storage rm --recursive "gs://$BUCKET"
# Never delete a client service account or unrelated resources.
```

**VERIFY / PASS: Verify dataset/bucket absent; other resources preserved.**

## Troubleshooting and escalation

PERMISSION_DENIED: check which identity runs the action. Console account and impersonated CLI identity may differ. Never work around IAM by creating private keys. SERVICE_DISABLED: authorized administrator enables API. RATE_LIMIT_EXCEEDED: stop repeated mutations. Quota/region error: keep resources in us-central1 and request approved quota. Unexpected pre-existing resource: stop rather than overwrite or delete it.

## Proof to retain

Capture the resource page, successful execution/job status, final query result, and cleanup list. No claims of successful cloud deployment are made until real project outputs are verified.

## Official references

- https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv

- https://cloud.google.com/storage/docs/creating-buckets
