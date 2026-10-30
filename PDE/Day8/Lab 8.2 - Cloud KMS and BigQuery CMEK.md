# Day 6 · Lab 6.2 — Cloud KMS and BigQuery CMEK

**Professional hands-on execution guide | Console UI first · Cloud Shell CLI alternative**

**Objective:** Create a regional Cloud KMS key and a BigQuery table encrypted with a customer-managed encryption key (CMEK); verify both the data and the actual encryption configuration.

> **Choose one method per operation:** Follow the numbered **Google Cloud Console UI** steps. The accompanying **CLI alternative** achieves the same result and is not an additional required action. For verification, running both is safe. **Do not create duplicate resources or insert the sample rows twice.**
>
> **Client protection:** Do not delete or disable `nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com`, change shared/project IAM policies, disable APIs, or touch unrelated resources. Never disable or destroy an active KMS key version. Key rings and keys cannot be deleted in the usual way; obtain client approval before creating them. KMS and BigQuery operations can incur charges.

## Exact lab values

| Field | Value |
|---|---|
| Project ID | `alchemyso-1727076968159` |
| Project number | `1083122553554` |
| Region for KMS, dataset and queries | `us-central1` (NOT `US`) |
| Training service account | `nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com` |
| Key ring | `pde-d6-kms` |
| Cryptographic key | `bq-training-key` |
| Key purpose / protection | Symmetric encryption/decryption; Software |
| BigQuery encryption service agent | `bq-1083122553554@bigquery-encryption.iam.gserviceaccount.com` |
| Dataset | `pde_d6_l2` |
| Table | `secure_orders` |
| Full KMS key name | `projects/alchemyso-1727076968159/locations/us-central1/keyRings/pde-d6-kms/cryptoKeys/bq-training-key` |
| Expected records | `(1,100)` and `(2,200)` |
| Expected aggregate | `rows_count=2`, `total=300` |

## Prerequisites and IAM permissions

1. Client-approved project access and billing; Cloud KMS API `cloudkms.googleapis.com` and BigQuery API `bigquery.googleapis.com` enabled.
2. Training identity: **Cloud KMS Admin** (`roles/cloudkms.admin`) at project level for creating a new key ring and key; **BigQuery Job User** (`roles/bigquery.jobUser`) at project level; **BigQuery Data Editor** (`roles/bigquery.dataEditor`) on lab dataset (plus dataset-creation permission if it does not yet exist).
3. Google-managed **BigQuery encryption service account**, not your training account, needs **Cloud KMS CryptoKey Encrypter/Decrypter** (`roles/cloudkms.cryptoKeyEncrypterDecrypter`) **on `bq-training-key`**.
4. Console actions run as the browser-signed-in human user. The Cloud Shell `gcloud` impersonation setting does **not** apply to the Console; the human user needs separate permissions or an administrator must perform the UI IAM steps.
5. Confirm client approval before creating permanent KMS metadata. Do not grant project-wide IAM in this guide.

## 00 — Prepare Cloud Shell and verify identity

### Google Cloud Console UI — primary

1. Open **https://console.cloud.google.com/** and select project **`alchemyso-1727076968159`** at the top.
2. Click **Activate Cloud Shell** (terminal icon).
3. If you plan to run the CLI alternative, use the commands below to set the exact values and test impersonation.
4. If the Console reports missing project access, ask the administrator to grant the signed-in user the required access. Do not assume Cloud Shell impersonation fixes Console access.

### Cloud Shell CLI — alternative / setup

```bash
export PROJECT_ID="alchemyso-1727076968159"
export REGION="us-central1"
export SA_EMAIL="nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com"
export KEYRING="pde-d6-kms"
export KEY="bq-training-key"
export DATASET="pde_d6_l2"
export BQ_ENCRYPTION_SA="bq-1083122553554@bigquery-encryption.iam.gserviceaccount.com"
export KMS_KEY="projects/$PROJECT_ID/locations/$REGION/keyRings/$KEYRING/cryptoKeys/$KEY"
gcloud auth list
gcloud auth print-access-token --impersonate-service-account="$SA_EMAIL" >/dev/null && echo "IMPERSONATION_OK"
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
```

**Pass check:** `IMPERSONATION_OK` appears. If not, stop; avoid creating JSON service-account keys.

## 01 — Confirm Google Cloud APIs

### Google Cloud Console UI — primary

1. Open **Navigation menu → APIs & Services → Enabled APIs & services**.
2. In project **`alchemyso-1727076968159`**, find **Cloud Key Management Service (KMS) API** (`cloudkms.googleapis.com`).
3. Find **BigQuery API** (`bigquery.googleapis.com`).
4. If either is disabled, ask an authorized administrator to enable it from **APIs & Services → Library**. Do not repeatedly retry failed activations.
5. Confirm both say **Enabled** before continuing.

### Cloud Shell CLI — alternative verification

```bash
gcloud services list --enabled --project="$PROJECT_ID" \
  --format="value(config.name)" | grep -E '^(cloudkms|bigquery)\.googleapis\.com$'
```

**Pass check:** Both API names are listed. If Service Usage listing is denied, the administrator must verify them; a listing failure alone does not prove they are disabled.

## 02 — Create or verify the Cloud KMS key ring

### Google Cloud Console UI — primary

1. Open **Security → Key Management**: https://console.cloud.google.com/security/kms?project=alchemyso-1727076968159 .
2. Check for an existing **`pde-d6-kms`** key ring in **`us-central1`**. If one exists, confirm it belongs to this lab; **reuse it**, do not create a duplicate or touch an unrelated ring.
3. If it does not exist and approval is confirmed, click **Create key ring**.
4. Enter **Key ring name:** `pde-d6-kms`.
5. Select **Location:** `us-central1` (regional; do **not** select `US` or `global`).
6. Click **Create**.
7. Open the new key ring and confirm location **`us-central1`**.

### Cloud Shell CLI — alternative

```bash
# First check; NOT_FOUND means creation is needed.
gcloud kms keyrings describe "$KEYRING" \
  --location="$REGION" --project="$PROJECT_ID"

# Run ONLY if not found and client-approved:
gcloud kms keyrings create "$KEYRING" \
  --location="$REGION" --project="$PROJECT_ID"
```

**Pass check:** The key ring exists in `us-central1`. **Stop on `cloudkms.keyRings.create` denied**; project-scope KMS administration is required.

## 03 — Create or verify the cryptographic key

### Google Cloud Console UI — primary

1. Open key ring **`pde-d6-kms`**.
2. Check whether **`bq-training-key`** already exists. If it belongs to this lab, reuse it.
3. Otherwise click **Create key**.
4. Enter **Key name:** `bq-training-key`.
5. Set **Key type:** Generated key / **Purpose:** Symmetric encrypt/decrypt.
6. Set **Protection level:** Software. Retain the approved default rotation policy unless the client specifies another.
7. Click **Create**.
8. Open the key and verify the primary version is **Enabled**.

### Cloud Shell CLI — alternative

```bash
# Check first.
gcloud kms keys describe "$KEY" \
  --keyring="$KEYRING" --location="$REGION" --project="$PROJECT_ID"

# Run ONLY if the key is absent and approved:
gcloud kms keys create "$KEY" \
  --keyring="$KEYRING" --location="$REGION" \
  --purpose=encryption --protection-level=software \
  --project="$PROJECT_ID"

# Read-only confirmation:
gcloud kms keys describe "$KEY" \
  --keyring="$KEYRING" --location="$REGION" --project="$PROJECT_ID"
```

**Pass check:** The key resource exists at the exact KMS path above, with an enabled primary version. **Do not disable/destroy the key or key versions.**

## 04 — Create or verify the regional BigQuery dataset

### Google Cloud Console UI — primary

1. Open **BigQuery Studio**: https://console.cloud.google.com/bigquery?project=alchemyso-1727076968159 .
2. In Explorer, expand **`alchemyso-1727076968159`** and check whether **`pde_d6_l2`** already exists.
3. If it exists, open dataset **Details** and confirm its location is **`us-central1`** and it belongs to this lab. Do not recreate it.
4. Otherwise choose **View actions (three dots) → Create dataset**.
5. Enter **Dataset ID:** `pde_d6_l2`; **Location type:** Region; **Region:** `us-central1`.
6. Click **Create dataset**.
7. Open **Details** and verify the regional location.

### Cloud Shell CLI — alternative

```bash
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"

# Run only when the lab dataset does not already exist:
bq --project_id="$PROJECT_ID" --location="$REGION" \
  mk --dataset "$PROJECT_ID:$DATASET"

bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"
```

**Pass check:** Dataset `pde_d6_l2` exists in `us-central1`, NOT the `US` multi-region.

## 05 — Grant the BigQuery encryption service account access to the key

### Google Cloud Console UI — primary

1. In **BigQuery**, identify the **BigQuery encryption service account** (or confirm it from the read-only CLI shown below). For this project it is **`bq-1083122553554@bigquery-encryption.iam.gserviceaccount.com`**.
2. Open **Security → Key Management → `pde-d6-kms`**.
3. Select/check **`bq-training-key`** and open the **Permissions** tab for that *key*.
4. Check whether the exact BigQuery encryption service account already has **Cloud KMS CryptoKey Encrypter/Decrypter**. If it does, do not create another binding.
5. If missing and client-approved, click **Grant access / Add principal**.
6. **New principal:** `bq-1083122553554@bigquery-encryption.iam.gserviceaccount.com`.
7. **Role:** `Cloud KMS → Cloud KMS CryptoKey Encrypter/Decrypter` (`roles/cloudkms.cryptoKeyEncrypterDecrypter`).
8. Click **Save**; verify the binding appears on **`bq-training-key`**, not a different key or just the training account.

### Cloud Shell CLI — alternative

```bash
# Discover/confirm the actual encryption service account:
bq show --encryption_service_account --project_id="$PROJECT_ID"

# Read key IAM first; only add the missing binding if authorized:
gcloud kms keys get-iam-policy "$KEY" \
  --keyring="$KEYRING" --location="$REGION" \
  --project="$PROJECT_ID" --format="yaml(bindings)"

# IAM-changing operation — run ONLY if binding is missing and client-approved:
gcloud kms keys add-iam-policy-binding "$KEY" \
  --keyring="$KEYRING" --location="$REGION" \
  --project="$PROJECT_ID" \
  --member="serviceAccount:$BQ_ENCRYPTION_SA" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

**Pass check:** Key IAM lists `bq-1083122553554@bigquery-encryption.iam.gserviceaccount.com` under `roles/cloudkms.cryptoKeyEncrypterDecrypter`. Without it, table creation can fail with `cloudkms.cryptoKeyVersions.useToEncrypt denied`.

## 06 — Create the CMEK-protected table

### Google Cloud Console UI — primary

1. In **BigQuery Studio**, expand project **`alchemyso-1727076968159`** → dataset **`pde_d6_l2`**.
2. Check whether `secure_orders` already exists. If so, open **Details** and confirm it is the intended lab table **and** its customer-managed key is `bq-training-key`. Do not recreate or overwrite it.
3. If absent, click **Create table**.
4. **Create table from:** Empty table.
5. **Project:** `alchemyso-1727076968159`; **Dataset:** `pde_d6_l2`; **Table:** `secure_orders`.
6. Under **Schema**, add `order_id` type **INTEGER** and `amount` type **INTEGER**. Keep both as standard nullable fields for the exercise.
7. Expand **Advanced options** / **Encryption**. Select **Customer-managed encryption key** rather than Google-managed encryption.
8. Choose the exact key `projects/alchemyso-1727076968159/locations/us-central1/keyRings/pde-d6-kms/cryptoKeys/bq-training-key`.
9. Click **Create table**.
10. Open table **Details** and confirm the customer-managed key path is shown.

### Cloud Shell CLI — alternative

```bash
# Check before creating:
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET.secure_orders"

# Run ONLY if the lab table does not exist:
bq --project_id="$PROJECT_ID" --location="$REGION" \
  mk -t \
  --schema="order_id:INTEGER,amount:INTEGER" \
  --destination_kms_key="$KMS_KEY" \
  "$PROJECT_ID:$DATASET.secure_orders"

# Verify table metadata:
bq --project_id="$PROJECT_ID" show --format=prettyjson \
  "$PROJECT_ID:$DATASET.secure_orders" | grep -A4 encryptionConfiguration
```

**Pass check:** The table exists and `kmsKeyName` matches `bq-training-key`. If creation fails, stop; **do not run INSERT on a nonexistent table**.

## 07 — Insert the two sample records (safe for reruns)

### Google Cloud Console UI — primary

1. In **BigQuery Studio → Compose new query**, select processing location **`us-central1`** under Query settings if needed.
2. Paste the following **GoogleSQL**. It inserts missing IDs and updates existing IDs, avoiding duplicate records on reruns.
3. Click **Run** and confirm the query job succeeds.
4. In **Preview** or with the verification SQL in Step 08, confirm exactly two rows exist for this lab.

```sql
MERGE `alchemyso-1727076968159.pde_d6_l2.secure_orders` AS target
USING (
  SELECT 1 AS order_id, 100 AS amount UNION ALL
  SELECT 2 AS order_id, 200 AS amount
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET amount = source.amount
WHEN NOT MATCHED THEN INSERT (order_id, amount)
VALUES (source.order_id, source.amount);
```

### Cloud Shell CLI — alternative

```bash
bq --project_id="$PROJECT_ID" --location="$REGION" \
  query --use_legacy_sql=false <<'SQL'
MERGE `alchemyso-1727076968159.pde_d6_l2.secure_orders` AS target
USING (
  SELECT 1 AS order_id, 100 AS amount UNION ALL
  SELECT 2 AS order_id, 200 AS amount
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET amount = source.amount
WHEN NOT MATCHED THEN INSERT (order_id, amount)
VALUES (source.order_id, source.amount);
SQL
```

**Pass check:** The operation completes without error. For a brand-new empty table, exactly two rows are inserted. If prior unrelated rows exist, stop instead of deleting them.

## 08 — Verify data values and encryption (both required)

### Google Cloud Console UI — primary

1. In **BigQuery Studio**, open **`pde_d6_l2.secure_orders` → Details**.
2. Under **Encryption**, verify **Customer-managed key** equals `bq-training-key` at `us-central1`.
3. Click **Compose new query**, set query location `us-central1`, and run the query below.
4. Confirm `rows_count=2` and `total=300`.
5. Also inspect the individual rows, ensuring IDs 1 and 2 have amounts 100 and 200.
6. Capture evidence: one screenshot of table **Details → Encryption** and one of the BigQuery **Results**.

```sql
SELECT
  COUNT(*) AS rows_count,
  SUM(amount) AS total
FROM `alchemyso-1727076968159.pde_d6_l2.secure_orders`;
```

```sql
SELECT order_id, amount
FROM `alchemyso-1727076968159.pde_d6_l2.secure_orders`
ORDER BY order_id;
```

### Cloud Shell CLI — alternative

```bash
bq --project_id="$PROJECT_ID" --location="$REGION" \
  query --use_legacy_sql=false <<'SQL'
SELECT COUNT(*) AS rows_count, SUM(amount) AS total
FROM `alchemyso-1727076968159.pde_d6_l2.secure_orders`;
SQL

bq --project_id="$PROJECT_ID" show --format=prettyjson \
  "$PROJECT_ID:$DATASET.secure_orders" | grep -A4 encryptionConfiguration
```

**Pass criteria:** `rows_count = 2`, `total = 300`, and actual table metadata names the expected CMEK. An aggregate result by itself does **not** prove CMEK encryption.

## 09 — Troubleshooting / stop conditions

| Error / result | Meaning and next action |
|---|---|
| `cloudkms.keyRings.create` or `cloudkms.cryptoKeys.create` denied | Training identity lacks KMS creation permission; ask authorized admin for project-scope KMS Admin or admin-created approved key. |
| `cloudkms.cryptoKeyVersions.useToEncrypt` denied | Grant exact BigQuery **encryption service account** the key-level Encrypter/Decrypter role; do not grant it only to the training account. |
| `Table ... not found` after failed `mk` | Create and verify CMEK table first; then perform the SQL load. |
| Query shows `rows_count=0`, `total=NULL` | Table exists but no rows; execute Step 07; verify again. |
| Query returns more than two rows | Stop; check whether sample rows were inserted repeatedly or table includes pre-existing data. Do not blindly truncate. |
| `US` violates resource location constraint | Choose **regional `us-central1`** for both KMS and BigQuery. |
| Console permission denied while CLI succeeds | Browser user and Cloud Shell service-account impersonation have different IAM identities; ask admin for appropriate Console access. |
| KMS key ring/key `ALREADY_EXISTS` | Verify exact resource ownership and location, then reuse approved lab-owned resources; never overwrite unrelated keys. |
| BigQuery query fails because key is disabled | Do not attempt to destroy/re-enable keys unilaterally; contact authorized KMS administrator. |

## 10 — Safe cleanup (only after client approval and evidence capture)

### Google Cloud Console UI — primary

1. Confirm that **`pde_d6_l2.secure_orders`** contains only the lab's sample data and capture evidence.
2. BigQuery Studio → open **`secure_orders`** → **Delete**; confirm exact table name.
3. Only if **`pde_d6_l2`** is entirely dedicated to this lab and now empty, select that dataset → **Delete dataset**.
4. **Do not delete, disable, destroy, rotate unexpectedly, or change IAM bindings on** `pde-d6-kms` / `bq-training-key`. KMS metadata persists and may incur charges; discuss lifecycle with client security team.
5. Do not change the client-provided service account, project, shared IAM, unrelated datasets, or APIs.

### Cloud Shell CLI — alternative

```bash
# Read-only checks before choosing any destructive action:
bq --project_id="$PROJECT_ID" ls "$PROJECT_ID:$DATASET"
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET.secure_orders"

# ONLY after explicit lab cleanup approval:
bq --project_id="$PROJECT_ID" rm -f -t \
  "$PROJECT_ID:$DATASET.secure_orders"

# ONLY if the dataset is lab-owned and completely empty:
bq --project_id="$PROJECT_ID" rm -f -d \
  "$PROJECT_ID:$DATASET"
```

**Final evidence checklist:** [ ] KMS key created/enabled · [ ] BigQuery encryption identity has key IAM · [ ] CMEK table metadata confirmed · [ ] Two rows / sum 300 · [ ] Safe cleanup approved or resources retained for review.

## Google Cloud documentation

- [BigQuery: Customer-managed Cloud KMS keys](https://cloud.google.com/bigquery/docs/customer-managed-encryption)
- [Cloud KMS: Create a key ring](https://cloud.google.com/kms/docs/create-key-ring)
- [Cloud KMS: Create a key](https://cloud.google.com/kms/docs/creating-keys)

**Validation scope:** Commands, SQL, and workflow have been reviewed against documentation and the reported lab errors. This file was not executed against your live GCP environment. The actual completion requires the observed verification results in Step 08.
