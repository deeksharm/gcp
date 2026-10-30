# Day 6 · Lab 6.1 — Service Account IAM and Least Privilege

**Objective:** Read-only IAM audit plus isolated object-permission verification.

## Project and security prerequisites

| Setting | Exact value |
|---|---|
| Project | `alchemyso-1727076968159` |
| Region | `us-central1` (not multi-region `US`) |
| Client service account | `nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com` |
| Cloud Shell | Existing authorized human account with service-account impersonation |

**Protect the client:** never delete, disable, or change IAM bindings on the client-provided service account. Never alter organization policies, unrelated datasets, buckets, or infrastructure. Console uses your signed-in human user; Cloud Shell impersonation is separate. Billing and API activation must be approved.

### One-time Cloud Shell session setup

```bash
export PROJECT_ID="alchemyso-1727076968159"
export REGION="us-central1"
export SA_EMAIL="nl-sa-1251137@alchemyso-1727076968159.iam.gserviceaccount.com"
gcloud auth list
gcloud auth print-access-token --impersonate-service-account="$SA_EMAIL" >/dev/null && echo IMPERSONATION_OK
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
```

Stop on failed impersonation. Source verification here is from vendor documentation; **no live GCP execution has been performed for these new labs**.

## Access requirements

Service Accounts Viewer for UI audit, Storage permissions for lab bucket. Role assignment is intentionally **not** part of this lab.

## Step 01 — Audit identities and access

### Google Cloud Console UI
1. Open IAM & Admin → IAM and review project principals (read-only).
2. Open IAM & Admin → Service Accounts and locate nl-sa-1251137; do not edit.
3. If the browser shows Permission Denied, record it; Console human access and CLI impersonation differ.

### Cloud Shell CLI (alternative)

```bash
gcloud config get-value account
gcloud config get-value auth/impersonate_service_account
gcloud iam service-accounts list --project="$PROJECT_ID" --format='table(email,disabled)'
```

**Verification:** Account list or precisely identified missing permission.

**If blocked:** iam.serviceAccounts.list may be denied; do not try self-granting project IAM.

## Step 02 — Create isolated least-privilege storage test

### Google Cloud Console UI
1. Open Cloud Storage → Buckets → Create.
2. Bucket name: `alchemyso-1727076968159-d6-iam-8159`; region `us-central1`; uniform bucket-level access ON; public access prevention enforced.
3. Do not create if exact bucket already exists; verify its owner first.

### Cloud Shell CLI (alternative)

```bash
export BUCKET="${PROJECT_ID}-d6-iam-8159"
gcloud storage buckets list --project="$PROJECT_ID" --filter="name:$BUCKET"
# Run ONLY if it is absent and client approved the new bucket:
gcloud storage buckets create "gs://$BUCKET" --project="$PROJECT_ID" --location="$REGION" --uniform-bucket-level-access --public-access-prevention
```

**Verification:** Bucket is in US-CENTRAL1.

## Step 03 — Verify object read/write operations

### Google Cloud Console UI
1. Open new bucket → Objects → Upload file iam-check.txt containing `day6-iam-ok`.
2. Open object details to verify it was created.

### Cloud Shell CLI (alternative)

```bash
printf 'day6-iam-ok\n' > "$HOME/iam-check.txt"
gcloud storage cp "$HOME/iam-check.txt" "gs://$BUCKET/iam-check.txt"
gcloud storage cat "gs://$BUCKET/iam-check.txt"
```

**Verification:** Text prints `day6-iam-ok`.

## Step 04 — Inspect bucket IAM without changing it

### Google Cloud Console UI
1. Bucket → Permissions → inspect current access.
2. Do NOT click Add principal or change roles; this is a least-privilege audit.

### Cloud Shell CLI (alternative)

```bash
gcloud storage buckets get-iam-policy "gs://$BUCKET" --format=json
```

**Verification:** Read succeeds or missing bucket-policy permission is documented.

## Step 05 — Safe cleanup

### Google Cloud Console UI
1. Delete only iam-check.txt.
2. Delete lab bucket only if created by you and empty; preserve client service account.

### Cloud Shell CLI (alternative)

```bash
gcloud storage rm "gs://$BUCKET/iam-check.txt"
# Confirm bucket has no other objects before OPTIONAL deletion:
gcloud storage ls "gs://$BUCKET/"
# Optional, only for the new lab-owned empty bucket:
gcloud storage buckets delete "gs://$BUCKET"
```

**Verification:** No IAM bindings or identities changed.

## Documentation

- https://cloud.google.com/iam/docs/service-account-impersonation
- https://cloud.google.com/storage/docs/access-control/iam
