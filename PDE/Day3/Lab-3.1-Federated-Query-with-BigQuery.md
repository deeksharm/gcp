# Day 3 — Lab 3.1: Federated Query with BigQuery

> **Training exercise aligned to the trainer's eight-day plan, not an official Google Skills graded-lab reproduction.** This guide distinguishes a quick baseline from the named service's real functionality. Do not mark the full lab complete based on the baseline alone.

## Goal and acceptance criteria
**Full-service objective:** Cloud SQL PostgreSQL federation.  
**Evidence required:** Actual external query returns two source rows and 300 sum. If Cloud SQL permission denied, mark full-service test BLOCKED.

## Prerequisites and safe setup (Cloud Shell)
1. Open [Google Cloud Console](https://console.cloud.google.com/) and choose **your assigned training project**. From the top toolbar choose **Activate Cloud Shell**.
2. Verify the account, project and default location. This example uses the client's shared training project only if the instructor has confirmed it is approved:
```bash
export PROJECT_ID="$(gcloud config get-value project 2>/dev/null)"
export REGION=us-central1
export BQ_LOCATION=US
printf 'PROJECT_ID=%s\n' "$PROJECT_ID"
gcloud auth list --filter='status:ACTIVE'
```
3. Confirm correct project ID **before executing any command**. If blank, `gcloud config set project YOUR_PROJECT_ID`; never type a password, service-account key, or sensitive client data into lab source code.
4. Required access: BigQuery Job User and permissions to create/update tables in your specific training dataset for the SQL baseline. Full-service IAM, services, networking and cost approvals depend on the advanced section below. If permissions fail, **stop instead of escalating or weakening policy**.
5. For a dedicated dataset create it if permitted; `bq show` confirms existence:
```bash
export DATASET=pde_d3_l1
bq --location=US mk -d "$PROJECT_ID:$DATASET" || bq show "$PROJECT_ID:$DATASET"
```
6. The lab resources are billable on some projects. Cloud SQL, Composer, Data Fusion, Dataflow, Bigtable, Vertex AI and KMS may create charges that continue until stopped. Get approval and use a dedicated project or uniquely named resources.

## Part A — Baseline SQL, exact steps
1. In Console search for **BigQuery**, open **BigQuery Studio**, expand **Explorer → your project → `pde_d3_l1`**.
2. Click **SQL query**, open `Lab-3.1/queries.sql` from this ZIP or repository, and copy its contents to the SQL editor.
3. Replace SQL identifier `DATASET` with **`PROJECT_ID.pde_d3_l1`**, replacing `PROJECT_ID` with your actual project ID. Replace any `BUCKET` with an actual existing bucket name if the SQL uses one. Do **not** replace `DATASET` with the bare dataset name in a project you cannot select implicitly.
4. Read the SQL and intended output before clicking **Run**. If it contains multiple statements, confirm each statement has a green job status; check **View results** for the final SELECT.
5. Run the supplemental checks below if relevant. Record actual rows and screenshot. A successful CREATE statement alone is not enough.

### Original SQL exercise
```sql
-- Replace DATASET with pde_d3_l1 (and BUCKET / PROJECT_ID as needed).
-- A runnable federated-query analogue: query Google-hosted BigQuery data without copying it.
CREATE OR REPLACE VIEW `DATASET.federated_demo` AS
SELECT word, word_count FROM `bigquery-public-data.samples.shakespeare` LIMIT 100;
SELECT * FROM `DATASET.federated_demo` LIMIT 5;
```

### What passes Part A?
Actual external query returns two source rows and 300 sum. If Cloud SQL permission denied, mark full-service test BLOCKED.. Where the named service is Dataflow, Cloud SQL, Composer, Catalog, Spark, Bigtable or Gemini, Part A remains an independent teaching demonstration only.

## Part B — Real practical, step by step
1. In Cloud Console open **Cloud SQL → Instances**. If `cloudsql.instances.list` is denied, stop Part B and request Cloud SQL Viewer access. Do not assume the list is empty.
2. Reuse an approved PostgreSQL instance. Otherwise obtain explicit billing approval and Cloud SQL provisioning IAM; create a PostgreSQL instance (choose an approved region, set administrator password securely). Note its **instance connection name** `PROJECT:REGION:INSTANCE`.
3. In **Cloud SQL Studio**, select the `postgres` database, authenticate, and execute:
```sql
CREATE TABLE IF NOT EXISTS public.orders(id INT PRIMARY KEY, amount INT);
INSERT INTO public.orders VALUES (1,100),(2,200)
ON CONFLICT(id) DO UPDATE SET amount=EXCLUDED.amount;
SELECT * FROM public.orders ORDER BY id;
```
4. Enable `bigqueryconnection.googleapis.com` with administrator authorization. In **BigQuery → Add data → Connections to external data sources**, choose **Cloud SQL**, engine **PostgreSQL**, location compatible with the instance and your data needs, and connection ID `pde_cloudsql`. Provide the approved database name, user and password in the protected connection form, not a Git file.
5. Inspect the new connection's service account. Admin grants it Cloud SQL access (normally `roles/cloudsql.client`) and grants the querying learner `roles/bigquery.connectionUser` on this connection as appropriate. Avoid blanket project Owner privileges.
6. From BigQuery SQL editor run, replacing `REGION` with the **BigQuery connection location** (e.g. `us` for US):
```sql
SELECT id, amount FROM EXTERNAL_QUERY(
  'PROJECT_ID.REGION.pde_cloudsql',
  'SELECT id, amount FROM public.orders') ORDER BY id;
SELECT SUM(amount) AS total_amount FROM EXTERNAL_QUERY(
  'PROJECT_ID.REGION.pde_cloudsql',
  'SELECT id, amount FROM public.orders');
```
7. Confirm `(1,100),(2,200)` and `total_amount=300`. This, not the baseline view, is the federation acceptance test.

## Troubleshooting and permission decisions
| Observation | Meaning and fix |
|---|---|
| `PERMISSION_DENIED` enabling an API or making a resource | Ask client administrator for minimal necessary role; don't try a different identity or bypass IAM. |
| `Not found: Dataset` | Verify project, BigQuery location and dataset name; create dataset only with permission. |
| Resource or BigQuery connection location mismatch | Keep service locations and BigQuery connection/dataset locations compatible. |
| Query succeeds but no rows appear downstream | Check that the **actual service** wrote the destination; inspect job/subscription/pipeline logs and schema. |
| Cost, quota or policy restriction | Mark Part B **BLOCKED** with screenshot and ask client for approved environment/resources. |
| Tutorial console labels differ | Use the current service-specific documentation and the search box. Do not infer success from a generic placeholder UI. |

## Proof of completion
- [ ] Project ID and intended service location recorded.
- [ ] Part A query/job ID and result screenshot saved.
- [ ] Part B service was actually used and its job/connection/scan/model identifier captured.
- [ ] Part B verified against acceptance criterion above (or explicitly **BLOCKED/NOT RUN**).
- [ ] Cost-bearing resources stopped after instructor confirms it is safe.

## Safe cleanup
1. Stop active streaming/Dataflow jobs, delete lab-specific instances/subscriptions/pipelines only if **you created them**; never delete shared client resources.
2. After saving evidence, optionally remove only this dedicated dataset:
```bash
bq rm -r -f -d "$PROJECT_ID:pde_d3_l1"
```
3. Remove created GCS objects/buckets and other service resources separately. Composer/Data Fusion/Cloud SQL/Bigtable resources are **not** deleted by the BigQuery cleanup command. KMS key rings cannot be deleted.

## Instructor note
This lab has **not** been executed in the client's live GCP environment by the author. Cloud Console screens and service configurations may vary by region, permissions, and product release. Record verified outcomes explicitly and never claim a managed-service pass based on SQL-only results.

[Google Cloud documentation](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries)
