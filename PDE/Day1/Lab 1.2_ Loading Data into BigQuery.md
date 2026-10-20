# Lab 1.2: Loading Data into BigQuery

**Day 1 · Official Module: Data Engineering Tasks and Components**

| Item | Details |
|---|---|
| **Official lab** | *Loading data into BigQuery*, in the course [Introduction to Data Engineering on Google Cloud](https://www.skills.google/paths/16/course_templates/1157) |
| **Related catalog lab** | [Loading Your Own Data into BigQuery (GSP865)](https://www.skills.google/focuses/17816) |
| **Level** | Introductory |
| **Duration** | ~30 minutes |
| **Services** | BigQuery, Cloud Storage, Cloud Shell |
| **Dataset** | NYC Yellow Taxi trips, 2018 (subset) |

---

## 🎯 Objectives
1. Create a BigQuery **dataset**
2. Load a **local CSV file** into a table using the Console
3. Append data from **Cloud Storage** using the `bq` command line
4. Create new tables from existing tables using **DDL (CREATE TABLE AS SELECT)**
5. Run queries to analyse the loaded data

---

## Concepts
| Term | Meaning |
|---|---|
| **Dataset** | A container for tables and views, created in a location (for example `US`) |
| **Load job** | Batch-loads files (CSV, JSON, Avro, Parquet, ORC) into a table. Free; you only pay for storage |
| **Schema auto-detect** | BigQuery infers column names and types from the data |
| **Write disposition** | `WRITE_APPEND` (`--noreplace`), `WRITE_TRUNCATE` (`--replace`), `WRITE_EMPTY` |
| **DDL** | SQL statements such as `CREATE TABLE ... AS SELECT` |

---

## Task 1. Create a dataset
1. Go to **Navigation menu → BigQuery**.
2. In the **Explorer** panel, click **⋮ (View actions)** next to your project ID → **Create dataset**.
3. Dataset ID: `nyctaxi`
4. Location type: **Multi-region → US**
5. Click **Create dataset**.

**Command-line alternative:**
```bash
bq --location=US mk -d nyctaxi
```

✅ **Check:** `nyctaxi` appears under your project.

---

## Task 2. Load a local CSV file (Console)
1. Download the sample file to your computer:
   **[nyc_tlc_yellow_trips_2018_subset_1.csv](https://storage.googleapis.com/spls/gsp865/OCBL013/nyc_tlc_yellow_trips_2018_subset_1.csv)**
2. In BigQuery, click **⋮** next to `nyctaxi` → **Create table**.
3. Fill in the form:

| Setting | Value |
|---|---|
| Create table from | **Upload** |
| Select file | The CSV you downloaded |
| File format | **CSV** |
| Table | `2018trips` |
| Schema | ✅ **Auto detect** |

4. Click **Create table**.
5. Open the `2018trips` table → **Preview** and check that the columns loaded.

**Query the data:** find the 5 most expensive trips.
```sql
SELECT *
FROM `nyctaxi.2018trips`
ORDER BY fare_amount DESC
LIMIT 5;
```

> 💡 Table names that start with a number must be enclosed in backticks.

✅ **Check:** the query returns 5 rows.

---

## Task 3. Append data from Cloud Storage (`bq` command line)
Open **Cloud Shell** and load a second subset of the same data **into the same table**:

```bash
bq load \
  --source_format=CSV \
  --autodetect \
  --noreplace \
  nyctaxi.2018trips \
  gs://spls/gsp865/OCBL013/nyc_tlc_yellow_trips_2018_subset_2.csv
```

- `--noreplace` **appends** rows to the existing table.
- `--autodetect` infers the schema.

Check the row count:
```bash
bq show --format=prettyjson nyctaxi.2018trips | grep numRows
```

✅ **Check:** the row count has roughly **doubled**. Re-run the Task 2 query to see if the top 5 changed.

---

## Task 4. Create tables from other tables (DDL)
Create a table that contains only **January** trips:

```sql
CREATE OR REPLACE TABLE nyctaxi.january_trips AS
SELECT *
FROM `nyctaxi.2018trips`
WHERE EXTRACT(MONTH FROM pickup_datetime) = 1;
```

Find the **longest trip** in January:
```sql
SELECT *
FROM nyctaxi.january_trips
ORDER BY trip_distance DESC
LIMIT 1;
```

✅ **Check:** the `january_trips` table exists and the query returns 1 row.

---

## Task 5 (Extension). Load from your own bucket (created in Lab 1.1)
Practise the full **Cloud Storage → BigQuery** path, using the bucket and file you uploaded in Lab 1.1:

```bash
export PROJECT_ID=$(gcloud config get-value project)

# Check the file from Lab 1.1 is there
gcloud storage ls gs://${PROJECT_ID}-raw/nyctaxi/

# Load it into a new table
bq load --source_format=CSV --autodetect \
  nyctaxi.trips_from_my_bucket \
  gs://${PROJECT_ID}-raw/nyctaxi/nyc_tlc_yellow_trips_2018_subset_1.csv
```

Optional: query the file **without loading it**, using an external table:
```sql
CREATE OR REPLACE EXTERNAL TABLE nyctaxi.trips_external
OPTIONS (
  format = 'CSV',
  uris = ['gs://<PROJECT_ID>-raw/nyctaxi/*.csv'],
  skip_leading_rows = 1
);

SELECT COUNT(*) AS trips FROM nyctaxi.trips_external;
```
> Replace `<PROJECT_ID>` with your project ID.

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| `Access Denied` on `gs://spls/...` | The sample bucket may not be readable from your lab account. Use the **fallback** below |
| `Not found: Dataset` | Check that the dataset is named `nyctaxi` and is in location `US` |
| Syntax error on `2018trips` | Enclose the table name in backticks: `` `nyctaxi.2018trips` `` |

**Fallback: build the same data from the BigQuery public dataset**
```sql
CREATE OR REPLACE TABLE nyctaxi.2018trips AS
SELECT *
FROM `bigquery-public-data.new_york_taxi_trips.tlc_yellow_trips_2018`
WHERE RAND() < 0.0001;
```
Then export it to your bucket as CSV, to practise loading files:
```bash
bq extract --destination_format=CSV nyctaxi.2018trips gs://${PROJECT_ID}-raw/nyctaxi/trips_*.csv
```

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

➡️ [Lab 1.4](Lab-1.4-Datastream-PostgreSQL-to-BigQuery.md)
