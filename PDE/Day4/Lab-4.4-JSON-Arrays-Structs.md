# Lab 4.4: Working with JSON, Arrays, and Structs in BigQuery

**Day 4 · Build a Data Warehouse with BigQuery (Skill Badge)**

| Item | Details |
|---|---|
| **Related official lab** | [Working with JSON, Arrays, and Structs in BigQuery](https://www.skills.google/paths/16/course_templates/624) |
| **Region** | `us-central1` (not the US multi-region) |
| **Dataset** | `pde_d4_l4` |
| **Services** | BigQuery, Cloud Shell |
| **Duration** | ~25 minutes |
| **How to use this guide** | **⌨️ Cloud Shell (CLI)** is the main method. A **🖱️ Console** alternative is given: paste the same SQL into BigQuery Studio |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ Never delete, disable or modify the client-provided service account (`nl-sa-…`), project IAM, shared APIs, unrelated datasets or other labs. Clean-up targets only this lab's dataset.

---

## 💡 What We're Trying to Achieve
Real-world data (APIs, apps, event logs) is often **nested**: an order **contains** a customer and **several** line items, plus flexible metadata. BigQuery can store this **without splitting it into many tables**:

| Type | What it is | Example |
|---|---|---|
| **STRUCT** (RECORD) | A group of fields inside one column | `customer = {name: 'Asha', country: 'IN'}` |
| **ARRAY** (REPEATED) | A list of values in one column | `lines = [{pen, 2}, {paper, 3}]` |
| **JSON** | Semi-structured data with a flexible schema | `metadata = {"channel":"web","priority":true}` |

You'll **store** nested data, **query** it without flattening, **flatten** it with `UNNEST`, **extract** JSON fields, **safely parse** bad JSON, and build a **flat reporting table**.

```
orders_nested (1 row: order + customer STRUCT + lines ARRAY + metadata JSON)
        │  UNNEST(lines)
        ▼
order_lines_flat (2 rows: one per line item)  ──►  line_rows = 2, item_quantity = 5
```

**✅ Expected final result:** `order_lines_flat` has **2 rows** and **item_quantity = 5**.

> 💡 The SQL uses `dataset.table` names (no project ID), so the **same SQL works in Cloud Shell and in BigQuery Studio** for every participant.

---

## Step 1. Set up variables (auto-detected)
Open the [BigQuery Console](https://console.cloud.google.com/bigquery), select your lab project, and click **Activate Cloud Shell**. Run:

```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "❌ No project set. Run: gcloud config set project <YOUR_PROJECT_ID>"
fi
export REGION="us-central1"
export DATASET="pde_d4_l4"
export SA_EMAIL=$(gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email~^nl-sa-" --format="value(email)" | head -1)

# Helper: run SQL from Cloud Shell in your project and region
bqq() { bq --project_id="$PROJECT_ID" --location="$REGION" query --use_legacy_sql=false "$@"; }

echo "PROJECT_ID=$PROJECT_ID | REGION=$REGION | DATASET=$DATASET | SA_EMAIL=$SA_EMAIL"
```

**Optional: run as the lab service account** (if your lab uses impersonation):
```bash
gcloud auth print-access-token --impersonate-service-account="$SA_EMAIL" >/dev/null \
  && echo "Impersonation works" \
  && gcloud config set auth/impersonate_service_account "$SA_EMAIL"
```

✅ **Check:** all values are filled in.

> 💡 If Cloud Shell reconnects, variables and the `bqq` helper are lost; re-run this step.
> 💡 Cloud Shell impersonation does **not** apply to BigQuery Studio in the browser; the Console uses your own login.

---

## Step 2. Check BigQuery access and that the dataset doesn't exist yet
```bash
bq --project_id="$PROJECT_ID" ls --datasets
```

✅ **Check:** the command works, and `pde_d4_l4` is **not** listed. If it exists, inspect it rather than overwriting it.

🖱️ **Console alternative:** **BigQuery → Explorer → your project**; check that `pde_d4_l4` isn't there.

---

## Step 3. Create the lab dataset
```bash
bq --project_id="$PROJECT_ID" --location="$REGION" mk --dataset "$PROJECT_ID:$DATASET"
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"
```

✅ **Check:** the dataset's location is **us-central1**.

🖱️ **Console alternative:** **Explorer → ⋮ next to project → Create dataset** → ID `pde_d4_l4` → **Region → us-central1** → **Create dataset**.

> ⚠️ Use **either** the CLI **or** the Console for each step, not both.

---

## Step 4. Create the nested orders table
```bash
bqq <<'SQL'
CREATE OR REPLACE TABLE pde_d4_l4.orders_nested AS
SELECT
  1 AS order_id,
  STRUCT('Asha' AS name, 'IN' AS country)                        AS customer,
  [STRUCT('pen' AS sku, 2 AS quantity),
   STRUCT('paper' AS sku, 3 AS quantity)]                        AS lines,
  PARSE_JSON('{"channel":"web","priority":true}')                AS metadata;
SQL

bq --project_id="$PROJECT_ID" show --schema --format=prettyjson "$PROJECT_ID:pde_d4_l4.orders_nested"
```

✅ **Check:** 1 order row. The schema shows `customer` as **RECORD**, `lines` as **RECORD / REPEATED** (an array of structs), and `metadata` as **JSON**.

🖱️ **Console:** **Explorer → `pde_d4_l4` → `orders_nested` → Schema** tab.

---

## Step 5. Query nested fields without flattening
```bash
bqq <<'SQL'
SELECT order_id,
       customer.name       AS customer_name,
       ARRAY_LENGTH(lines) AS line_count,
       lines
FROM pde_d4_l4.orders_nested;
SQL
```

✅ **Check:** **1 row** with **line_count = 2**.

> 📝 STRUCT fields are read with **dot notation** (`customer.name`). The array stays **inside one row**.

---

## Step 6. Flatten the ARRAY with UNNEST
```bash
bqq <<'SQL'
SELECT t.order_id,
       t.customer.name                     AS customer_name,
       line.sku,
       line.quantity,
       JSON_VALUE(t.metadata, '$.channel') AS channel
FROM pde_d4_l4.orders_nested AS t
CROSS JOIN UNNEST(t.lines) AS line
ORDER BY line.sku;
SQL
```

✅ **Check:** **2 rows**: `paper` (quantity **3**) and `pen` (quantity **2**), both with channel **web**.

> 📝 `UNNEST` turns each array element into its own row. The parent fields (order_id, customer) **repeat** for each line, just like the one-to-many join in Lab 4.3.

---

## Step 7. JSON_VALUE vs JSON_QUERY
```bash
bqq <<'SQL'
SELECT JSON_VALUE(metadata, '$.channel')  AS channel,
       JSON_QUERY(metadata, '$.priority') AS priority_json
FROM pde_d4_l4.orders_nested;
SQL
```

✅ **Check:** **channel = web**; **priority_json = true** (as JSON).

| Function | Returns | Use for |
|---|---|---|
| `JSON_VALUE` | A **scalar string** | Text and numbers to filter or group by |
| `JSON_QUERY` | **JSON** (objects, arrays or values) | Nested objects and arrays |

---

## Step 8. Safely parse invalid JSON
```bash
bqq <<'SQL'
SELECT SAFE.PARSE_JSON('{"valid":1}')    IS NOT NULL AS valid_parsed,
       SAFE.PARSE_JSON('{invalid json}') IS NULL     AS invalid_is_null;
SQL
```

✅ **Check:** **both values are `true`**.

> 💡 `PARSE_JSON` on bad input **fails the whole query**. The `SAFE.` prefix returns **NULL** instead, which is essential when loading messy data. Then count the NULLs to measure data quality.

---

## Step 9. Create a flattened reporting table
```bash
bqq <<'SQL'
CREATE OR REPLACE TABLE pde_d4_l4.order_lines_flat AS
SELECT t.order_id,
       t.customer.name                     AS customer_name,
       line.sku,
       line.quantity,
       JSON_VALUE(t.metadata, '$.channel') AS channel
FROM pde_d4_l4.orders_nested t
CROSS JOIN UNNEST(t.lines) line;
SQL

bqq 'SELECT * FROM pde_d4_l4.order_lines_flat ORDER BY sku'
```

✅ **Check:** the flattened table has **2 line records**.

> 📝 **Nested** tables are efficient for storage and ingestion; **flat** tables are often easier for BI tools and analysts. Data engineers frequently build both.

---

## Step 10. Verify line totals
```bash
bqq 'SELECT COUNT(*) AS line_rows, SUM(quantity) AS item_quantity FROM pde_d4_l4.order_lines_flat'
```

✅ **Check:** **line_rows = 2**, **item_quantity = 5**.

---

## 🖱️ Console Alternative (Steps 4–10)
Every SQL block above also runs in **BigQuery Studio**:
1. **BigQuery → Studio → + (new query tab)**.
2. Paste **only the SQL** (the lines between `bqq <<'SQL'` and `SQL`; or, for single-line `bqq '...'` commands, the text inside the quotes).
3. Click **Run** and compare with the ✅ checks.
4. For created tables: **Query results → Go to table → Schema / Preview**.

> 💡 If you get a location error in the Console: **More → Query settings → Location → us-central1**.

---

## ✅ Validation Checklist

| # | Check | Pass / Fail |
|---|---|---|
| 1 | Variables set; `bqq` helper defined | |
| 2 | BigQuery access works; dataset didn't exist | |
| 3 | `pde_d4_l4` created in **us-central1** | |
| 4 | `orders_nested`: STRUCT, ARRAY&lt;STRUCT&gt; and JSON in the schema | |
| 5 | Nested query: 1 row, line_count = 2 | |
| 6 | UNNEST: 2 rows (paper 3, pen 2), channel web | |
| 7 | JSON_VALUE = web; JSON_QUERY = true | |
| 8 | SAFE.PARSE_JSON: both true | |
| 9 | `order_lines_flat`: 2 rows | |
| 10 | line_rows = 2, item_quantity = 5 | |

```bash
bq --project_id="$PROJECT_ID" ls "$PROJECT_ID:$DATASET"
```

---

## 🛠️ Troubleshooting

| Error | Fix |
|---|---|
| `The required property [project] is not currently set` | `gcloud config set project <PROJECT_ID>`, then re-run Step 1 |
| `bqq: command not found` | Cloud Shell reconnected; re-run Step 1 |
| `PERMISSION_DENIED` / `bigquery.jobs.create` | Check the account (Console user vs impersonated service account) and BigQuery permissions; ask the admin, don't edit IAM |
| `SERVICE_DISABLED` | The admin needs to enable the BigQuery API |
| `US violates constraint … resourceLocations` / location error | Use **us-central1** (CLI: `--location`; Console: **Query settings → Location**) |
| `Values referenced in UNNEST must be arrays` | `UNNEST` only works on ARRAY columns, e.g. `t.lines` |
| `JSON_VALUE` returns NULL | Check the JSON path (`'$.channel'`) and spelling; paths are case-sensitive |
| SQL syntax error in Cloud Shell | Paste SQL only between `bqq <<'SQL'` and `SQL`, or into the BigQuery editor |

---

## 🧹 Clean-up (after all checks and screenshots)
```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)

# Check the dataset contains only this lab's tables
bq --project_id="$PROJECT_ID" ls "$PROJECT_ID:pde_d4_l4"

# Delete the lab dataset and its tables
bq --project_id="$PROJECT_ID" rm -r -f -d "$PROJECT_ID:pde_d4_l4"

# Confirm it's gone
bq --project_id="$PROJECT_ID" ls --datasets

# Stop impersonating the service account (if you set it)
gcloud config unset auth/impersonate_service_account
```

🖱️ **Console alternative:** **Explorer → `pde_d4_l4` → ⋮ → Delete** → type `delete`.

> 🛡️ Never run `gcloud iam service-accounts delete/disable`, `gcloud projects delete`, or clean-up affecting other datasets.

---

## 🔁 Knowledge Check
1. How do you read a field inside a STRUCT? → **Dot notation**, e.g. `customer.name`
2. What does `UNNEST` do? → Turns each **array element into a row**
3. `JSON_VALUE` vs `JSON_QUERY`? → `JSON_VALUE` returns a **scalar string**; `JSON_QUERY` returns **JSON**
4. Why use `SAFE.PARSE_JSON`? → Invalid JSON becomes **NULL** instead of **failing the query**
5. Why keep both nested and flat tables? → Nested is **efficient**; flat is **easy for BI and analysts**

---

## 📚 References
- [Working with JSON data in GoogleSQL](https://cloud.google.com/bigquery/docs/json-data)
- [Work with arrays](https://cloud.google.com/bigquery/docs/arrays)
- [Google Skills – Professional Data Engineer path](https://www.skills.google/paths/16)

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab 4.3](Lab-4.3-Join-Pitfalls.md) | ➡️ [Day 4 Overview](README.md)
