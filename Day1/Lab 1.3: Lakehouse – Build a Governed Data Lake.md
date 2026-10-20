# Lab 1.3: Lakehouse – Build a Governed Data Lake

**Day 1 · Official Module: The Extract and Load Data Pipeline Pattern**

| Item | Details |
|---|---|
| **Official lab** | [Lakehouse: Qwik Start (GSP1040)](https://www.skills.google/focuses/37985) |
| **Builds on** | Lab 1.1 (`-raw` bucket, IAM) and Lab 1.2 (`nyctaxi` dataset, `trips_external` table) |
| **Level** | Introductory |
| **Duration** | ~35 minutes |
| **Services** | BigQuery, Cloud Storage, IAM (no Cloud SQL needed) |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

---

## 🎯 Objectives
1. Create a **connection resource** and give its service account access to the data lake
2. Create a **Lakehouse table** on files in Cloud Storage
3. Query the Lakehouse table from BigQuery
4. Protect sensitive columns with **column-level security** (policy tags)
5. **Upgrade** the external table from Lab 1.2 to a Lakehouse table

---

## Concepts

| Term | Meaning |
|---|---|
| **External table** (Lab 1.2) | BigQuery reads files in Cloud Storage, but **users need direct access to the bucket**, and fine-grained security isn't supported |
| **Lakehouse table** (formerly BigLake) | BigQuery reads the files **through a connection's service account**. Users don't need bucket access, and row- and column-level security works |
| **Connection resource** | A BigQuery object with its own **service account**, used to access Cloud Storage on behalf of users (access delegation) |
| **Policy tag** | A label on a column. Only users with the **Fine-Grained Reader** role on the tag can read that column |

```
User ──SQL──► BigQuery ──(connection service account)──► Cloud Storage (-raw bucket)
                 │
                 └── enforces row- and column-level security
```

---

## Task 1. Prepare the customer data in your data lake
Open **Cloud Shell** and upload a customer file to the bucket from Lab 1.1:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET=gs://${PROJECT_ID}-raw

cat > customer.csv <<EOF
customer_id,first_name,last_name,company,address,city,state,country,postal_code,phone,email
1,Asha,Verma,Infotech Ltd,12 MG Road,Bengaluru,KA,India,560001,+91-9800000001,asha.verma@example.com
2,Ravi,Kumar,DataWorks,45 Park Street,Kolkata,WB,India,700016,+91-9800000002,ravi.kumar@example.com
3,Meena,Iyer,,8 Marine Drive,Mumbai,MH,India,400020,+91-9800000003,meena.iyer@example.com
4,John,Smith,CloudCo,221 King St,Toronto,ON,Canada,M5H 1K5,+1-4165550104,john.smith@example.com
5,Sara,Lee,,77 Orchard Rd,Singapore,,Singapore,238841,+65-65550105,sara.lee@example.com
6,Karan,Mehta,Analytics Hub,3 Civil Lines,Pune,MH,India,411001,+91-9800000006,karan.mehta@example.com
EOF

gcloud storage cp customer.csv ${BUCKET}/customers/customer.csv
gcloud storage ls ${BUCKET}/customers/
```

> All names, phone numbers and emails are made up.

✅ **Check:** `customers/customer.csv` is in your bucket.

---

## Task 2. Create a connection resource
Turn on the API first:

```bash
gcloud services enable bigqueryconnection.googleapis.com
```

**Console:**
1. Go to **BigQuery → Studio**. In the **Explorer** panel, click **+ Add data**.
2. Search for and select **Agent Platform** (older Console versions call it **Vertex AI**). Choose **BigQuery Federation**.
3. Connection type: **Agent Platform remote models, remote functions, Lakehouse and Spanner (Cloud Resource)**
4. Connection ID: `my-connection`
5. Location type: **Multi-region → US**
6. Click **Create connection**.
7. Open the connection (Explorer → **Connections** → `us.my-connection`) and **copy the Service Account ID**.

**Command-line alternative:**
```bash
bq mk --connection --location=US --connection_type=CLOUD_RESOURCE my-connection
bq show --connection --location=US my-connection
```
Copy the `serviceAccountId` value from the output.

✅ **Check:** the connection `us.my-connection` exists and you have its service account email.

---

## Task 3. Give the connection access to the data lake
Grant the connection's service account **read-only** access to **your bucket only**:

```bash
export CONN_SA=<paste-the-service-account-id>

gcloud storage buckets add-iam-policy-binding ${BUCKET} \
  --member="serviceAccount:${CONN_SA}" \
  --role="roles/storage.objectViewer"
```

**Console alternative:** Cloud Storage → your `-raw` bucket → **Permissions** → **Grant access**. Paste the service account, choose the role **Storage Object Viewer**, then **Save**.

> 💡 Granting the role on the bucket rather than the whole project is the least-privilege approach. In production you would also remove direct bucket access from users, so they can only read the data through the Lakehouse table.

---

## Task 4. Create the Lakehouse table

**4.1 Create a dataset**
BigQuery → **⋮** next to your project → **Create dataset** → Dataset ID `demo_dataset`, location **Multi-region US** → **Create dataset**.

**4.2 Create the table in the Console**
1. Click **⋮** next to `demo_dataset` → **Create table**.

| Setting | Value |
|---|---|
| Create table from | **Google Cloud Storage** |
| Select file | `<PROJECT_ID>-raw/customers/customer.csv` |
| File format | CSV |
| Table | `customer_lakehouse` |
| Table type | **External table** |
| ✅ Create a Lakehouse table using a Cloud Resource connection | Connection `us.my-connection` |
| Schema | **Edit as text** (paste below) |
| Advanced options → Header rows to skip | `1` |

```json
[
 {"name":"customer_id","type":"INTEGER","mode":"REQUIRED"},
 {"name":"first_name","type":"STRING","mode":"REQUIRED"},
 {"name":"last_name","type":"STRING","mode":"REQUIRED"},
 {"name":"company","type":"STRING","mode":"NULLABLE"},
 {"name":"address","type":"STRING","mode":"NULLABLE"},
 {"name":"city","type":"STRING","mode":"NULLABLE"},
 {"name":"state","type":"STRING","mode":"NULLABLE"},
 {"name":"country","type":"STRING","mode":"NULLABLE"},
 {"name":"postal_code","type":"STRING","mode":"NULLABLE"},
 {"name":"phone","type":"STRING","mode":"NULLABLE"},
 {"name":"email","type":"STRING","mode":"REQUIRED"}
]
```

2. Click **Create table**.

**SQL alternative** (replace `<PROJECT_ID>` with your project ID):
```sql
CREATE OR REPLACE EXTERNAL TABLE demo_dataset.customer_lakehouse (
  customer_id INT64, first_name STRING, last_name STRING, company STRING,
  address STRING, city STRING, state STRING, country STRING,
  postal_code STRING, phone STRING, email STRING
)
WITH CONNECTION `us.my-connection`
OPTIONS (
  format = 'CSV',
  uris = ['gs://<PROJECT_ID>-raw/customers/customer.csv'],
  skip_leading_rows = 1
);
```

✅ **Check:** the table details show **Type: External** with the connection `us.my-connection`.

---

## Task 5. Query the Lakehouse table
```sql
SELECT * FROM demo_dataset.customer_lakehouse ORDER BY customer_id;

SELECT country, COUNT(*) AS customers
FROM demo_dataset.customer_lakehouse
GROUP BY country;
```

✅ **Check:** all 6 customers are returned. The data still lives only in Cloud Storage.

---

## Task 6. Protect sensitive columns (column-level security)

**6.1 Create a policy tag taxonomy**
1. Go to **BigQuery → Policy tags** (under *Governance* in the left menu; in some Console versions it's under **Dataplex → Policy tags**).
2. Click **Create taxonomy**:
   - Taxonomy name: `pii-taxonomy`
   - Location: **Multi-region US** (it must match the dataset location)
   - Policy tag: `pii-policy`
3. Click **Create**.
4. Open `pii-taxonomy` and switch **Enforce access control** to **ON**.

**6.2 Tag the sensitive columns**
1. Open `demo_dataset.customer_lakehouse` → **Schema** → **Edit schema**.
2. Select **address**, **postal_code** and **phone** → **Add policy tag** → `pii-taxonomy` → `pii-policy` → **Select** → **Save**.

**6.3 Test the security**
```sql
-- Fails with an access-denied error on the tagged columns
SELECT * FROM demo_dataset.customer_lakehouse;

-- Works: excludes the protected columns
SELECT * EXCEPT (address, postal_code, phone)
FROM demo_dataset.customer_lakehouse;
```

> 📝 **Discuss:** even the project owner can't read the tagged columns. Access needs the **Fine-Grained Reader** role on the policy tag. Optionally, grant it to yourself (open `pii-policy` → **Manage data policies / permissions** → add your login ID with **Fine-Grained Reader**) and re-run the first query.

✅ **Check:** `SELECT *` is denied and `SELECT * EXCEPT(...)` succeeds.

---

## Task 7. Upgrade the Lab 1.2 external table to a Lakehouse table
In Lab 1.2 you created `nyctaxi.trips_external`, a plain external table. Now link it to the connection so it becomes a Lakehouse table:

```bash
bq mkdef \
  --connection_id=${PROJECT_ID}.us.my-connection \
  --source_format=CSV \
  --autodetect \
  "${BUCKET}/nyctaxi/*.csv" > trips_def.json

bq update --external_table_definition=trips_def.json nyctaxi.trips_external

bq show nyctaxi.trips_external
```

Check it still works:
```sql
SELECT COUNT(*) AS trips FROM nyctaxi.trips_external;
```

✅ **Check:** `bq show` (or the table's **Details** tab) shows the connection `us.my-connection`.

---

## 🔁 Lab 1.2 vs Lab 1.3

| | External table (Lab 1.2) | Lakehouse table (Lab 1.3) |
|---|---|---|
| Who reads Cloud Storage | The **user** (needs bucket access) | The **connection service account** |
| Row- and column-level security | ❌ | ✅ |
| Open formats (Parquet, Iceberg) | Limited | ✅ Full support |
| Data stays in Cloud Storage | ✅ | ✅ |

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| *Access Denied* when creating the table | Task 3 wasn't done, or the role hasn't applied yet; wait 1–2 minutes and retry |
| Location mismatch error | The dataset, connection and taxonomy must all be in **US** |
| Policy tag doesn't block access | **Enforce access control** must be **ON** for the taxonomy |
| Can't create a taxonomy | The lab account may lack Data Catalog permissions; the trainer will demo |
| `bq update` fails on `trips_external` | Check the bucket path `nyctaxi/*.csv` exists from Lab 1.1 / Lab 1.2 |
| Connection can't be created | Run `gcloud services enable bigqueryconnection.googleapis.com` and retry |

---

## 🔁 Knowledge Check
1. What lets BigQuery read Cloud Storage without giving users bucket access? → A **connection resource (service account)**
2. Which table type supports column-level security on data in Cloud Storage? → A **Lakehouse table**
3. What role allows a user to read a column protected by a policy tag? → **Fine-Grained Reader**
4. Which Day 1 pipeline pattern does this lab represent? → **Extract and Load (EL)**: data stays in the lake and is queried in place

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab 1.2](Lab-1.2-Loading-Data-into-BigQuery.md) | ➡️ [Day 1 Overview](README.md)
