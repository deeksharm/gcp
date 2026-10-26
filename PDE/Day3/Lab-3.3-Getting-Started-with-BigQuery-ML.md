# Lab 3.3: BigQuery ML – Predict Customer Conversion with SQL

**Day 3 · Build Data Lakes and Data Warehouses on Google Cloud (Labs and Best Practices)**

| Item | Details |
|---|---|
| **Related official lab** | [Getting Started with BigQuery ML](https://www.skills.google/paths/16/course_templates/54) |
| **Region** | `us-central1 (Iowa)` |
| **Services** | BigQuery, BigQuery ML |
| **Duration** | ~30 minutes |
| **How to use this guide** | Each step has a **🖱️ Console (UI)** option first and a **⌨️ Cloud Shell (CLI)** alternative. The SQL uses `dataset.table` names, so **no project ID is needed** in the editor |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ Never delete or modify the client-provided service account (`nl-sa-…`), the project, shared IAM bindings, unrelated datasets or existing connections.

---

## 💡 What We're Trying to Achieve
**BigQuery ML** lets you **train and use machine learning models with plain SQL**, inside BigQuery, without exporting data or writing Python.

In this lab we:
1. **Generate** a small, synthetic customer dataset (page views, visit minutes, device, and whether they converted)
2. **Train** a **logistic regression** model (binary classification: will this customer convert, 1 or 0?)
3. **Evaluate** the model (accuracy, precision, recall…)
4. **Predict** conversion for **new** customers, with probabilities

```
customer_training (300 rows) ──CREATE MODEL──► conversion_logreg ──ML.EVALUATE──► metrics
                                                      │
prediction_inputs (3 new customers) ──ML.PREDICT──────┘──► predicted class + probability
```

| Term | Meaning |
|---|---|
| **Label** | What we predict: `converted` (1 = yes, 0 = no) |
| **Features** | Inputs: `page_views`, `visit_minutes` (numeric), `device` (categorical) |
| **Logistic regression** | A model for yes/no predictions; outputs a **probability** between 0 and 1 |
| **Threshold** | Probability ≥ 0.5 → predicted class 1 (by default) |

**✅ Expected results:** model created; `ML.EVALUATE` returns metrics; `ML.PREDICT` returns 3 predictions with probabilities between 0 and 1.

### Values used in this lab

| Item | Value |
|---|---|
| Region | `us-central1 (Iowa)` |
| Dataset | `pde_d3_l3` |
| Training table | `customer_training` |
| Model | `conversion_logreg` |
| New customers table | `prediction_inputs` |

> ⚠️ `${...}` / `$VARIABLE` values only work in **Cloud Shell**. The BigQuery editor SQL in this lab doesn't need them.

---

## Step 1. Select your project (and set CLI variables, optional)

**🖱️ Console**
1. Open [https://console.cloud.google.com/](https://console.cloud.google.com/) and select your lab project in the **project picker**.
2. Go to **BigQuery → Studio**.

**⌨️ CLI alternative:** run once in Cloud Shell. All values are detected automatically:
```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "❌ No project set. Run: gcloud config set project <YOUR_PROJECT_ID>"
fi
export REGION="us-central1"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)
export DATASET="pde_d3_l3"
export TRAIN_TABLE="customer_training"
export MODEL="conversion_logreg"
export PRED_TABLE="prediction_inputs"

echo "PROJECT_ID=$PROJECT_ID | SA_EMAIL=$SA_EMAIL"

# Run CLI commands as the lab service account
gcloud config set auth/impersonate_service_account "$SA_EMAIL"
```

✅ **Check:** the project and service account values are filled in.

> 💡 If Cloud Shell reconnects, variables are lost; re-run this block.

---

## Step 2. Check the BigQuery API

**🖱️ Console:** **APIs & Services → Enabled APIs & services** → check that **BigQuery API** is listed.

**⌨️ CLI alternative**
```bash
gcloud services list --enabled --format='value(config.name)' | grep '^bigquery.googleapis.com$'
```

> 💡 No other API or connection is needed; BigQuery ML logistic regression runs inside BigQuery. BigQuery ML training is **billable** (small for this lab).

---

## Step 3. Create the dataset

**🖱️ Console**
1. **BigQuery → Explorer → ⋮** next to your project → **Create dataset**.
2. Dataset ID: **`pde_d3_l3`**
3. Location type: **Region → us-central1 (Iowa)**
4. Click **Create dataset**.

**⌨️ CLI alternative**
```bash
bq --location="$REGION" mk --dataset "$PROJECT_ID:$DATASET"
```

✅ **Check:** `pde_d3_l3` exists in **us-central1**.

---

## Step 4. Generate the training data (synthetic, with SQL)
The data is generated entirely in BigQuery, with no files or buckets. It's repeatable, and it's learnable but not perfectly separable.

**🖱️ Console:** open **BigQuery Studio → + (new query tab)**, paste and **Run**:
```sql
CREATE OR REPLACE TABLE pde_d3_l3.customer_training AS
SELECT
  customer_id,
  CAST(MOD(customer_id * 7, 100) AS FLOAT64)  AS page_views,
  CAST(MOD(customer_id * 13, 30) AS FLOAT64)  AS visit_minutes,
  IF(MOD(customer_id, 3) = 0, 'mobile', 'desktop') AS device,
  IF(MOD(customer_id * 7, 100) + MOD(customer_id * 13, 30) * 2
     + IF(MOD(customer_id, 3) = 0, -5, 5) > 75, 1, 0) AS converted
FROM UNNEST(GENERATE_ARRAY(1, 300)) AS customer_id;
```

Check it:
```sql
SELECT COUNT(*) AS n, SUM(converted) AS positive_cases
FROM pde_d3_l3.customer_training;
```

Preview the data (optional):
```sql
SELECT * FROM pde_d3_l3.customer_training ORDER BY customer_id LIMIT 10;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.$TRAIN_TABLE\` AS
SELECT customer_id,
  CAST(MOD(customer_id * 7, 100) AS FLOAT64) AS page_views,
  CAST(MOD(customer_id * 13, 30) AS FLOAT64) AS visit_minutes,
  IF(MOD(customer_id, 3)=0, 'mobile', 'desktop') AS device,
  IF(MOD(customer_id*7,100) + MOD(customer_id*13,30)*2 + IF(MOD(customer_id,3)=0,-5,5) > 75, 1, 0) AS converted
FROM UNNEST(GENERATE_ARRAY(1,300)) AS customer_id"

bq --location="$REGION" query --use_legacy_sql=false \
"SELECT COUNT(*) AS n, SUM(converted) AS positive_cases FROM \`$PROJECT_ID.$DATASET.$TRAIN_TABLE\`"
```

✅ **Check:** **n = 300**, and `positive_cases` is between 1 and 299 (both classes exist).

---

## Step 5. Train the logistic regression model

**🖱️ Console:** run in BigQuery Studio:
```sql
CREATE OR REPLACE MODEL pde_d3_l3.conversion_logreg
OPTIONS (
  model_type        = 'LOGISTIC_REG',
  input_label_cols  = ['converted'],
  data_split_method = 'NO_SPLIT',
  max_iterations    = 20
) AS
SELECT page_views, visit_minutes, device, converted
FROM pde_d3_l3.customer_training;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE OR REPLACE MODEL \`$PROJECT_ID.$DATASET.$MODEL\`
 OPTIONS(model_type='LOGISTIC_REG', input_label_cols=['converted'],
         data_split_method='NO_SPLIT', max_iterations=20) AS
 SELECT page_views, visit_minutes, device, converted
 FROM \`$PROJECT_ID.$DATASET.$TRAIN_TABLE\`"
```

> ⏱️ Training takes **1–3 minutes**.

| Option | Meaning |
|---|---|
| `model_type='LOGISTIC_REG'` | Binary classification (yes/no) |
| `input_label_cols=['converted']` | The column to predict |
| `data_split_method='NO_SPLIT'` | Use all 300 rows for training (fine for a demo; real projects keep a test set) |
| `max_iterations=20` | Upper limit on training rounds |

✅ **Check:** in **Explorer → `pde_d3_l3` → Models**, `conversion_logreg` appears. Open it and look at the **Details** and **Training** tabs (loss per iteration).

---

## Step 6. Evaluate the model

**🖱️ Console:** run in BigQuery Studio:
```sql
SELECT precision, recall, accuracy, f1_score, log_loss, roc_auc
FROM ML.EVALUATE(
  MODEL pde_d3_l3.conversion_logreg,
  (SELECT page_views, visit_minutes, device, converted
   FROM pde_d3_l3.customer_training)
);
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"SELECT precision, recall, accuracy, f1_score, log_loss, roc_auc
 FROM ML.EVALUATE(MODEL \`$PROJECT_ID.$DATASET.$MODEL\`,
   (SELECT page_views, visit_minutes, device, converted FROM \`$PROJECT_ID.$DATASET.$TRAIN_TABLE\`))"
```

✅ **Check:** one row of numeric metrics. The values depend on the trained model.

| Metric | Meaning |
|---|---|
| **Accuracy** | Share of all predictions that were correct |
| **Precision** | Of the customers predicted to convert, how many did |
| **Recall** | Of the customers who converted, how many we caught |
| **F1 score** | Balance of precision and recall |
| **Log loss** | Lower is better (confidence-weighted error) |
| **ROC AUC** | 0.5 = random, 1.0 = perfect ranking |

> ⚠️ These metrics are measured on the **training data**, so they're optimistic. Real projects evaluate on **unseen** data.

---

## Step 7. Create new customers to score

**🖱️ Console:** run in BigQuery Studio:
```sql
CREATE OR REPLACE TABLE pde_d3_l3.prediction_inputs AS
SELECT 901 AS customer_id, 8.0  AS page_views, 2.0  AS visit_minutes, 'mobile'  AS device UNION ALL
SELECT 902,                85.0,               24.0,                  'desktop'           UNION ALL
SELECT 903,                50.0,               15.0,                  'mobile';
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.$PRED_TABLE\` AS
 SELECT 901 AS customer_id, 8.0 AS page_views, 2.0 AS visit_minutes, 'mobile' AS device UNION ALL
 SELECT 902, 85.0, 24.0, 'desktop' UNION ALL
 SELECT 903, 50.0, 15.0, 'mobile'"
```

✅ **Check:** `prediction_inputs` has 3 rows. There's **no `converted` column**; that's what the model predicts.

---

## Step 8. Predict with ML.PREDICT

**🖱️ Console:** run in BigQuery Studio:
```sql
SELECT
  customer_id,
  predicted_converted,
  predicted_converted_probs
FROM ML.PREDICT(
  MODEL pde_d3_l3.conversion_logreg,
  (SELECT customer_id, page_views, visit_minutes, device
   FROM pde_d3_l3.prediction_inputs)
)
ORDER BY customer_id;
```

A friendlier version shows the probability of converting as one column:
```sql
SELECT
  customer_id,
  predicted_converted,
  ROUND((SELECT prob FROM UNNEST(predicted_converted_probs) WHERE label = 1), 3) AS prob_convert
FROM ML.PREDICT(
  MODEL pde_d3_l3.conversion_logreg,
  (SELECT customer_id, page_views, visit_minutes, device
   FROM pde_d3_l3.prediction_inputs)
)
ORDER BY customer_id;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"SELECT customer_id, predicted_converted, predicted_converted_probs
 FROM ML.PREDICT(MODEL \`$PROJECT_ID.$DATASET.$MODEL\`,
   (SELECT customer_id, page_views, visit_minutes, device FROM \`$PROJECT_ID.$DATASET.$PRED_TABLE\`))
 ORDER BY customer_id"
```

✅ **Check:** exactly **3 rows** (901, 902, 903). Each `predicted_converted` is **0 or 1**, and the probabilities are between **0 and 1**.

> 📝 **Interpret:** customer **902** (85 page views, 24 minutes, desktop) should have a **high** conversion probability; **901** (8 page views, 2 minutes, mobile) should be **low**.

---

## Step 9 (optional). Training information

**🖱️ Console:** open **`pde_d3_l3` → Models → `conversion_logreg` → Training** tab, or run:
```sql
SELECT *
FROM ML.TRAINING_INFO(MODEL pde_d3_l3.conversion_logreg)
ORDER BY iteration
LIMIT 5;
```

**⌨️ CLI alternative**
```bash
bq --location="$REGION" query --use_legacy_sql=false \
"SELECT * FROM ML.TRAINING_INFO(MODEL \`$PROJECT_ID.$DATASET.$MODEL\`) ORDER BY iteration LIMIT 5"
```

> 📝 **Discuss:** label vs features; numeric vs categorical features (BigQuery ML **one-hot encodes** `device` automatically); probability vs predicted class; the 0.5 decision threshold; and **overfitting** (why training-data accuracy isn't real-world performance).

---

## ✅ Validation Checklist

| # | Check | Pass / Fail |
|---|---|---|
| 1 | Project selected (CLI variables filled in, if used) | |
| 2 | BigQuery API enabled | |
| 3 | Dataset `pde_d3_l3` in **us-central1** | |
| 4 | Training table: **300 rows**, both classes | |
| 5 | Model `conversion_logreg` created (visible under Models) | |
| 6 | `ML.EVALUATE` returns one row of metrics | |
| 7 | `prediction_inputs` has 3 rows | |
| 8 | `ML.PREDICT` returns 3 predictions, probabilities 0–1 | |
| 9 | Training info viewed (optional) | |

---

## 🛠️ Troubleshooting

| Error | Fix |
|---|---|
| `The required property [project] is not currently set` | `gcloud config set project <PROJECT_ID>`, then re-run Step 1 |
| `bigquery.jobs.create` permission denied | Check you're signed in with the lab account and the project is selected; else use the CLI option or ask the trainer |
| `US violates constraint … resourceLocations` | Create the dataset in **us-central1** |
| `Not found: Dataset pde_d3_l3` / location error | Step 3 not done, or the query runs in another location: **More → Query settings → Location → us-central1** |
| Model creation permission denied | The identity needs BigQuery job and model-create permissions (BigQuery Data Editor + Job User); ask the trainer |
| `Already exists: Model` / `Table` | The examples use `CREATE OR REPLACE`; re-run as is |
| `${PROJECT_ID}` errors in the editor | Variables only work in Cloud Shell; the editor SQL in this lab needs no project ID |

---

## 🧹 Clean up resources (after recording results)

**🖱️ Console**
1. **BigQuery → Explorer → `pde_d3_l3` → ⋮ → Delete**.
2. Type `delete` and confirm. This removes the dataset, both tables **and the model**.

**⌨️ CLI alternative** (auto-detects your project)
```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)

# Check what will be deleted
bq --project_id="$PROJECT_ID" ls "$PROJECT_ID:pde_d3_l3"

# Delete the lab dataset, tables and model
bq --project_id="$PROJECT_ID" rm -r -f -d "$PROJECT_ID:pde_d3_l3"

# Confirm it's gone (expect "Not found")
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:pde_d3_l3"

# Stop impersonating the service account
gcloud config unset auth/impersonate_service_account
```

> 🛡️ This deletes **only** `pde_d3_l3`. It doesn't touch the `nl-sa-…` service account, other datasets, connections or IAM bindings.

---

## 🔁 Knowledge Check
1. What does BigQuery ML let you do? → **Train and use ML models with SQL**, inside BigQuery
2. Which model type predicts yes/no outcomes? → **Logistic regression** (`LOGISTIC_REG`)
3. Which function returns accuracy, precision and recall? → **`ML.EVALUATE`**
4. Which function scores new rows? → **`ML.PREDICT`**
5. Why are metrics on training data optimistic? → The model has **already seen** that data (risk of **overfitting**)

---

## 📚 References
- [Create machine learning models in BigQuery ML](https://cloud.google.com/bigquery/docs/create-machine-learning-model)
- [ML.EVALUATE function](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-evaluate)
- [ML.PREDICT function](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-predict)

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab 3.2](Lab-3.2-External-Native-Iceberg-Tables.md) | ➡️ [Day 3 Overview](README.md)
