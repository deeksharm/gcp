# Lab 2.1b (Extension): Dataform Customer Pipeline

**Day 2 · Official Module: The Extract, Load, and Transform (ELT) Data Pipeline Pattern**

| Item | Details |
|---|---|
| **Type** | Extension to Lab 2.1 (uses the same Dataform repository and workspace) |
| **Data** | Customer data from **Lab 1.3** (`customer.csv`) |
| **Level** | Introductory |
| **Duration** | ~25 minutes |
| **Services** | BigQuery, Dataform, Cloud Shell |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).
> ✅ **Prerequisite:** complete [Lab 2.1](Lab-2.1-Dataform-SQL-Workflow.md) (repository `quickstart-repository`, workspace `quickstart-workspace`, and `defaultLocation: us-central1` in `workflow_settings.yaml`).

---

## 💡 Why This Lab?
Lab 2.1 showed the Dataform mechanics with fruit data. Here you apply the same ideas to **real business data**: the **customer records from Lab 1.3**. This links the two days:

```
Day 1 (Lab 1.3): Load customer data         Day 2 (this lab): Transform it with Dataform
customer.csv ──► BigQuery raw table   ──►   clean view ──► summary table  (+ data quality checks)
```

You'll build a small **ELT pipeline**:

```
raw_data.customers (declaration) ──► customer-source (view) ──► customer-summary (table)
                                           └──► assertions (uniqueKey, nonNull)
```

---

## 🎯 Objectives
1. Load the Lab 1.3 customer data into a **raw BigQuery table**
2. **Declare** an existing table as a Dataform source
3. Build a **clean view** that removes PII columns and tidies the data
4. Build a **summary table** that depends on the view
5. Add **assertions** (data-quality checks) and run the pipeline

---

## Concepts

| Term | Meaning |
|---|---|
| **Declaration** (`type: "declaration"`) | Tells Dataform about a table that **already exists** (Dataform didn't create it), so other files can `ref()` it |
| **View** (`type: "view"`) | A saved query; no data stored |
| **Table** (`type: "table"`) | Query results stored as a table, rebuilt on every run |
| **`${ref("name")}`** | Creates a dependency, so Dataform runs objects in the right order |
| **Assertion** | A data-quality test. It **passes when it finds zero bad rows** |
| **PII** | Personally identifiable information (address, phone, postal code), which should be kept out of reporting tables |

---

## Task 1. Load the Lab 1.3 customer data into BigQuery
Open **Cloud Shell**:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1        # the region allowed in your lab

cat > customer.csv <<EOF
customer_id,first_name,last_name,company,address,city,state,country,postal_code,phone,email
1,Asha,Verma,Infotech Ltd,12 MG Road,Bengaluru,KA,India,560001,+91-9800000001,asha.verma@example.com
2,Ravi,Kumar,DataWorks,45 Park Street,Kolkata,WB,India,700016,+91-9800000002,ravi.kumar@example.com
3,Meena,Iyer,,8 Marine Drive,Mumbai,MH,India,400020,+91-9800000003,meena.iyer@example.com
4,John,Smith,CloudCo,221 King St,Toronto,ON,Canada,M5H 1K5,+1-4165550104,john.smith@example.com
5,Sara,Lee,,77 Orchard Rd,Singapore,,Singapore,238841,+65-65550105,sara.lee@example.com
6,Karan,Mehta,Analytics Hub,3 Civil Lines,Pune,MH,India,411001,+91-9800000006,karan.mehta@example.com
EOF
```

> All names, phone numbers and emails are made up. It's the same file as Lab 1.3.

Create a dataset for raw data and load the file:
```bash
bq --location=${REGION} mk -d raw_data

bq load --source_format=CSV --skip_leading_rows=1 \
  raw_data.customers ./customer.csv \
  customer_id:INTEGER,first_name:STRING,last_name:STRING,company:STRING,address:STRING,city:STRING,state:STRING,country:STRING,postal_code:STRING,phone:STRING,email:STRING

bq query --use_legacy_sql=false 'SELECT * FROM raw_data.customers ORDER BY customer_id'
```

✅ **Check:** the query returns **6 customers**.

> 💡 The dataset must be in the **same location** as Dataform (`us-central1`), or Dataform can't read it.

---

## Task 2. Declare the raw table in Dataform
1. Go to **BigQuery → Dataform → quickstart-repository → quickstart-workspace**.
2. In the **Files** pane, hover over `definitions` → **⋮** → **Create file**.
3. File path: `definitions/customers-raw.sqlx` → **Create file**.
4. Add:

```sql
config {
  type: "declaration",
  schema: "raw_data",
  name: "customers",
  description: "Raw customer data loaded from customer.csv (same data as Lab 1.3)"
}
```

> 💡 A declaration doesn't create anything. It registers `raw_data.customers` as a **source**, so you can use `${ref("customers")}`.

---

## Task 3. Create the clean source view (no PII)
1. **Create file:** `definitions/customer-source.sqlx`
2. Add:

```sql
config {
  type: "view",
  description: "Clean customer data without PII columns",
  assertions: {
    uniqueKey: ["customer_id"],
    nonNull: ["customer_id", "email"]
  }
}

SELECT
  customer_id,
  INITCAP(first_name)  AS first_name,
  INITCAP(last_name)   AS last_name,
  NULLIF(company, '')  AS company,
  city,
  NULLIF(state, '')    AS state,
  country,
  LOWER(email)         AS email
FROM ${ref("customers")}
```

| Part | What it does |
|---|---|
| Columns selected | **Leaves out** `address`, `postal_code` and `phone` (PII) |
| `INITCAP()` | Capitalises names |
| `NULLIF(x, '')` | Turns empty text into `NULL` |
| `LOWER(email)` | Standardises emails |
| `uniqueKey` assertion | Fails if any `customer_id` appears more than once |
| `nonNull` assertion | Fails if `customer_id` or `email` is empty |

---

## Task 4. Create the summary table
1. **Create file:** `definitions/customer-summary.sqlx`
2. Add:

```sql
config {
  type: "table",
  description: "Customers per country"
}

SELECT
  country,
  COUNT(*)                   AS total_customers,
  COUNT(company)             AS customers_with_company,
  COUNT(*) - COUNT(company)  AS individual_customers
FROM ${ref("customer-source")}
GROUP BY country
ORDER BY total_customers DESC
```

3. Open the **Compiled graph** tab.

✅ **Check:** the graph shows `customers` → `customer-source` → `customer-summary`, plus the assertion checks on `customer-source`.

---

## Task 5. Run the pipeline
1. Click **Start execution → Execute actions**.
2. Authentication: **Execute with user credentials**.
3. **Select actions to execute → Select all → OK**. (This also re-runs the fruit objects from Lab 2.1, which is fine.)
4. Click **Start execution**.

✅ **Check:** the execution status is **Succeeded**, and the assertion actions show as passed.

---

## Task 6. Check the results in BigQuery
```sql
SELECT * FROM `dataform.customer-source` ORDER BY customer_id;
SELECT * FROM `dataform.customer-summary`;
```

✅ **Expected `customer-summary`:**

| country | total_customers | customers_with_company | individual_customers |
|---|---|---|---|
| India | 4 | 3 | 1 |
| Canada | 1 | 1 | 0 |
| Singapore | 1 | 0 | 1 |

✅ `customer-source` has 6 rows, **no** address, phone or postal code columns, and lowercase emails.

> 💡 Object names contain hyphens, so wrap them in **backticks**.

---

## Task 7 (Optional). See an assertion fail
1. In BigQuery, add a duplicate customer to the raw table:
```sql
INSERT INTO raw_data.customers (customer_id, first_name, last_name, city, country, email)
VALUES (6, 'Duplicate', 'Row', 'Pune', 'India', 'dup@example.com');
```
2. Run the Dataform execution again.

✅ **Expected:** the **`uniqueKey` assertion fails** (customer_id 6 appears twice). This is how Dataform catches bad data before it reaches reports.

3. Remove the bad row and re-run:
```sql
DELETE FROM raw_data.customers WHERE email = 'dup@example.com';
```

> ⚠️ An `INSERT` can fail with a "streaming buffer" error only if data was streamed in; this table was batch-loaded, so it works.

---

## 🔁 What You Built

| Layer | Object | Type | Purpose |
|---|---|---|---|
| Raw | `raw_data.customers` | Declaration | Data as loaded (Lab 1.3 file) |
| Clean | `dataform.customer-source` | View | Tidy data, PII removed, quality-checked |
| Reporting | `dataform.customer-summary` | Table | Ready for dashboards |

This **raw → clean → reporting** layering is the standard pattern for ELT pipelines in BigQuery.

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| `US violates constraint constraints/gcp.resourceLocations` | Set `defaultLocation: us-central1` in `workflow_settings.yaml` |
| `Not found: Table raw_data.customers` / *not found in location* | Task 1 not done, or `raw_data` isn't in `us-central1` (`bq show raw_data`) |
| `ref("customers")` error in compilation | Check the declaration file: `schema: "raw_data"`, `name: "customers"` |
| Assertion fails unexpectedly | Query the assertion's SQL from the execution log to see the bad rows |
| `Access Denied` | Check you chose **Execute with user credentials** |

---

## 🔁 Knowledge Check
1. How do you use a table in Dataform that Dataform didn't create? → A **declaration**
2. When does an assertion pass? → When its query returns **zero rows**
3. Why did we exclude `address`, `phone` and `postal_code`? → They're **PII**; keep them out of reporting tables
4. Which object runs first, and why? → `customer-source`, because `customer-summary` **refs** it

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab 2.1](Lab-2.1-Dataform-SQL-Workflow.md) | ➡️ [Lab 2.2](Lab-2.2-Serverless-Spark-to-BigQuery.md)
