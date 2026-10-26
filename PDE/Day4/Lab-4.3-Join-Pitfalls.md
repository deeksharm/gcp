# Lab 4.3: Troubleshooting and Solving Data Join Pitfalls

**Day 4 · Build a Data Warehouse with BigQuery (Skill Badge)**

| Item | Details |
|---|---|
| **Related official lab** | [Troubleshooting and Solving Data Join Pitfalls](https://www.skills.google/paths/16/course_templates/624) |
| **Region** | `us-central1` (not the US multi-region) |
| **Dataset** | `pde_d4_l3` |
| **Services** | BigQuery, Cloud Shell |
| **Duration** | ~25 minutes |
| **How to use this guide** | **⌨️ Cloud Shell (CLI)** is the main method. A **🖱️ Console** alternative is given: paste the same SQL into BigQuery Studio |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

> 🛡️ Never delete, disable or modify the client-provided service account (`nl-sa-…`), project IAM, shared APIs, unrelated datasets or other labs. Clean-up targets only this lab's dataset.

---

## 💡 What We're Trying to Achieve
Joins are the most common source of **wrong numbers** in a data warehouse. In this lab you'll see three classic pitfalls, then fix them:

| Pitfall | What goes wrong |
|---|---|
| **One-to-many row multiplication** | Joining a customer to several orders repeats the customer, so totals and counts get **inflated** |
| **NULL join keys** | Orders with a `NULL` customer never match, so they **silently disappear** from joined reports |
| **Missing matches** | A `LEFT JOIN` keeps customers with no orders, and `COUNT(*)` counts them as 1 instead of 0 |
| **The fix** | **Aggregate the fact table before joining**, use `COUNT(column)`, and audit orphan rows |

```
customers (2 rows)  ──LEFT JOIN──►  orders (3 rows: 2 for customer 1, 1 with NULL customer)
          │
          └── fix: aggregate orders first ──► customer_totals (1 row per customer)
```

**✅ Expected final result:** `customer_totals` has **2 customers** with **matched_sales = 300**, and the audit shows **1 orphan order worth 50** (total 350).

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
export DATASET="pde_d4_l3"
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

✅ **Check:** the command works, and `pde_d4_l3` is **not** listed. If it exists, inspect it rather than overwriting it.

🖱️ **Console alternative:** **BigQuery → Explorer → your project**; check that `pde_d4_l3` isn't there.

---

## Step 3. Create the lab dataset
```bash
bq --project_id="$PROJECT_ID" --location="$REGION" mk --dataset "$PROJECT_ID:$DATASET"
bq --project_id="$PROJECT_ID" show "$PROJECT_ID:$DATASET"
```

✅ **Check:** the dataset's location is **us-central1**.

🖱️ **Console alternative:** **Explorer → ⋮ next to project → Create dataset** → ID `pde_d4_l3` → **Region → us-central1** → **Create dataset**.

> ⚠️ Use **either** the CLI **or** the Console for each step, not both.

---

## Step 4. Create the customer dimension
```bash
bqq <<'SQL'
CREATE OR REPLACE TABLE pde_d4_l3.customers AS
SELECT 1 AS id, 'Asha' AS name UNION ALL
SELECT 2, 'Ravi';
SQL

bqq 'SELECT * FROM pde_d4_l3.customers ORDER BY id'
```

✅ **Check:** **2 customer rows** (Asha, Ravi).

---

## Step 5. Create the orders fact table (with data-quality issues)
```bash
bqq <<'SQL'
CREATE OR REPLACE TABLE pde_d4_l3.orders AS
SELECT 101 AS order_id, 1 AS customer_id, 100 AS amount UNION ALL
SELECT 102, 1, 200 UNION ALL
SELECT 103, CAST(NULL AS INT64), 50;
SQL

bqq 'SELECT * FROM pde_d4_l3.orders ORDER BY order_id'
```

✅ **Check:** **3 orders**: two belong to customer 1, and one has a **NULL** customer.

| order_id | customer_id | amount | Note |
|---|---|---|---|
| 101 | 1 | 100 | |
| 102 | 1 | 200 | Customer 1 has **two** orders (one-to-many) |
| 103 | NULL | 50 | **Orphan**: no customer |

---

## Step 6. Observe row multiplication and unmatched customers
```bash
bqq <<'SQL'
SELECT c.id, c.name, o.order_id, o.amount
FROM pde_d4_l3.customers c
LEFT JOIN pde_d4_l3.orders o ON c.id = o.customer_id
ORDER BY c.id, o.order_id;
SQL
```

✅ **Check:** **3 rows**: customer 1 appears **twice**; customer 2 appears once with a **NULL** order.

> 📝 **Discuss:**
> - Customer 1 is **duplicated**, one row per order. Summing a customer-level column here would **double-count**.
> - Order 103 (NULL customer) is **missing**: `NULL = 1` is never true, so it never matches.

---

## Step 7. COUNT(*) vs COUNT(column)
```bash
bqq <<'SQL'
SELECT c.id,
       COUNT(*)          AS joined_rows,
       COUNT(o.order_id) AS real_orders
FROM pde_d4_l3.customers c
LEFT JOIN pde_d4_l3.orders o ON c.id = o.customer_id
GROUP BY c.id
ORDER BY c.id;
SQL
```

✅ **Check:**

| id | joined_rows | real_orders |
|---|---|---|
| 1 | 2 | 2 |
| 2 | **1** | **0** |

> 📝 `COUNT(*)` counts **rows**, so customer 2's empty LEFT JOIN row counts as 1. `COUNT(o.order_id)` ignores NULLs, so it gives the **real** order count, 0.

---

## Step 8. Fix duplication by aggregating before the join
```bash
bqq <<'SQL'
WITH by_customer AS (
  SELECT customer_id, SUM(amount) AS total
  FROM pde_d4_l3.orders
  WHERE customer_id IS NOT NULL
  GROUP BY customer_id
)
SELECT c.id, c.name, COALESCE(o.total, 0) AS total
FROM pde_d4_l3.customers c
LEFT JOIN by_customer o ON c.id = o.customer_id
ORDER BY c.id;
SQL
```

✅ **Check:** exactly **2 rows**: customer 1 total = **300**, customer 2 total = **0**.

> 💡 **Rule:** bring the fact table to the **same grain** as the dimension (one row per customer) **before** joining. `COALESCE` turns "no orders" into 0.

---

## Step 9. Audit unassigned (orphan) orders
```bash
bqq <<'SQL'
SELECT COUNTIF(customer_id IS NULL)          AS orphan_orders,
       SUM(IF(customer_id IS NULL, amount, 0)) AS orphan_amount,
       SUM(amount)                             AS all_order_amount
FROM pde_d4_l3.orders;
SQL
```

✅ **Check:** **orphan_orders = 1**, **orphan_amount = 50**, **all_order_amount = 350**.

> 📝 300 (matched) + 50 (orphan) = 350 (total). Always reconcile: the 50 must be **reported**, not silently lost or wrongly assigned.

---

## Step 10. Build and validate the cleaned summary table
```bash
bqq <<'SQL'
CREATE OR REPLACE TABLE pde_d4_l3.customer_totals AS
WITH agg AS (
  SELECT customer_id, SUM(amount) AS total
  FROM pde_d4_l3.orders
  WHERE customer_id IS NOT NULL
  GROUP BY customer_id
)
SELECT c.id, c.name, COALESCE(agg.total, 0) AS total
FROM pde_d4_l3.customers c
LEFT JOIN agg ON c.id = agg.customer_id;
SQL

bqq 'SELECT COUNT(*) AS customers, SUM(total) AS matched_sales FROM pde_d4_l3.customer_totals'
```

✅ **Check:** **customers = 2**, **matched_sales = 300**.

---

## 🖱️ Console Alternative (Steps 4–10)
Every SQL block above also runs in **BigQuery Studio**:
1. **BigQuery → Studio → + (new query tab)**.
2. Paste **only the SQL** (the lines between `bqq <<'SQL'` and `SQL`; or, for single-line `bqq '...'` commands, the text inside the quotes).
3. Click **Run** and compare with the ✅ checks.
4. For created tables: **Query results → Go to table → Preview**.

> 💡 If you get a location error in the Console: **More → Query settings → Location → us-central1**.

---

## ✅ Validation Checklist

| # | Check | Pass / Fail |
|---|---|---|
| 1 | Variables set; `bqq` helper defined | |
| 2 | BigQuery access works; dataset didn't exist | |
| 3 | `pde_d4_l3` created in **us-central1** | |
| 4 | `customers`: 2 rows | |
| 5 | `orders`: 3 rows (one NULL customer) | |
| 6 | LEFT JOIN: 3 rows; customer 1 twice | |
| 7 | Customer 1: 2 / 2; customer 2: 1 / 0 | |
| 8 | Pre-aggregated join: 300 and 0 | |
| 9 | Orphans: 1 / 50 / 350 | |
| 10 | `customer_totals`: 2 / 300 | |

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
| `Already exists` | Inspect the owner; don't overwrite unrelated resources (lab SQL uses `CREATE OR REPLACE` only for lab tables) |
| SQL syntax error in Cloud Shell | Paste SQL only between `bqq <<'SQL'` and `SQL`, or into the BigQuery editor |

---

## 🧹 Clean-up (after all checks and screenshots)
```bash
export PROJECT_ID=$(gcloud config get-value project 2>/dev/null)

# Check the dataset contains only this lab's tables
bq --project_id="$PROJECT_ID" ls "$PROJECT_ID:pde_d4_l3"

# Delete the lab dataset and its tables
bq --project_id="$PROJECT_ID" rm -r -f -d "$PROJECT_ID:pde_d4_l3"

# Confirm it's gone
bq --project_id="$PROJECT_ID" ls --datasets
```

🖱️ **Console alternative:** **Explorer → `pde_d4_l3` → ⋮ → Delete** → type `delete`.

> 🛡️ Never run `gcloud iam service-accounts delete/disable`, `gcloud projects delete`, or clean-up affecting other datasets.

---

## 🔁 Knowledge Check
1. Why did customer 1 appear twice after the join? → **One-to-many**: one customer, two orders
2. Why did order 103 disappear from the join? → Its key is **NULL**, and `NULL = value` is never true
3. `COUNT(*)` or `COUNT(order_id)`: which gives the real order count after a LEFT JOIN? → **`COUNT(order_id)`**
4. How do you avoid inflated totals? → **Aggregate the fact table to the dimension's grain before joining**
5. Why audit orphan rows? → To **reconcile** totals (300 + 50 = 350) and not lose data silently

---

## 📚 References
- [BigQuery: optimize query computation](https://cloud.google.com/bigquery/docs/best-practices-performance-compute)
- [Google Skills – Professional Data Engineer path](https://www.skills.google/paths/16)

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 4 Overview](README.md) | ➡️ [Lab 4.4](Lab-4.4-JSON-Arrays-Structs.md)
