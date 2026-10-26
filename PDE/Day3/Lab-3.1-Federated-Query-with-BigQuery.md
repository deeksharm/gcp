# Lab 3.1: Cloud SQL PostgreSQL to BigQuery Federated Query

**Day 3 · Execution guide: from-scratch setup, end-to-end validation, and safe clean-up**

| Item | Details |
|---|---|
| **Related official lab** | [Federated query with BigQuery](https://www.skills.google/paths/16/course_templates/54) (course: Build Data Lakes and Data Warehouses on Google Cloud) |
| **Region** | `us-central1` |
| **Services** | Cloud SQL for PostgreSQL, BigQuery, BigQuery Connection API, IAM |
| **Duration** | ~45 minutes (Cloud SQL creation takes 5–10 minutes) |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ **PROTECTION:** never delete or modify the client-provided service account. Never delete the project, unrelated datasets, or shared IAM bindings.

## 🎯 Lab objective
Provision **Cloud SQL PostgreSQL 15**, insert two rows into an `orders` table, create a **BigQuery Cloud SQL connection**, grant its Google-managed service agent **Cloud SQL Client** access, and query the PostgreSQL data from BigQuery using **`EXTERNAL_QUERY()`**.

**✅ Expected final result:** `total_orders = 2`, `total_amount = 300`.

> 🔑 The guide uses interactive database-password entry. Don't paste passwords into chat, shell history or files.

```
Cloud SQL (PostgreSQL 15)  ◄──  BigQuery connection (pde_cloudsql)  ◄──  EXTERNAL_QUERY() in BigQuery
   pde_lab31.public.orders        service agent has Cloud SQL Client          results returned to BigQuery
```

---

## 01 | Open Cloud Shell and define the lab resources
Open the [Google Cloud Console](https://console.cloud.google.com/) with your lab project selected, then click **Activate Cloud Shell**. Copy and run the entire block.

**All values are detected automatically from your session; nothing is hard-coded.**

```bash
# Project (from the active Cloud Shell configuration)
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "❌ No project set. Run: gcloud config set project <YOUR_PROJECT_ID>  and re-run this block."
fi

# Project number (needed for the BigQuery connection service agent)
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
export CONNECTION="pde_cloudsql"
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

✅ **Check:** every value is filled in; none is empty.

| If empty | Fix |
|---|---|
| `PROJECT_ID` | `gcloud config set project <YOUR_PROJECT_ID>`, then re-run the block |
| `PROJECT_NUMBER` | Copy it from **Console → Dashboard → Project info** and run `export PROJECT_NUMBER=<number>`, then re-run the `CONNECTION_AGENT` line |
| `SA_EMAIL` | Find the `nl-sa-…` account under **IAM & Admin → Service Accounts** (or `client_email` in the lab panel's `serviceAccountKeyJson`) and run `export SA_EMAIL=<email>` |

**Use the lab service account for gcloud/bq commands:**
```bash
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
gcloud config set project "$PROJECT_ID"
gcloud config get-value auth/impersonate_service_account
```

✅ **Check:** the impersonation identity equals `$SA_EMAIL`.

> 💡 Cloud Shell sessions can restart. If you reconnect, **re-run this whole step** before continuing.

---

## 02 | Validate authentication and check for existing lab resources
```bash
gcloud auth print-access-token \
  --impersonate-service-account="$SA_EMAIL" \
  > /dev/null && echo "Impersonation successful"

gcloud sql instances list --project="$PROJECT_ID"
bq --project_id="$PROJECT_ID" ls --datasets
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"
```

Continue only if this lab's instance (`pde-day3-postgres`), dataset (`pde_d3_l1`) and connection (`pde_cloudsql`) **don't already exist**. Don't delete unrelated resources.

> 💡 A Cloud Resource Manager API warning doesn't by itself mean impersonation failed; the access-token test is the identity check.

---

## 03 | Enable or confirm the required APIs (administrator-assisted if needed)
Required services: `sqladmin.googleapis.com`, `bigquery.googleapis.com`, `bigqueryconnection.googleapis.com`, `cloudresourcemanager.googleapis.com`, `serviceusage.googleapis.com`.

First, a read-only check:
```bash
gcloud services list --enabled --project="$PROJECT_ID" --format="value(config.name)" \
  | grep -E "sqladmin|bigquery|bigqueryconnection|cloudresourcemanager|serviceusage"
```

If any required API is missing, an authorized project administrator should enable it in **APIs & Services → Library**, or run this **once** with an authorized identity:
```bash
gcloud services enable \
  sqladmin.googleapis.com \
  bigquery.googleapis.com \
  bigqueryconnection.googleapis.com \
  cloudresourcemanager.googleapis.com \
  --project="$PROJECT_ID"
```

> ⚠️ If `SERVICE_DISABLED` or `429 RATE_LIMIT_EXCEEDED` appears, stop making API changes and ask the client administrator to enable or verify the Service Usage and Cloud Resource Manager APIs. Don't keep retrying the enable command.

---

## 04 | Create Cloud SQL PostgreSQL 15
This uses a small Enterprise configuration. It **incurs Cloud SQL charges while running**. No broad authorized networks are added.

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
```

```bash
gcloud sql instances describe "$INSTANCE" \
  --project="$PROJECT_ID" \
  --format="table(name,state,region,settings.tier,connectionName)"
```

✅ **Check:** the instance state is **RUNNABLE**, and `connectionName` equals `$CONNECTION_NAME`.

> ⚠️ If `ERROR_STOCKOUT` occurs, stop and request an approved alternative tier or capacity. Don't switch to a disallowed region.

---

## 05 | Create the database and set the postgres password securely
```bash
gcloud sql databases create "$DB_NAME" \
  --instance="$INSTANCE" \
  --project="$PROJECT_ID"

gcloud sql users set-password postgres \
  --instance="$INSTANCE" \
  --project="$PROJECT_ID" \
  --prompt-for-password
```

At the hidden prompt, enter a strong password according to client policy, and keep it safe. You'll need it for Cloud SQL Studio and the BigQuery connection. The `postgres` user is built in; don't try to create it again.

```bash
gcloud sql databases list --instance="$INSTANCE" --project="$PROJECT_ID"
gcloud sql users list     --instance="$INSTANCE" --project="$PROJECT_ID"
```

✅ **Check:** database `pde_lab31` and user `postgres` are present.

---

## 06 | Log in to Cloud SQL Studio and create the source table
Print your Cloud SQL Console link:
```bash
echo "https://console.cloud.google.com/sql/instances?project=${PROJECT_ID}"
```

1. Open the link and select **`pde-day3-postgres`**.
2. Click **Cloud SQL Studio**.
3. Choose database **`pde_lab31`**, **Built-in authentication**, username **`postgres`**, and the password from Step 05.
4. Click **Authenticate**.
5. Open a query editor, paste the SQL below, and click **Run**.

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

✅ **Check:** rows `(1, 100)` and `(2, 200)`; totals `2` and `300`. If different, correct the table contents before continuing.

> 💡 Cloud SQL Studio runs with the permissions of the person **signed in to the Console**; it doesn't use the account configured in Cloud Shell. If Studio is denied, ask the client for the right Console-user permissions.

---

## 07 | Create the BigQuery lab dataset in the permitted region
```bash
bq --project_id="$PROJECT_ID" --location="$REGION" \
  mk --dataset "$PROJECT_ID:$DATASET"

bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"
```

✅ **Check:** the dataset exists and its location is **`us-central1`** (not the US multi-region).

---

## 08 | Create the BigQuery Cloud SQL connection (Console, secure password entry)
Print the values you'll need for the form:
```bash
echo "BigQuery link       : https://console.cloud.google.com/bigquery?project=${PROJECT_ID}"
echo "Connection ID       : ${CONNECTION}"
echo "Location            : ${REGION}"
echo "Cloud SQL conn name : ${CONNECTION_NAME}"
echo "Database            : ${DB_NAME}"
echo "Username            : postgres"
```

1. Open the BigQuery link.
2. In **Explorer**, select the project and open **+ Add data → Connections to external data sources** (UI wording can vary).
3. Connection type: **Cloud SQL for PostgreSQL**.
4. Connection ID: **`pde_cloudsql`**
5. Location: **`us-central1`**
6. Cloud SQL connection name: the **Cloud SQL conn name** printed above
7. Database: **`pde_lab31`**
8. Username: **`postgres`**
9. Password: the password from Step 05
10. Click **Create connection**.
11. Open the new connection and copy its **Service account ID**.

> 💡 The connection's service account is a **Google-managed service agent** (`service-<PROJECT_NUMBER>@gcp-sa-bigqueryconnection.iam.gserviceaccount.com`). It is **not** the client-provided `nl-sa-…` account.

```bash
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"

bq --project_id="$PROJECT_ID" show --connection "$PROJECT_ID.$REGION.$CONNECTION"
```

Confirm the service agent matches the variable from Step 01:
```bash
echo "Expected agent: $CONNECTION_AGENT"
bq --project_id="$PROJECT_ID" show --format=prettyjson --connection "$PROJECT_ID.$REGION.$CONNECTION" | grep -i serviceAccountId
```

✅ **Check:** `pde_cloudsql` exists in `us-central1`, points to the right instance and database, and its `serviceAccountId` equals `$CONNECTION_AGENT`. If it differs, run `export CONNECTION_AGENT=<serviceAccountId shown>`.

---

## 09 | Grant Cloud SQL Client to the connection service agent
BigQuery needs `cloudsql.instances.get` and `cloudsql.instances.connect` through its managed service agent. Ask an administrator to confirm whether the agent already has `roles/cloudsql.client`. If not, an **authorized IAM administrator** should run:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$CONNECTION_AGENT" \
  --role="roles/cloudsql.client"
```

- If you're explicitly authorized to administer project IAM, run the same command under the impersonated service account.
- It's normal for this command to fail when the Cloud Resource Manager or Service Usage APIs are disabled, or when `resourcemanager.projects.setIamPolicy` is missing. Those need a client administrator.
- **Don't** grant this role to the `nl-sa-…` account instead of the connection agent.

The **query-running identity** also needs BigQuery query permissions and permission to use the connection (for example `roles/bigquery.connectionUser`, or a broader role). In the Console, that's the signed-in user; in Cloud Shell, the impersonated service account.

---

## 10 | Query the PostgreSQL table from the BigQuery CLI
`bq` uses the configured impersonation. Always run the job in `us-central1`.

**Row-level query:**
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

✅ **Check:** records `(1, 100)` and `(2, 200)`.

**Aggregate query:**
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

✅ **Check:** `total_orders = 2` and `total_amount = 300`. Only then mark federation as complete.

> 💡 **How it works:** BigQuery sends the inner SQL (`SELECT id, amount FROM public.orders`) to PostgreSQL through the connection, gets the results back, and the outer SQL runs in BigQuery. The data stays in Cloud SQL.

---

## 11 | Additional validation: BigQuery baseline
This optional check confirms BigQuery itself works, separately from federation. It doesn't replace the `EXTERNAL_QUERY` test.

```bash
bq --project_id="$PROJECT_ID" \
  --location="$REGION" \
  query --use_legacy_sql=false \
  "CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.sample_words\` AS
   SELECT 'alpha' AS word, 10 AS count UNION ALL
   SELECT 'beta',    20 UNION ALL
   SELECT 'gamma',   30 UNION ALL
   SELECT 'delta',   40 UNION ALL
   SELECT 'epsilon', 50"
```

```bash
bq --project_id="$PROJECT_ID" \
  --location="$REGION" \
  query --use_legacy_sql=false \
  "SELECT COUNT(*) AS total_rows, SUM(count) AS total_word_count
   FROM \`$PROJECT_ID.$DATASET.sample_words\`"
```

✅ **Check:** baseline rows = **5** and sum = **150**. Federation must still independently return **2** and **300**.

---

## 12 | Troubleshooting

| Error | Fix |
|---|---|
| `iam.serviceAccounts.getAccessToken` denied | Ask the client administrator to grant the training user `roles/iam.serviceAccountTokenCreator` **on** the `nl-sa-…` account; verify impersonation again |
| `SERVICE_DISABLED` (cloudresourcemanager / serviceusage) | Ask the admin to enable and verify these APIs; IAM changes can't be verified until project API access works |
| `429 RATE_LIMIT_EXCEEDED` | Stop repeating API enable commands; retry once after the quota window resets, or contact the client admin |
| `resourcemanager.projects.setIamPolicy` denied | The client IAM admin must grant Cloud SQL Client to the Google-managed BigQuery connection service agent |
| Cloud SQL `ERROR_STOCKOUT` | Cloud SQL capacity limit; request an approved tier or zone in `us-central1`. Don't assume creation succeeded |
| BigQuery connection permission denied | Give the query identity permission to use the connection and create BigQuery jobs, scoped by client policy |
| PostgreSQL login failed | Re-enter the password from Step 05; if needed, reset it securely and update the connection credentials |
| `EXTERNAL_QUERY` can't connect | Check the instance is RUNNABLE with a public IP (no broad authorized networks), the connection's instance and database are exact, the password is valid, and the connection agent has Cloud SQL Client |
| `public.orders does not exist` | Go back to Cloud SQL Studio, sign in to **`pde_lab31`** specifically, and run Step 06 |
| BigQuery location mismatch | Run both the connection and the BigQuery job in `us-central1`; don't use the US multi-region |
| A variable is empty after reconnecting | Cloud Shell restarted; re-run Step 01 |

---

## 13 | Evidence checklist and completion criteria
Capture evidence of:
1. A successful service account impersonation test
2. The Cloud SQL instance in **RUNNABLE** state
3. The `pde_lab31` database and the `orders` rows in Cloud SQL Studio
4. The BigQuery connection details and its service agent
5. The `EXTERNAL_QUERY` row result
6. The aggregate result **2 / 300**

Record any administrator-provided IAM grant confirmation. **Don't capture or share database passwords or access tokens.**

---

## 14 | Optional clean-up, AFTER completing and recording the lab results
Clean-up deletes lab data permanently. Never run it before collecting evidence, and confirm no one else uses these lab resources. The client-provided service account is explicitly preserved.

**Inspect before deleting:**
```bash
bq --project_id="$PROJECT_ID" ls --connection --location="$REGION"
bq --project_id="$PROJECT_ID" ls --datasets
gcloud sql instances list --project="$PROJECT_ID"
```

**Delete ONLY the Lab 3.1 connection:**
```bash
bq --project_id="$PROJECT_ID" rm --connection "$PROJECT_ID.$REGION.$CONNECTION"
```

**Delete ONLY the Lab 3.1 dataset and its contents:**
```bash
bq --project_id="$PROJECT_ID" rm -r -d "$PROJECT_ID:$DATASET"
```

**Delete ONLY the Lab 3.1 PostgreSQL instance:**
```bash
gcloud sql instances delete "$INSTANCE" --project="$PROJECT_ID"
```

> 🛡️ Don't delete the `nl-sa-…` service account, the BigQuery managed service agent, the project, unrelated BigQuery datasets (for example `dataform`, `loadavro`, `nyctaxi`), or project IAM role bindings. Don't disable project APIs during clean-up.

**Optional: stop impersonating the service account when you're done:**
```bash
gcloud config unset auth/impersonate_service_account
```

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
