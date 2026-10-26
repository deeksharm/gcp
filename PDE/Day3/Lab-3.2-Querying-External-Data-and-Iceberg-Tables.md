# Lab 3.2: Query Cloud Storage Data from BigQuery (External, Native and Iceberg Tables)

**Day 3 · Build Data Lakes and Data Warehouses on Google Cloud**

| Item | Details |
|---|---|
| **Related official labs** | [Querying external data and Iceberg tables](https://www.skills.google/paths/16/course_templates/54) |
| **Region** | `us-central1 (Iowa)` |
| **Services** | Cloud Storage, BigQuery, BigQuery Connection API |
| **Duration** | ~40 minutes |
| **How to use this guide** | Each step has a **🖱️ Console (UI)** option first and a **⌨️ Cloud Shell (CLI)** alternative. Use either |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ Never delete or modify the client-provided service account (`nl-sa-…`), the project, shared IAM bindings, unrelated datasets or existing connections.

---

## 💡 What We're Trying to Achieve
Raw data often lands in **Cloud Storage** as files (the **data lake**). BigQuery can work with that data in different ways, and this lab compares them:

| Table type | Where the data lives | Use case |
|---|---|---|
| **External table** | Stays in Cloud Storage (CSV); BigQuery reads it at query time | Query lake files without loading them |
| **Native table** | Copied into BigQuery storage | Fast, regular analytics |
| **Iceberg table** (optional) | Open **Apache Iceberg / Parquet** files in **your** bucket, managed by BigQuery | **Lakehouse**: open format usable by BigQuery **and** other engines such as Spark |

```
orders.csv (Cloud Storage) ──► orders_external (external table) ──► orders_native (copy in BigQuery)

BigQuery ──(Cloud Resource connection)──► orders_iceberg (Iceberg/Parquet files in your bucket)   [optional]
```

**✅ Expected results:** external and native tables → **3 rows / total 600**; Iceberg table → **2 rows / total 300**.

### Values used in this lab

| Item | Value |
|---|---|
| Region | `us-central1 (Iowa)` |
| Bucket | `YOUR_PROJECT_ID-pde-d3-l2` |
| CSV file | `lab32/orders.csv` |
| Dataset | `pde_d3_l2` |
| Tables | `orders_external`, `orders_native`, `orders_iceberg` |
| Iceberg connection | `pde_d3_l2_iceberg` |

> 💡 **`YOUR_PROJECT_ID`** = your project ID, shown in the Console's **project picker**. Replace it wherever it appears in the **Console and SQL** steps.
> ⚠️ `${...}` / `$VARIABLE` values only work in **Cloud Shell**. In the **BigQuery editor**, always type the real values.

---

## Step 1. Select your project (and set CLI variables, optional)

**🖱️ Console**
1. Open [https://console.cloud.google.com/](https://console.cloud.google.com/) and select your lab project in the **project picker**.
2. Note your **Project ID**.

**⌨️ CLI alternative:** run once in Cloud Shell if you'll use any CLI options. All values are detected automatically:
```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "❌ No project set. Run: gcloud config set project <YOUR_PROJECT_ID>"
fi
export REGION="us-central1"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)
export DATASET="pde_d3_l2"
export BUCKET="${PROJECT_ID}-pde-d3-l2"
export CSV_OBJECT="lab32/orders.csv"
export EXT_TABLE="orders_external"
export NATIVE_TABLE="orders_native"
export ICE_CONN="pde_d3_l2_iceberg"
export ICE_TABLE="orders_iceberg"

echo "PROJECT_ID=$PROJECT_ID | BUCKET=$BUCKET | SA_EMAIL=$SA_EMAIL"

# Run CLI commands as the lab service account
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
```

✅ **Check:** the project, bucket and service account values are filled in.

> 💡 If Cloud Shell reconnects, variables are lost; re-run this block.

---

## Step 2. Check the required APIs

**🖱️ Console**
1. **APIs & Services → Enabled APIs & services**.
2. Check that **BigQuery API**, **Cloud Storage API** and **BigQuery Connection API** (needed for Iceberg) are listed.
3. If one is missing: **APIs & Services → Library** → search → **Enable**.

**⌨️ CLI alternative**
```bash
gcloud services list --enabled --format='value(config.name)' \
  | grep -E '^(bigquery|storage|bigqueryconnection)\.googleapis\.com$'
```

✅ **Check:** all three APIs are enabled.

---

## Step 3. Create the bucket and dataset

**🖱️ Console: bucket**
1. **Cloud Storage → Buckets → Create**.
2. Name: **`YOUR_PROJECT_ID-pde-d3-l2`**
3. Location type: **Region → us-central1 (Iowa)**
4. Storage class: **Standard**
5. Access control: **Uniform**, and ✅ **Enforce public access prevention**
6. Click **Create**.

**🖱️ Console: dataset**
1. **BigQuery → Studio → Explorer → ⋮** next to your project → **Create dataset**.
2. Dataset ID: **`pde_d3_l2`**
3. Location type: **Region → us-central1 (Iowa)**
4. Click **Create dataset**.

**⌨️ CLI alternative**
```bash
gcloud storage buckets create "gs://$BUCKET" --location="$REGION" \
  --uniform-bucket-level-access --public-access-prevention
bq --location="$REGION" mk --dataset "$PROJECT_ID:$DATASET"
```

✅ **Check:** the bucket and dataset both exist in **us-central1**.

> 💡 Bucket names are global. If the name is taken, add a suffix (e.g. `-01`) and use that name everywhere in this lab.

---

## Step 4. Create and upload the CSV file

**🖱️ Console**
1. On your computer, create a text file named **`orders.csv`** containing:
```
id,amount,category
1,100,books
2,200,toys
3,300,books
```
2. **Cloud Storage → your bucket → Create folder** → name it `lab32` → **Create**.
3. Open `lab32` → **Upload → Upload files** → select `orders.csv`.

**⌨️ CLI alternative**
```bash
mkdir -p "$HOME/pde-lab32"
cat > "$HOME/pde-lab32/orders.csv" <<'CSV'
id,amount,category
1,100,books
2,200,toys
3,300,books
CSV
gcloud storage cp "$HOME/pde-lab32/orders.csv" "gs://$BUCKET/$CSV_OBJECT"
gcloud storage ls "gs://$BUCKET/$CSV_OBJECT"
```

✅ **Check:** `lab32/orders.csv` is in the bucket.

---

## Step 5. Create the external table (data stays in Cloud Storage)

**🖱️ Console: Create table form**
1. **BigQuery → Explorer → `pde_d3_l2` → ⋮ → Create table**.
2. Fill in:

| Setting | Value |
|---|---|
| Create table from | **Google Cloud Storage** |
| Select file | `YOUR_PROJECT_ID-pde-d3-l2/lab32/orders.csv` |
| File format | **CSV** |
| Table | `orders_external` |
| Table type | **External table** |
| Schema | **Edit as text** → `id:INTEGER,amount:INTEGER,category:STRING` |
| Advanced options → Header rows to skip | `1` |

3. Click **Create table**.

**🖱️ Console: SQL alternative** (BigQuery Studio query tab; replace `YOUR_PROJECT_ID`):
```sql
CREATE EXTERNAL TABLE pde_d3_l2.orders_external
(id INT64, amount INT64, category STRING)
OPTIONS (
  format = 'CSV',
  skip_leading_rows = 1,
  uris = ['gs://YOUR_PROJECT_ID-pde-d3-l2/lab32/orders.csv']
);
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE EXTERNAL TABLE \`$PROJECT_ID.$DATASET.$EXT_TABLE\`
 (id INT64, amount INT64, category STRING)
 OPTIONS(format='CSV', skip_leading_rows=1, uris=['gs://$BUCKET/$CSV_OBJECT'])"
```

> 💡 An explicit schema avoids wrong type guesses from the CSV.

✅ **Check:** the table's **Details** tab shows **Type: External** and the source URI.

---

## Step 6. Validate the external table

**🖱️ Console:** in a BigQuery Studio query tab, run:
```sql
SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount
FROM pde_d3_l2.orders_external;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount FROM \`$PROJECT_ID.$DATASET.$EXT_TABLE\`"
```

✅ **Check:** **total_orders = 3**, **total_amount = 600**.

---

## Step 7. Create a native table and compare

**🖱️ Console:** run in BigQuery Studio:
```sql
CREATE TABLE pde_d3_l2.orders_native AS
SELECT * FROM pde_d3_l2.orders_external;

SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount
FROM pde_d3_l2.orders_native;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE TABLE \`$PROJECT_ID.$DATASET.$NATIVE_TABLE\` AS SELECT * FROM \`$PROJECT_ID.$DATASET.$EXT_TABLE\`"
bq --location="$REGION" query --use_legacy_sql=false \
"SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount FROM \`$PROJECT_ID.$DATASET.$NATIVE_TABLE\`"
```

✅ **Check:** the native table also returns **3 / 600**.

**Compare:** open each table's **Details** tab.

| | `orders_external` | `orders_native` |
|---|---|---|
| Type | External | Table (native) |
| Storage in BigQuery | 0 bytes (data in Cloud Storage) | Has storage size and row count |

> 📝 **Try it:** add a row `4,400,games` to `orders.csv`, upload it again (overwrite), then re-run both counts. The **external** table shows **4** rows immediately; the **native** table still shows **3**, because it's a copy.

---

## Step 8 (optional). Iceberg: create a Cloud Resource connection
*Why:* a BigQuery-managed **Iceberg** table writes **open-format files into your bucket**. BigQuery does this through a **connection's service account**.

**🖱️ Console**
1. **BigQuery → Explorer → + Add data**.
2. Search **`Vertex AI`** (some Console versions say **Agent Platform**) → choose **BigQuery Federation**.
3. Connection type: **Vertex AI remote models, remote functions, BigLake and Spanner (Cloud Resource)** (wording may vary).
4. Connection ID: **`pde_d3_l2_iceberg`**
5. Location: **Region → us-central1**
6. Click **Create connection**.
7. Open it (**Explorer → Connections**) and copy the **Service account ID** (`bqcx-…@gcp-sa-bigquery-condel.iam.gserviceaccount.com`).

**⌨️ CLI alternative**
```bash
bq --location="$REGION" mk --connection --connection_type=CLOUD_RESOURCE "$ICE_CONN"
export ICE_AGENT=$(bq show --format=prettyjson --connection "$PROJECT_ID.$REGION.$ICE_CONN" \
  | grep -o '"serviceAccountId": *"[^"]*"' | cut -d'"' -f4)
echo "ICE_AGENT=$ICE_AGENT"
```

✅ **Check:** the connection exists in **us-central1** and you have its service account.

> ⚠️ Don't reuse unrelated existing connections.

---

## Step 9 (optional). Give the connection access to the bucket

**🖱️ Console**
1. **Cloud Storage → `YOUR_PROJECT_ID-pde-d3-l2` → Permissions → Grant access**.
2. New principals: the **service account ID** from Step 8.
3. Role: **Storage Object User** (read, write, delete and list objects in this bucket).
4. Click **Save**.

**⌨️ CLI alternative**
```bash
gcloud storage buckets add-iam-policy-binding "gs://$BUCKET" \
  --member="serviceAccount:$ICE_AGENT" --role="roles/storage.objectUser"
```

✅ **Check:** the bucket's **Permissions** tab lists the connection's service account with **Storage Object User**.

> ⚠️ Grant access on **this lab bucket only**, not the whole project. If you get a permission error, the lab provider or admin must do it.

---

## Step 10 (optional). Create and validate the Iceberg table

**🖱️ Console:** run in BigQuery Studio (replace `YOUR_PROJECT_ID`):
```sql
CREATE TABLE pde_d3_l2.orders_iceberg
(id INT64, amount INT64)
WITH CONNECTION `us-central1.pde_d3_l2_iceberg`
OPTIONS (
  file_format  = 'PARQUET',
  table_format = 'ICEBERG',
  storage_uri  = 'gs://YOUR_PROJECT_ID-pde-d3-l2/iceberg/orders_iceberg'
);

INSERT INTO pde_d3_l2.orders_iceberg VALUES (1, 100), (2, 200);

SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount
FROM pde_d3_l2.orders_iceberg;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE TABLE \`$PROJECT_ID.$DATASET.$ICE_TABLE\` (id INT64, amount INT64)
 WITH CONNECTION \`$PROJECT_ID.$REGION.$ICE_CONN\`
 OPTIONS (file_format='PARQUET', table_format='ICEBERG', storage_uri='gs://$BUCKET/iceberg/orders_iceberg')"

bq --location="$REGION" query --use_legacy_sql=false \
"INSERT INTO \`$PROJECT_ID.$DATASET.$ICE_TABLE\` VALUES (1,100),(2,200)"

bq --location="$REGION" query --use_legacy_sql=false \
"SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount FROM \`$PROJECT_ID.$DATASET.$ICE_TABLE\`"
```

✅ **Check:** **total_orders = 2**, **total_amount = 300**.

> 📝 **See the open format:** go to **Cloud Storage → your bucket → `iceberg/orders_iceberg/`**. You'll find `data/` (Parquet files) and `metadata/` (Iceberg metadata). Other engines such as Spark can read these files. Don't edit these files directly; change the table only through BigQuery.

---

## ✅ Validation Checklist

| # | Check | Pass / Fail |
|---|---|---|
| 1 | Project selected (CLI variables filled in, if used) | |
| 2 | BigQuery, Storage and Connection APIs enabled | |
| 3 | Bucket and dataset in **us-central1** | |
| 4 | `lab32/orders.csv` uploaded | |
| 5 | `orders_external` created (Type: External) | |
| 6 | External table returns **3 / 600** | |
| 7 | `orders_native` returns **3 / 600**; Details compared | |
| 8 | Iceberg connection created; service account copied (optional) | |
| 9 | Bucket permission granted to the connection's service account (optional) | |
| 10 | Iceberg table returns **2 / 300**; files visible in the bucket (optional) | |

---

## 🛠️ Troubleshooting

| Error | Fix |
|---|---|
| `The required property [project] is not currently set` | `gcloud config set project <PROJECT_ID>`, then re-run Step 1 |
| Bucket name already exists | Add a suffix (`-01`) and use it everywhere in the lab |
| `US violates constraint … resourceLocations` | Create the bucket, dataset and connection in **us-central1** |
| `bigquery.jobs.create` permission denied in the Console | Check you're signed in with the lab account and the right project is selected; if it persists, use the CLI option or ask the trainer |
| Query in the editor fails on `${PROJECT_ID}` / `$BUCKET` | Variables only work in Cloud Shell; type the real values in the editor |
| *Access Denied* reading the external table | Your account needs read access to the bucket; check it's your lab bucket |
| Location mismatch | The bucket, dataset and connection must all be in **us-central1**; set **More → Query settings → Location → us-central1** |
| Iceberg `CREATE TABLE` 403 / access denied | Step 9 is missing; the connection's service account needs **Storage Object User** on the bucket |
| Iceberg error that the storage URI isn't empty | Use a new, empty folder path; don't re-run `CREATE TABLE` on a used prefix |
| "Vertex AI" / "Cloud Resource" option not in **Add data** | Use the CLI alternative in Step 8 |

---

## 🧹 Optional Clean-up (after recording your results)

**🖱️ Console**
1. **BigQuery → `pde_d3_l2.orders_iceberg` → Delete** (if created).
2. **BigQuery → Explorer → Connections → `pde_d3_l2_iceberg` → Delete**.
3. **BigQuery → `pde_d3_l2` → ⋮ → Delete** → type `delete`.
4. **Cloud Storage → Buckets → `YOUR_PROJECT_ID-pde-d3-l2`** → **Delete** → type `DELETE`.

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
  "DROP TABLE IF EXISTS \`$PROJECT_ID.$DATASET.$ICE_TABLE\`"
bq rm -f --connection "$PROJECT_ID.$REGION.$ICE_CONN"
bq rm -r -f -d "$PROJECT_ID:$DATASET"
gcloud storage rm --recursive "gs://$BUCKET"
rm -rf "$HOME/pde-lab32"
gcloud config unset auth/impersonate_service_account
```

> 💡 Dropping an Iceberg table **doesn't delete its files** in Cloud Storage; deleting the bucket removes them. Only delete the lab bucket, never a shared one.

---

## 🔁 Knowledge Check
1. Where does an external table's data live? → **In Cloud Storage**; BigQuery reads it at query time
2. Why did the external table show the new CSV row immediately, but the native table didn't? → The native table is a **copy** stored in BigQuery
3. What makes an Iceberg table a "lakehouse" table? → **Open format** files in your bucket, usable by BigQuery **and** other engines
4. Why does the Iceberg table need a connection? → BigQuery writes the files **as the connection's service account**, which needs access to the bucket
5. Why must the bucket, dataset and connection share a region? → BigQuery requires **co-located** resources (here **us-central1**)

---

## 📚 References
- [Query Cloud Storage data in external tables](https://cloud.google.com/bigquery/docs/external-data-cloud-storage)
- [BigLake Iceberg tables in BigQuery](https://cloud.google.com/bigquery/docs/iceberg-tables)
- [Create and set up a Cloud resource connection](https://cloud.google.com/bigquery/docs/create-cloud-resource-connection)

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab 3.1](Lab-3.1-CloudSQL-Federated-Query.md) | ➡️ [Day 3 Overview](README.md)
