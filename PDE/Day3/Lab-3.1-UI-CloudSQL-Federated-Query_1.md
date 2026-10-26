# Lab 3.1: Cloud SQL PostgreSQL to BigQuery Federated Query

**Day 3 · From-scratch setup, end-to-end validation, and safe clean-up**

| Item | Details |
|---|---|
| **Related official lab** | [Federated query with BigQuery](https://www.skills.google/paths/16/course_templates/54) (course: Build Data Lakes and Data Warehouses on Google Cloud) |
| **Region** | `us-central1` |
| **Services** | Cloud SQL for PostgreSQL, BigQuery, BigQuery Connection API, IAM |
| **Duration** | ~45 minutes (Cloud SQL creation takes 5–10 minutes) |
| **How to use this guide** | Each step has a **🖱️ Console (UI) option** and a **⌨️ Cloud Shell (CLI) option**. Use either one; both give the same result |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ **PROTECTION:** never delete or modify the client-provided service account. Never delete the project, unrelated datasets, or shared IAM bindings.

---

## 💡 What We're Trying to Achieve

Companies keep **operational data** (orders, customers, payments) in transactional databases such as **Cloud SQL (PostgreSQL)**. Analysts want to query that data from **BigQuery**, often joining it with warehouse data, **without building a pipeline to copy it first**.

**Federated queries** solve this. BigQuery sends a query **directly to Cloud SQL** through a **connection** and gets the results back, while the data stays in PostgreSQL.

```
                         EXTERNAL_QUERY()
BigQuery  ───────────────────────────────────────►  Cloud SQL (PostgreSQL 15)
(analyst runs SQL)   through connection "psql"      database pde_lab31, table public.orders
        ◄───────────────────────────────────────
                     rows come back to BigQuery
```

| | Load data into BigQuery (Day 1) | Federated query (this lab) |
|---|---|---|
| Data location | Copied into BigQuery | **Stays in Cloud SQL** |
| Freshness | As fresh as the last load | **Live**, current data |
| Speed on large data | Fast | Slower (depends on the source database) |
| Typical use | Regular reporting | Quick lookups, joining live operational data, exploration |

### 🎯 Lab objective
1. Create a **Cloud SQL PostgreSQL 15** instance with an `orders` table (2 rows)
2. Create a **BigQuery connection** to that database
3. Grant the connection's **Google-managed service agent** the **Cloud SQL Client** role
4. Query PostgreSQL from BigQuery with **`EXTERNAL_QUERY()`**

**✅ Expected final result:** `total_orders = 2`, `total_amount = 300`.

> 🔑 Never paste database passwords into chat, files or shell commands where they're visible. This guide uses hidden password prompts.

---

## 01 | Open Cloud Shell and define the lab variables
Even if you use the Console for most steps, set these variables once; they print the exact values you'll type into the Console forms.

Open the [Google Cloud Console](https://console.cloud.google.com/), **select your lab project** in the project picker at the top, and click **Activate Cloud Shell**. Copy and run the whole block:

```bash
# Project (from the active Cloud Shell configuration)
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "❌ No project set. Run: gcloud config set project <YOUR_PROJECT_ID>  and re-run this block."
fi

# Project number (used in the BigQuery connection service agent)
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)" 2>/dev/null)
[ -z "$PROJECT_NUMBER" ] && export PROJECT_NUMBER=$(gcloud projects list --filter="projectId=$PROJECT_ID" --format="value(projectNumber)")

# Region allowed in the lab
export REGION="us-central1"

# Client-provided lab service account (starts with nl-sa-)
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

# Lab resource names
export INSTANCE="pde-day3-postgres"
export DB_NAME="pde_lab31"
export DATASET="pde_d3_l1"
export CONNECTION="pde_cloudsql"      # if the Console creates a different ID (e.g. psql), update this in Step 08
export CONNECTION_AGENT="service-${PROJECT_NUMBER}@gcp-sa-bigqueryconnection.iam.gserviceaccount.com"
export CONNECTION_NAME="${PROJECT_ID}:${REGION}:${INSTANCE}"

# Show what was detected
echo "PROJECT_ID       = $PROJECT_ID"
echo "PROJECT_NUMBER   = $PROJECT_NUMBER"
echo "REGION           = $REGION"
echo "SA_EMAIL         = $SA_EMAIL"
echo "CONNECTION_AGENT = $CONNECTION_AGENT"
echo "CONNECTION_NAME  = $CONNECTION_NAME"
```

✅ **Check:** every value is filled in.

| If empty | Fix |
|---|---|
| `PROJECT_ID` | `gcloud config set project <YOUR_PROJECT_ID>`, then re-run the block |
| `PROJECT_NUMBER` | Copy it from **Console → Dashboard → Project info**, run `export PROJECT_NUMBER=<number>`, then re-run the `CONNECTION_AGENT` line |
| `SA_EMAIL` | Find the `nl-sa-…` account under **IAM & Admin → Service Accounts** (or `client_email` in the lab panel's `serviceAccountKeyJson`) and run `export SA_EMAIL=<email>` |

**Run CLI commands as the lab service account:**
```bash
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
gcloud config set project "$PROJECT_ID"
gcloud config get-value auth/impersonate_service_account
```

✅ **Check:** the impersonation identity equals `$SA_EMAIL`.

> 💡 Every CLI command now prints `WARNING: This command is using service account impersonation…`. **That's normal.**
> 💡 If Cloud Shell reconnects, variables are lost. **Re-run this step** (and any `export CONNECTION=…` from Step 08).

---

## 02 | Validate authentication and check for existing lab resources

**🖱️ Console option**
- **Cloud SQL → Instances**: `pde-day3-postgres` should **not** exist yet.
- **BigQuery → Explorer**: dataset `pde_d3_l1` should **not** exist.
- **BigQuery → Explorer → Connections**: no Lab 3.1 connection yet.

**⌨️ CLI option**
```bash
gcloud auth print-access-token \
  --impersonate-service-account="$SA_EMAIL" \
  > /dev/null && echo "Impersonation successful"

gcloud sql instances list --project="$PROJECT_ID"
bq --project_id="$PROJECT_ID" ls --datasets
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"
```

✅ **Check:** `Impersonation successful`, and the lab instance, dataset and connection don't already exist. Don't delete unrelated resources.

> 💡 A Cloud Resource Manager API warning doesn't by itself mean impersonation failed; the access-token test is the identity check.

---

## 03 | Enable or confirm the required APIs (administrator-assisted if needed)
Required: **Cloud SQL Admin**, **BigQuery**, **BigQuery Connection**, **Cloud Resource Manager**, **Service Usage**.

**🖱️ Console option**
1. Go to **APIs & Services → Enabled APIs & services**.
2. Check that the five APIs above are listed.
3. If one is missing: **APIs & Services → Library** → search for it → **Enable**.

**⌨️ CLI option**
```bash
gcloud services list --enabled --project="$PROJECT_ID" --format="value(config.name)" \
  | grep -E "sqladmin|bigquery|bigqueryconnection|cloudresourcemanager|serviceusage"
```
If any are missing, an authorized identity runs this **once**:
```bash
gcloud services enable \
  sqladmin.googleapis.com \
  bigquery.googleapis.com \
  bigqueryconnection.googleapis.com \
  cloudresourcemanager.googleapis.com \
  --project="$PROJECT_ID"
```

> ⚠️ If `SERVICE_DISABLED` or `429 RATE_LIMIT_EXCEEDED` appears, stop and ask the client administrator. Don't keep retrying.

---

## 04 | Create Cloud SQL PostgreSQL 15
*Why:* this is the **operational database** holding the orders we want to query from BigQuery. It incurs Cloud SQL charges while running.

**🖱️ Console option**
1. Go to **Cloud SQL → Create instance → Choose PostgreSQL**.
2. Fill in:

| Setting | Value |
|---|---|
| Cloud SQL edition | **Enterprise** |
| Edition preset | **Sandbox** (or Development) |
| Database version | **PostgreSQL 15** |
| Instance ID | `pde-day3-postgres` |
| Password | Set a strong password for the `postgres` user and keep it safe (you can then skip setting the password in Step 05) |
| Region | **us-central1 (Iowa)** |
| Zonal availability | **Single zone** |
| Machine (Customize your instance → Machine configuration) | **Shared core → 1 vCPU, 1.7 GB** (`db-g1-small`) |
| Storage | **SSD, 10 GB** |
| Connections | **Public IP** ✅ (no authorized networks needed) |
| Data protection | Untick **Enable deletion protection** |

3. Click **Create instance**, then wait 5–10 minutes until the status shows a ✅ green tick.

**⌨️ CLI option**
```bash
gcloud sql instances create "$INSTANCE" \
  --project="$PROJECT_ID" \
  --database-version=POSTGRES_15 \
  --edition=ENTERPRISE \
  --tier=db-g1-small \
  --region="$REGION" \
  --availability-type=ZONAL \
  --storage-type=SSD \
  --storage-size=10GB \
  --assign-ip \
  --no-deletion-protection

gcloud sql instances describe "$INSTANCE" \
  --project="$PROJECT_ID" \
  --format="table(name,state,region,settings.tier,connectionName)"
```

✅ **Check:** state **RUNNABLE**, and the **Connection name** (Console: instance **Overview** page) equals `$CONNECTION_NAME`.

> ⚠️ If `ERROR_STOCKOUT` occurs, stop and request an approved alternative tier or capacity. Don't switch to a disallowed region.

---

## 05 | Create the database and set the postgres password
*Why:* the `pde_lab31` database holds our `orders` table; the `postgres` password is used by Cloud SQL Studio and by the BigQuery connection.

**🖱️ Console option**
1. Open **Cloud SQL → `pde-day3-postgres` → Databases → Create database**.
2. Database name: `pde_lab31` → **Create**.
3. If you didn't set a password in Step 04: **Users** → `postgres` → **⋮ → Change password** → enter a strong password → **OK**.

**⌨️ CLI option**

Create the database:
```bash
gcloud sql databases create "$DB_NAME" \
  --instance="$INSTANCE" \
  --project="$PROJECT_ID"
```

Set the password, **running this command on its own** (don't paste it together with other commands):
```bash
gcloud sql users set-password postgres \
  --instance="$INSTANCE" \
  --project="$PROJECT_ID" \
  --prompt-for-password
```
At `New Password:`, **type** the password (nothing is shown while you type) and press **Enter**.

> ⚠️ If you paste several commands at once, the next line is taken as an empty password, causing *"Missing user password for PostgreSQL instance"*. Just re-run the command alone.

```bash
gcloud sql databases list --instance="$INSTANCE" --project="$PROJECT_ID"
gcloud sql users list     --instance="$INSTANCE" --project="$PROJECT_ID"
```

✅ **Check:** database `pde_lab31` and user `postgres` are present. The `postgres` user is built in; don't create it again.

---

## 06 | Create the source table in PostgreSQL
*Why:* this is the **live operational data** BigQuery will query.

**🖱️ Console option (Cloud SQL Studio)**
1. Open **Cloud SQL → `pde-day3-postgres` → Cloud SQL Studio**.
2. Database **`pde_lab31`**, **Built-in authentication**, user **`postgres`**, the password from Step 04/05 → **Authenticate**.
3. Open a query editor, paste the SQL below, and click **Run**.

**⌨️ CLI option (psql from Cloud Shell)**
```bash
gcloud sql connect "$INSTANCE" --user=postgres --database="$DB_NAME" --project="$PROJECT_ID"
```
Enter the password when prompted, then paste the SQL below. Type `\q` to exit.

**SQL (for either option):**
```sql
SELECT current_database(), current_user;

CREATE TABLE IF NOT EXISTS public.orders (
  id     INTEGER PRIMARY KEY,
  amount INTEGER NOT NULL
);

INSERT INTO public.orders (id, amount)
VALUES (1, 100), (2, 200)
ON CONFLICT (id) DO UPDATE
SET amount = EXCLUDED.amount;

SELECT id, amount FROM public.orders ORDER BY id;

SELECT COUNT(*)    AS total_orders,
       SUM(amount) AS total_amount
FROM public.orders;
```

✅ **Check:** rows `(1, 100)` and `(2, 200)`; totals `2` and `300`.

> 💡 Cloud SQL Studio runs with the permissions of the person **signed in to the Console**, not the Cloud Shell service account. If Studio is denied, use the CLI option or ask the client for Console permissions.

---

## 07 | Create the BigQuery lab dataset in the permitted region
*Why:* a dataset in `us-central1` holds any BigQuery objects for this lab. BigQuery jobs and the connection must use the **same region** as Cloud SQL.

**🖱️ Console option**
1. **BigQuery → Explorer** → **⋮** next to your project → **Create dataset**.
2. Dataset ID: `pde_d3_l1`
3. Location type: **Region → us-central1 (Iowa)** (not the US multi-region)
4. Click **Create dataset**.

**⌨️ CLI option**
```bash
bq --project_id="$PROJECT_ID" --location="$REGION" \
  mk --dataset "$PROJECT_ID:$DATASET"

bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"
```

✅ **Check:** the dataset exists and its location is **`us-central1`**.

---

## 08 | Create the BigQuery connection to Cloud SQL
*Why:* the **connection** stores how BigQuery reaches the database (instance, database, credentials). It gets its own **Google-managed service agent**.

First, print the values you'll need:
```bash
echo "BigQuery link       : https://console.cloud.google.com/bigquery?project=${PROJECT_ID}"
echo "Region              : ${REGION}"
echo "Cloud SQL conn name : ${CONNECTION_NAME}"
echo "Database name       : ${DB_NAME}"
echo "Database username   : postgres"
```

**🖱️ Console option**
1. Open **BigQuery → Explorer → + Add data**.
2. Search **`postgres`** and click the **PostgreSQL** card.
3. Choose **BigQuery Federation** (query the data in place).
4. Fill in the **External data source** form:

| Field | Value |
|---|---|
| Connection ID (if shown, at the top) | `pde_cloudsql` |
| Region | **us-central1 (Iowa)** |
| Friendly name | `pde_cloudsql` (optional) |
| Encryption | Google-managed encryption key |
| **Cloud SQL connection name** | **`PROJECT_ID:REGION:INSTANCE`**, i.e. the *Cloud SQL conn name* printed above (e.g. `alchemyso-…:us-central1:pde-day3-postgres`). **Not** the database name |
| Database name | `pde_lab31` |
| Database username | `postgres` |
| Database password | The password from Step 04/05 |

5. Click **Create connection**.
6. Open the connection (**Explorer → Connections**) and note its **Connection ID** and **Service account ID**.

**⌨️ CLI option** (the password is read silently, not shown or saved)
```bash
read -s -p "Postgres password: " PG_PASS; echo

bq --project_id="$PROJECT_ID" mk --connection \
  --location="$REGION" \
  --connection_type=CLOUD_SQL \
  --properties="{\"instanceId\":\"${CONNECTION_NAME}\",\"database\":\"${DB_NAME}\",\"type\":\"POSTGRES\"}" \
  --connection_credential="{\"username\":\"postgres\",\"password\":\"${PG_PASS}\"}" \
  "$CONNECTION"

unset PG_PASS
```

**Check the connection ID, and update the variable if needed:**
```bash
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"
```
The connection name looks like `<PROJECT_NUMBER>.us-central1.<CONNECTION_ID>`. If the ID isn't `pde_cloudsql` (the Console may create **`psql`**), set it:
```bash
export CONNECTION=psql          # use the ID shown in the list
```

Confirm the details and the service agent:
```bash
bq --project_id="$PROJECT_ID" show --connection "$PROJECT_ID.$REGION.$CONNECTION"
echo "Expected agent: $CONNECTION_AGENT"
```

✅ **Check:** type `CLOUD_SQL`, `hasCredential: True`, `instanceId` = your connection name, `database: pde_lab31`, `type: POSTGRES`, and `serviceAccountId` = `$CONNECTION_AGENT`.

> 💡 The connection's service agent (`service-<PROJECT_NUMBER>@gcp-sa-bigqueryconnection…`) is **not** the `nl-sa-…` account.

---

## 09 | Grant Cloud SQL Client to the connection service agent
*Why:* BigQuery reaches Cloud SQL **as the connection's service agent**, which needs `cloudsql.instances.get` and `cloudsql.instances.connect`, i.e. the **Cloud SQL Client** role.

**🖱️ Console option**
1. Go to **IAM & Admin → IAM → Grant access**.
2. New principals: the `$CONNECTION_AGENT` value (`echo $CONNECTION_AGENT`).
3. Role: **Cloud SQL Client**.
4. Click **Save**.

**⌨️ CLI option**
```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$CONNECTION_AGENT" \
  --role="roles/cloudsql.client"
```

- This needs project IAM admin rights. If it fails with `setIamPolicy` denied, ask the **client administrator** to grant the role.
- **Don't** grant this role to the `nl-sa-…` account instead of the connection agent.
- The **identity running the query** also needs BigQuery job permissions and permission to use the connection (for example `roles/bigquery.connectionUser`): the signed-in user in the Console, or the impersonated service account in Cloud Shell.

---

## 10 | Query PostgreSQL from BigQuery with EXTERNAL_QUERY
*Why:* this is the goal of the lab: BigQuery runs the inner SQL **in PostgreSQL** and returns the results.

**🖱️ Console option (BigQuery Studio)**

Print the ready-to-paste queries with your values filled in:
```bash
cat <<EOF

-- Query 1: rows
SELECT id, amount
FROM EXTERNAL_QUERY('${PROJECT_ID}.${REGION}.${CONNECTION}', 'SELECT id, amount FROM public.orders')
ORDER BY id;

-- Query 2: totals
SELECT COUNT(*) AS total_orders, SUM(amount) AS total_amount
FROM EXTERNAL_QUERY('${PROJECT_ID}.${REGION}.${CONNECTION}', 'SELECT id, amount FROM public.orders');
EOF
```
1. Open **BigQuery → Studio → + (new query)**.
2. Paste **Query 1**, click **Run**, and check the results.
3. Paste **Query 2**, click **Run**, and check the totals.

> 💡 If you get a location error, open **More → Query settings → Location** and choose **us-central1**.

**⌨️ CLI option**

Row-level query:
```bash
bq --project_id="$PROJECT_ID" \
  --location="$REGION" \
  query --use_legacy_sql=false \
  "SELECT id, amount
   FROM EXTERNAL_QUERY(
     '$PROJECT_ID.$REGION.$CONNECTION',
     'SELECT id, amount FROM public.orders'
   )
   ORDER BY id"
```

Aggregate query:
```bash
bq --project_id="$PROJECT_ID" \
  --location="$REGION" \
  query --use_legacy_sql=false \
  "SELECT COUNT(*)    AS total_orders,
          SUM(amount) AS total_amount
   FROM EXTERNAL_QUERY(
     '$PROJECT_ID.$REGION.$CONNECTION',
     'SELECT id, amount FROM public.orders'
   )"
```

✅ **Check:** rows `(1, 100)` and `(2, 200)`; totals **`total_orders = 2`**, **`total_amount = 300`**. Only then mark federation as complete.

> 📝 **Discuss:** change the data in PostgreSQL (for example `UPDATE public.orders SET amount = 150 WHERE id = 1;` in Cloud SQL Studio) and re-run Query 2. BigQuery immediately shows the new total, because the data is **live**, not copied.

---

## 11 | Additional validation: BigQuery baseline (optional)
This confirms BigQuery itself works, separately from federation.

**🖱️ Console option:** in BigQuery Studio, run:
```sql
CREATE OR REPLACE TABLE pde_d3_l1.sample_words AS
SELECT 'alpha' AS word, 10 AS count UNION ALL
SELECT 'beta',    20 UNION ALL
SELECT 'gamma',   30 UNION ALL
SELECT 'delta',   40 UNION ALL
SELECT 'epsilon', 50;

SELECT COUNT(*) AS total_rows, SUM(count) AS total_word_count
FROM pde_d3_l1.sample_words;
```

**⌨️ CLI option**
```bash
bq --project_id="$PROJECT_ID" --location="$REGION" query --use_legacy_sql=false \
  "CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.sample_words\` AS
   SELECT 'alpha' AS word, 10 AS count UNION ALL
   SELECT 'beta',    20 UNION ALL
   SELECT 'gamma',   30 UNION ALL
   SELECT 'delta',   40 UNION ALL
   SELECT 'epsilon', 50"

bq --project_id="$PROJECT_ID" --location="$REGION" query --use_legacy_sql=false \
  "SELECT COUNT(*) AS total_rows, SUM(count) AS total_word_count
   FROM \`$PROJECT_ID.$DATASET.sample_words\`"
```

✅ **Check:** rows = **5**, sum = **150**. Federation must still independently return **2** and **300**.

---

## 12 | Troubleshooting

| Error | Fix |
|---|---|
| `The required property [project] is not currently set` | `gcloud config set project <PROJECT_ID>`, then re-run Step 01 |
| A variable is empty / `Not found: Connection pde_cloudsql` after reconnecting | Cloud Shell restarted; re-run Step 01 and `export CONNECTION=<your connection ID>` |
| `Not found: Connection pde_cloudsql` (but the connection exists) | The Console created a different ID (e.g. `psql`); `export CONNECTION=psql` |
| `Missing user password for PostgreSQL instance` | Re-run `set-password` **on its own** and type the password at the prompt |
| Connection form rejects the Cloud SQL connection name | Use `PROJECT_ID:us-central1:pde-day3-postgres`, not the database name |
| `iam.serviceAccounts.getAccessToken` denied | Ask the client admin to grant the training user `roles/iam.serviceAccountTokenCreator` **on** the `nl-sa-…` account |
| `SERVICE_DISABLED` (cloudresourcemanager / serviceusage) | Ask the admin to enable and verify these APIs |
| `429 RATE_LIMIT_EXCEEDED` | Stop repeating API enable commands; retry once later or contact the admin |
| `resourcemanager.projects.setIamPolicy` denied | The client IAM admin must grant Cloud SQL Client to the connection's service agent |
| Cloud SQL `ERROR_STOCKOUT` | Capacity limit; request an approved tier or zone in `us-central1` |
| BigQuery connection permission denied | Give the query identity permission to use the connection and create BigQuery jobs |
| PostgreSQL login failed | Re-enter the password; if needed, reset it and update the connection credentials (**Connections → Edit**) |
| `EXTERNAL_QUERY` can't connect / `cloudsql.instances.connect` denied | Instance RUNNABLE with a public IP, exact connection name and database, valid password, and the service agent has Cloud SQL Client |
| `relation "public.orders" does not exist` | Run Step 06 while connected to database **`pde_lab31`** |
| Location mismatch | Run the connection and BigQuery jobs in `us-central1` (Console: **Query settings → Location**) |

---

## 13 | Evidence checklist and completion criteria
Capture evidence of:
1. A successful service account impersonation test
2. The Cloud SQL instance in **RUNNABLE** state
3. The `pde_lab31` database and the `orders` rows
4. The BigQuery connection details and its service agent
5. The `EXTERNAL_QUERY` row result
6. The aggregate result **2 / 300**

Don't capture or share database passwords or access tokens.

---

## 14 | Optional clean-up, AFTER completing and recording the lab results
Clean-up deletes lab data permanently. The client-provided service account is preserved.

**🖱️ Console option**
1. **BigQuery → Explorer → Connections** → your Lab 3.1 connection → **Delete**.
2. **BigQuery → Explorer** → `pde_d3_l1` → **⋮ → Delete** → type `delete` to confirm.
3. **Cloud SQL → `pde-day3-postgres` → Delete** → type the instance ID to confirm.

**⌨️ CLI option**
```bash
# Inspect before deleting
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"
bq --project_id="$PROJECT_ID" ls --datasets
gcloud sql instances list --project="$PROJECT_ID"

# Delete ONLY the Lab 3.1 connection
bq --project_id="$PROJECT_ID" rm --connection "$PROJECT_ID.$REGION.$CONNECTION"

# Delete ONLY the Lab 3.1 dataset and its contents
bq --project_id="$PROJECT_ID" rm -r -d "$PROJECT_ID:$DATASET"

# Delete ONLY the Lab 3.1 PostgreSQL instance
gcloud sql instances delete "$INSTANCE" --project="$PROJECT_ID"

# Stop impersonating the service account
gcloud config unset auth/impersonate_service_account
```

> 🛡️ Don't delete the `nl-sa-…` service account, the BigQuery managed service agent, the project, unrelated datasets (for example `dataform`, `loadavro`, `nyctaxi`), or project IAM bindings. Don't disable project APIs.

---

## 🔁 Knowledge Check
1. Where does the data live when you use `EXTERNAL_QUERY`? → **In Cloud SQL**; BigQuery only receives the results
2. Which identity connects to Cloud SQL on BigQuery's behalf? → The connection's **Google-managed service agent**
3. Which role does it need? → **Cloud SQL Client**
4. When would you load data instead of federating? → For large, frequently queried data where **performance** matters
5. Why must the connection and query run in `us-central1`? → They must be in the **same region as the Cloud SQL instance** (and the lab blocks the US multi-region)

---

## 📚 References
- [Create a Cloud SQL for PostgreSQL instance](https://cloud.google.com/sql/docs/postgres/create-instance)
- [Manage data using Cloud SQL Studio](https://cloud.google.com/sql/docs/postgres/manage-data-using-studio)
- [Connect BigQuery to Cloud SQL](https://cloud.google.com/bigquery/docs/connect-to-sql)
- [Cloud SQL federated queries](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries)
- [bq command-line reference](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference)

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 3 Overview](README.md)
