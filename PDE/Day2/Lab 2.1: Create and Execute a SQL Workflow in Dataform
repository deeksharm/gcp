# Lab 2.1: Create and Execute a SQL Workflow in Dataform

**Day 2 · Official Module: The Extract, Load, and Transform (ELT) Data Pipeline Pattern**

| Item | Details |
|---|---|
| **Official lab** | [Create and execute a SQL workflow in Dataform](https://www.skills.google/focuses/77087) |
| **Level** | Introductory |
| **Duration** | ~30 minutes |
| **Services** | BigQuery, Dataform |

> 🔐 Sign in to your lab account first using the [Lab Access Guide](../README.md).

---

## 🎯 Objectives
1. Create a **Dataform repository**
2. Create and initialise a **development workspace**
3. Define a **view** and a **table** using SQLX files
4. Understand the permissions Dataform needs in **BigQuery**
5. **Execute** the SQL workflow and review the **execution logs**

---

## 💡 Why This Lab?
On Day 1 you **loaded** data into BigQuery. Raw data is rarely ready to use; it needs cleaning, joining and summarising. That's the **T (Transform)** in **ELT**, and in BigQuery it's done with SQL.

```
Extract ──► Load raw data into BigQuery ──► Transform with SQL inside BigQuery
                     (Day 1)                       (Dataform: today)
```

**Dataform** is the tool inside BigQuery for managing these SQL transformations. Think of it as a recipe book: every SQL step is in one place, each one declares which tables it needs, and Dataform runs them **in the right order automatically**.

In a real company, one dashboard can depend on dozens of SQL steps:
```
raw_orders ────► clean_orders ────┐
raw_customers ─► clean_customers ─┼─► daily_sales ─► sales_dashboard
raw_products ──► clean_products ──┘
```
This lab is a small "hello world" with fruit data that shows exactly that mechanism: a **view** feeds a **table**, and Dataform works out the run order from `${ref()}`.

---

## Concepts

| Term | Meaning |
|---|---|
| **ELT** | Load raw data into BigQuery first, then **transform it with SQL** inside the warehouse |
| **Dataform** | A BigQuery service to develop, test, version and run SQL transformation workflows |
| **Repository** | Stores your Dataform code (it can be linked to Git) |
| **Development workspace** | Your personal editable copy of the repository, like a Git branch |
| **SQLX** | SQL plus a `config { }` block that tells Dataform what to create (`table`, `view`, `incremental`) |
| **`${ref("name")}`** | References another Dataform object. Dataform builds a **dependency graph** and runs objects in the right order |
| **Dataform service account** | The identity Dataform uses to run queries in BigQuery |

```
quickstart-source.sqlx (view)  ──ref()──►  quickstart-table.sqlx (table)
                 │                                   │
                 └──────────► BigQuery dataset "dataform" ◄──┘
```

---

## Task 1. Create a Dataform repository
Open **Cloud Shell** and enable the API:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1        # the region allowed in your lab
gcloud services enable dataform.googleapis.com
```

In the Console:
1. Go to **Navigation menu → BigQuery → Dataform**.
2. Click **Create repository**.
3. Repository ID: `quickstart-repository`
4. Region: **`us-central1`**
5. Click **Create**.
6. Note the **Dataform service account** shown after creation (format: `service-<PROJECT_NUMBER>@gcp-sa-dataform.iam.gserviceaccount.com`).
7. Ignore the warning about missing roles (see Task 5) and click **Go to repositories**.

> ⚠️ If you get a *permission denied* error about the API, wait 1–2 minutes and create the repository again.

✅ **Check:** `quickstart-repository` appears in the Dataform repository list.

---

## Task 2. Create and initialise a development workspace
1. Click **quickstart-repository**.
2. Click **Create development workspace**.
3. Workspace ID: `quickstart-workspace` → **Create**.
4. Open **quickstart-workspace** and click **Initialize workspace**.
5. **Set the BigQuery location.** Open **`workflow_settings.yaml`** and change `defaultLocation` to your lab region (add the line if it's missing):

```yaml
defaultProject: <your-project-id>          # already filled in
defaultLocation: us-central1               # change from US
defaultDataset: dataform
defaultAssertionDataset: dataform_assertions
```

> ⚠️ Only change the **`defaultLocation`** line; keep the other lines as they are. Your lab blocks the `US` location, so leaving it as `US` makes Task 6 fail with *"US violates constraint constraints/gcp.resourceLocations"*.

✅ **Check:** the Files pane shows a `definitions/` folder, and `workflow_settings.yaml` shows `defaultLocation: us-central1`.

> 💡 `workflow_settings.yaml` holds the default project, BigQuery **dataset** (`dataform`) and **location** for everything Dataform builds.

---

## Task 3. Create a SQLX file that defines a view
1. In the **Files** pane, hover over `definitions` → **⋮** → **Create file**.
2. File path: `definitions/quickstart-source.sqlx` → **Create file**.
3. Open the file and add:

```sql
config {
  type: "view"
}

SELECT "apples"  AS fruit, 2 AS count
UNION ALL
SELECT "oranges" AS fruit, 5 AS count
UNION ALL
SELECT "pears"   AS fruit, 1 AS count
UNION ALL
SELECT "bananas" AS fruit, 0 AS count
```

> Dataform saves your changes automatically.

---

## Task 4. Create a SQLX file that defines a table
1. In the **Files** pane, hover over `definitions` → **⋮** → **Create file**.
2. File path: `definitions/quickstart-table.sqlx` → **Create file**.
3. Open the file and add:

```sql
config {
  type: "table"
}

SELECT
  fruit,
  SUM(count) AS count
FROM ${ref("quickstart-source")}
GROUP BY 1
```

4. Click the **Compiled graph** tab (or the **Compiled queries** panel) and check that `quickstart-table` **depends on** `quickstart-source`.

> 💡 You may see a warning in the compiled queries panel at this point. Ignore it and continue; it clears once the workflow has run.

---

## Task 5. Review Dataform permissions (no action needed)
In the official lab, you grant the Dataform service account three BigQuery roles here. **In our training environment the lab provider manages IAM, so participants can't grant roles; skip the grant step.**

Why it still works: in Task 6 you run the workflow with **your own user credentials**, so the queries run with your lab account's BigQuery permissions.

| Role (official lab) | Why Dataform needs it |
|---|---|
| **BigQuery Job User** | Run query jobs |
| **BigQuery Data Editor** | Create and replace tables and views |
| **BigQuery Data Viewer** | Read source tables |

> 📝 **Real world:** for **scheduled** executions (release and workflow configurations), Dataform runs as a **service account**, and that service account must have these roles. An admin grants them once.
>
> 👩‍🏫 **Trainer note:** if executions fail with *Access Denied*, ask the lab provider to grant these three roles to `service-<PROJECT_NUMBER>@gcp-sa-dataform.iam.gserviceaccount.com`.

---

## Task 6. Execute the workflow
1. Go to **BigQuery → Dataform → quickstart-repository → quickstart-workspace**.
2. Click **Start execution** → **Execute actions**.
3. Authentication: **Execute with user credentials** (your lab login ID).
4. **Select actions to execute** → **Select all** → **OK**.
5. Leave the other options at their defaults and click **Start execution**.
6. If prompted, sign in with your lab account and click **Allow**.

Dataform creates the objects in a BigQuery dataset called **`dataform`**, in `us-central1`.

**Check the results in BigQuery:**
```sql
SELECT * FROM `dataform.quickstart-source`;
SELECT * FROM `dataform.quickstart-table` ORDER BY count DESC;
```

> 💡 The object names come from the file names and contain hyphens, so wrap them in **backticks**. If a query fails, check the exact names in the **Explorer** panel.

✅ **Check:** the `dataform` dataset contains the **view** `quickstart-source` and the **table** `quickstart-table` with 4 fruits (oranges 5, apples 2, pears 1, bananas 0).

---

## Task 7. View the execution logs
1. In **quickstart-workspace**, click **Executions** (or **Workflow execution logs**).
2. Open the latest execution.
3. Review each action: status, start and end time, and the **SQL that Dataform ran** in BigQuery.

> 📝 **Discuss:** Dataform ran the **view first**, then the **table**, because of `${ref()}`. That dependency management is what makes Dataform useful for large ELT pipelines.

---

## 🔁 Why Dataform for ELT?

| Without Dataform | With Dataform |
|---|---|
| Many separate SQL scripts and scheduled queries | One repository with all transformations |
| You manage the run order by hand | Dependencies resolved automatically with `ref()` |
| No built-in tests | **Assertions** to test data quality |
| Hard to version | Git integration, workspaces, code review |

---

## 🛠️ Troubleshooting
| Issue | Fix |
|---|---|
| Can't create the repository | Run `gcloud services enable dataform.googleapis.com`, wait 1–2 minutes, then retry |
| Execution fails with *US violates constraint constraints/gcp.resourceLocations* | Set `defaultLocation: us-central1` in `workflow_settings.yaml` (Task 2, step 5) and re-run. If a `dataform` dataset was half-created elsewhere, run `bq rm -r -f -d dataform` first |
| Execution fails with *Access Denied* | Check you chose **Execute with user credentials**; otherwise ask the trainer (the lab provider may need to grant roles) |
| `Not found: Dataset dataform` | The execution didn't finish; check **Executions** for errors |
| Query on `quickstart-source` fails | Wrap the name in backticks and check the exact name in BigQuery **Explorer** |

---

## 🔁 Knowledge Check
1. In ELT, where does the transformation happen? → **Inside the data warehouse (BigQuery), using SQL**
2. What Dataform function creates a dependency between two objects? → **`${ref("object_name")}`**
3. Which `config` types can a SQLX file create? → `table`, `view`, `incremental` (plus `assertion` and `operations`)
4. Which roles does a Dataform service account need for scheduled runs? → **BigQuery Job User, Data Editor, Data Viewer**

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 2 Overview](README.md) | ➡️ [Lab 2.2](Lab-2.2-Serverless-Spark-to-BigQuery.md)
