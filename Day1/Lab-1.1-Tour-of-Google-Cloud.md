# Lab 1.1: A Tour of Google Cloud

**Day 1 · Foundations & Introduction to Data Engineering**

| Item | Details |
|---|---|
| **Official lab** | [A Tour of Google Cloud Hands-on Labs (GSP282)](https://www.skills.google/focuses/2794) |
| **Level** | Introductory |
| **Duration** | ~30 minutes |
| **Services** | Cloud Console, IAM, API Library, Pub/Sub, Cloud Shell |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

---

## 🎯 Objectives
1. Sign in to the Cloud Console with your lab credentials
2. Identify your project's **name, ID and number**
3. Explore Google Cloud services from the **Navigation menu**
4. Review **basic and predefined IAM roles**
5. Enable an API from the **API Library**
6. Create your first resource: a **Pub/Sub topic**
7. Run basic **Cloud Shell** commands

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
   - Pub/Sub
   - Dataflow

---

## Task 4. Review roles and permissions
1. Go to **Navigation menu → IAM & Admin → IAM**.
2. Find your **lab login ID** in the **Principal** column.
3. Note the **roles** assigned to you.

| Role type | Example | Scope |
|---|---|---|
| **Basic** | `roles/viewer`, `roles/editor`, `roles/owner` | Broad access across all services in the project |
| **Predefined** | `roles/pubsub.admin`, `roles/bigquery.dataViewer` | Fine-grained access to one service (least privilege) |
| **Custom** | Created by you | A hand-picked set of permissions |

4. Click **+ Grant access** (don't save) to see how a role is granted to a principal, then **Cancel**.

> 📝 **Discuss:** why is giving everyone `Editor` a bad practice in production?

---

## Task 5. Enable an API
Every Google Cloud service is driven by an API that must be **enabled** in the project before it can be used.

1. Go to **Navigation menu → APIs & Services → Library**.
2. Search for **Dialogflow API** and open it.
3. Click **Enable**.
4. After it's enabled, open **APIs & Services → Enabled APIs & services** and check that Dialogflow appears in the list.
5. While you're here, also check that the **BigQuery API** and **Pub/Sub API** are enabled.

✅ **Check:** Dialogflow API shows as **Enabled**.

---

## Task 6. Create a Pub/Sub topic
1. Go to **Navigation menu → Pub/Sub → Topics**.
2. Click **Create topic**.
3. Topic ID: `day1-first-topic`
4. Leave the defaults (keep **Add a default subscription** checked) and click **Create**.
5. Open the topic and check that a subscription named `day1-first-topic-sub` was created.

✅ **Check:** the topic `day1-first-topic` is listed under **Topics**.

---

## Task 7. Use Cloud Shell
1. Click **Activate Cloud Shell** (the terminal icon at the top right) and click **Continue / Authorize** if prompted.
2. Run the following commands:

```bash
# Show the account you are signed in with
gcloud auth list

# Show the active project
gcloud config list project

# Store values you will reuse in every lab
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1        # use a region allowed in your lab Policies
echo "Project: $PROJECT_ID | Region: $REGION"

# Do the same tasks as above from the command line
gcloud services list --enabled --filter="name:dialogflow OR name:pubsub OR name:bigquery"
gcloud pubsub topics list
gcloud pubsub topics create day1-cli-topic
gcloud pubsub topics list
```

✅ **Check:** both `day1-first-topic` and `day1-cli-topic` appear in the list.

---

## 🧹 Clean Up
```bash
gcloud pubsub subscriptions delete day1-first-topic-sub --quiet
gcloud pubsub topics delete day1-first-topic day1-cli-topic --quiet
```

---

## 🔁 Knowledge Check
1. Which project identifier can **never** be changed and must be globally unique? → **Project ID**
2. Which role type follows the principle of least privilege? → **Predefined (or custom) roles**
3. What must be done before using a Google Cloud service in a project? → **Enable its API**

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| Can't enable an API | Check **Policies** in the lab panel; tell the trainer |
| Cloud Shell asks you to authorize | Click **Authorize** |
| Wrong project selected | Use the project selector to switch to the lab project |

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 1 Overview](README.md) | ➡️ [Lab 1.2](Lab-1.2-IAM-Custom-Roles.md)
