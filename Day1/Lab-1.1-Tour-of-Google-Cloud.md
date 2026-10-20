# Lab 1.1: A Tour of Google Cloud

**Day 1 · Foundations & Introduction to Data Engineering**

| Item | Details |
|---|---|
| **Official lab** | [A Tour of Google Cloud Hands-on Labs (GSP282)](https://www.skills.google/focuses/2794) |
| **Adaptation** | The official lab's Pub/Sub task is replaced with a **Cloud Storage** task, which matches today's topics. Pub/Sub is covered on Day 7 |
| **Level** | Introductory |
| **Duration** | ~35 minutes |
| **Services** | Cloud Console, IAM, API Library, Cloud Storage, Cloud Shell |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

---

## 🎯 Objectives
1. Sign in to the Cloud Console with your lab credentials
2. Identify your project's **name, ID and number**
3. Explore Google Cloud services from the **Navigation menu**
4. Review **basic and predefined IAM roles**
5. Enable an API from the **API Library**
6. Create your first resource: a **Cloud Storage bucket** (the landing zone of your data lake)
7. Manage Cloud Storage from **Cloud Shell** (lifecycle rules and versioning)

---

## Task 1. Sign in to the Cloud Console
1. Open the lab portal and start your lab (see the [Lab Access Guide](../README.md)).
2. Click **signinLink** and sign in with the **loginId** and **loginpassword** from the left panel.
3. Accept the terms if prompted. Don't add recovery options or start a free trial.

> 💡 Use an **Incognito window** so your personal Google account doesn't interfere.

✅ **Check:** the Google Cloud Console dashboard is open.

---

## Task 2. View your project
1. On the dashboard, find the **Project info** card.
2. Note the three identifiers:

| Identifier | Description |
|---|---|
| **Project name** | A friendly display name; can be changed |
| **Project ID** | Globally unique and permanent; used in commands and APIs |
| **Project number** | A unique number assigned automatically by Google |

3. Click the **project selector** drop-down in the top bar → **All** to see every project you can access.
4. Click **Cancel** to close the dialog.

> ⚠️ Always check that your **lab project** is selected before creating resources.

---

## Task 3. Explore the Navigation menu
1. Click the **Navigation menu (☰)** at the top left.
2. Click **View all products** and scroll through the categories: Compute, Storage, Databases, Analytics, AI/ML, Networking and more.
3. **Pin** these products for quick access during the training (hover → pin icon):
   - BigQuery
   - Cloud Storage
   - IAM & Admin
   - Datastream
   - Dataflow

---

## Task 4. Review roles and permissions
1. Go to **Navigation menu → IAM & Admin → IAM**.
2. Find your **lab login ID** in the **Principal** column.
3. Note the **roles** assigned to you.

---

## Task 5. Enable an API
Every Google Cloud service is driven by an API that must be **enabled** in the project before it can be used.

1. Go to **Navigation menu → APIs & Services → Library**.
2. Search for **Dialogflow API** and open it.
3. Click **Enable**.
4. After it's enabled, open **APIs & Services → Enabled APIs & services** and check that Dialogflow appears in the list.
5. While you're here, also check that the **BigQuery API** and **Cloud Storage API** are enabled.

✅ **Check:** Dialogflow API shows as **Enabled**.

---

## Task 6. Create a Cloud Storage bucket (Console)
Cloud Storage is Google Cloud's **object storage**. Data engineers use it as the **landing zone** of a data lake: raw files (CSV, JSON, Parquet, Avro) land here before they're loaded into BigQuery or processed by pipelines.

**6.1 Download the sample file**
Download this file to your computer (it's reused in Lab 1.3):
**[nyc_tlc_yellow_trips_2018_subset_1.csv](https://storage.googleapis.com/spls/gsp865/OCBL013/nyc_tlc_yellow_trips_2018_subset_1.csv)**

**6.2 Create the bucket**
1. Go to **Navigation menu → Cloud Storage → Buckets** and click **Create**.
2. Fill in:

| Setting | Value |
|---|---|
| Name | `<PROJECT_ID>-raw` (for example `alchemyso-1726588148343-raw`) |
| Location type | **Multi-region → US** (the same location as our BigQuery datasets) |
| Storage class | **Standard** |
| Access control | **Uniform** |
| Prevent public access | ✅ Enforced |
**don't select hierarchical namespace as its not support versioning**

3. Click **Create**.

**6.3 Upload the file**
1. Open the bucket → **Create folder** → name it `nyctaxi` → **Create**.
2. Open the `nyctaxi` folder → **Upload → Upload files** → select the CSV you downloaded.
3. Click the uploaded object and note its **size**, **storage class** and **gsutil URI** (`gs://...`).

✅ **Check:** `nyctaxi/nyc_tlc_yellow_trips_2018_subset_1.csv` is in your bucket.

---

## Task 7. Manage Cloud Storage from Cloud Shell
1. Click **Activate Cloud Shell** (top right) and click **Continue / Authorize** if prompted.

**7.1 Set up variables**
```bash
gcloud auth list
gcloud config list project

export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1        # use a region allowed in your lab Policies
export BUCKET=gs://${PROJECT_ID}-raw
echo "Project: $PROJECT_ID | Bucket: $BUCKET"
```

**7.2 List buckets and objects**
```bash
gcloud storage ls
gcloud storage ls -l ${BUCKET}/nyctaxi/
gcloud storage buckets describe ${BUCKET} --format="table(name,location,default_storage_class)"
```

**7.3 Add a lifecycle rule (cost optimisation)**
Move objects to **Nearline** after 30 days and **delete** them after 365 days:
```bash
cat > lifecycle.json <<EOF
{
  "rule": [
    {"action": {"type": "SetStorageClass", "storageClass": "NEARLINE"}, "condition": {"age": 30}},
    {"action": {"type": "Delete"}, "condition": {"age": 365}}
  ]
}
EOF

gcloud storage buckets update ${BUCKET} --lifecycle-file=lifecycle.json
```
In the Console, open the bucket → **Lifecycle** tab to see the two rules.

**7.4 Turn on object versioning (protection against overwrites and deletes)**
```bash
gcloud storage buckets update ${BUCKET} --versioning

# Create a small file, upload it, then overwrite it
echo "version 1" > notes.txt
gcloud storage cp notes.txt ${BUCKET}/notes.txt
echo "version 2" > notes.txt
gcloud storage cp notes.txt ${BUCKET}/notes.txt

# List all versions; you should see two
gcloud storage ls --all-versions ${BUCKET}/notes.txt

# Read the current version
gcloud storage cat ${BUCKET}/notes.txt
```

> 💡 To restore the old version, copy it using its full name with `#<generation>`:
> `gcloud storage cp ${BUCKET}/notes.txt#<generation> ${BUCKET}/notes.txt`

✅ **Check:** the bucket shows the lifecycle rules, and `notes.txt` has 2 versions.

---

## 🔁 Knowledge Check
1. Which project identifier can **never** be changed and must be globally unique? → **Project ID**
2. Which role type follows the principle of least privilege? → **Predefined (or custom) roles**
3. What must be done before using a Google Cloud service in a project? → **Enable its API**
4. Data is read often for 30 days, then rarely. How do you cut cost automatically? → A **lifecycle rule** that moves it to Nearline or Coldline
5. How do you protect objects from accidental overwrite or deletion? → **Object versioning**

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| "Bucket name not available" | Bucket names are global; add a suffix, for example `<PROJECT_ID>-raw-01`, and use that name in later labs |
| Can't enable an API | Check **Policies** in the lab panel; tell the trainer |
| Can't download the sample CSV | Upload any small CSV file instead; Lab 1.3 has a fallback dataset |
| Cloud Shell asks you to authorize | Click **Authorize** |
| Wrong project selected | Use the project selector to switch to the lab project |

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 1 Overview](README.md) | ➡️ [Lab 1.2](Lab-1.2-IAM-Custom-Roles.md)
