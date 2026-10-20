# Day 2: ELT, ETL & Automation Patterns

**Google Cloud Professional Data Engineer – 8-Day Training Program**

> 🔐 New to the lab? Follow the [Lab Access Guide](../README.md) to sign in to your GCP lab account first.

---

## 🎯 What You Will Learn Today

**Official course:** Introduction to Data Engineering on Google Cloud (Modules 5–8)
*(Module 4, the Extract and Load pattern, was covered on Day 1 with the Lakehouse lab.)*

### Module 5: The Extract, Load, and Transform (ELT) Data Pipeline Pattern
- **ELT architecture:** load raw data into BigQuery first, then transform it with SQL
- **SQL scripting and scheduling** with BigQuery (multi-statement queries, scheduled queries)
- **Dataform:** build, test and orchestrate SQL workflows (SQLX, dependencies, assertions)

### Module 6: The Extract, Transform, and Load (ETL) Data Pipeline Pattern
- **ETL architecture:** transform data *before* loading it into the warehouse
- **Google Cloud GUI tools for ETL:** Cloud Data Fusion, Dataprep, Dataflow Job Builder
- **Batch processing with Managed Service for Apache Spark** (Dataproc and Serverless for Apache Spark)
- **Streaming processing options:** Pub/Sub (messaging) + Dataflow (processing) + BigQuery (analytics)
- **Bigtable** in data pipelines (low-latency serving)

### Module 7: Automation Techniques
- **Automation patterns:** scheduled, event-driven and orchestrated pipelines
- **Cloud Scheduler** and **Workflows**
- **Managed Service for Apache Airflow** (Cloud Composer)
- **Cloud Run functions** and **Eventarc** for event-driven pipelines

### Module 8: Course Summary
- How to choose between **EL, ELT and ETL** for a given scenario

---

## 🗓️ Agenda
1. Recap of Day 1 (EL pattern, Lakehouse)
2. **Module A:** ELT Pattern + ETL Pattern (batch)
3. **Lab 2.1:** Create and execute a SQL workflow in Dataform
4. **Lab 2.2:** Use Serverless for Apache Spark to Load BigQuery
5. Break
6. **Module B:** ETL Pattern (streaming) + Automation Techniques + Course Summary
7. **Lab 2.3:** Creating a Streaming Data Pipeline for a Real-Time Dashboard with Dataflow
8. **Lab 2.4:** Use Cloud Run Functions to Load BigQuery
9. Scenario drill + quiz

---

## 🧪 Today's Labs (Official)

| # | Lab | Official module | What you'll do |
|---|---|---|---|
| 2.1 | [Create and execute a SQL workflow in Dataform](Lab-2.1-Dataform-SQL-Workflow.md) | ELT Pattern | Build a Dataform repository, define SQLX tables and views, run the workflow in BigQuery |
| 2.2 | [Use Serverless for Apache Spark to Load BigQuery](Lab-2.2-Serverless-Spark-to-BigQuery.md) | ETL Pattern | Run a Spark batch job without managing clusters to transform files and load them into BigQuery |
| 2.3 | [Creating a Streaming Data Pipeline for a Real-Time Dashboard with Dataflow](Lab-2.3-Streaming-Pipeline-Real-Time-Dashboard.md) | ETL Pattern (streaming) | Stream events from Pub/Sub through a Dataflow template into BigQuery and visualise them in Looker Studio |
| 2.4 | [Use Cloud Run Functions to Load BigQuery](Lab-2.4-Cloud-Run-Functions-to-BigQuery.md) | Automation Techniques | Trigger a function automatically when a file lands in Cloud Storage and load it into BigQuery |

---

## 🔁 Pipeline Patterns at a Glance

| Pattern | Flow | Typical Google Cloud tools | Lab |
|---|---|---|---|
| **EL** | Extract → Load (query in place) | `bq load`, BigQuery Data Transfer Service, Lakehouse | Day 1 (1.2, 1.3) |
| **ELT** | Extract → Load → Transform in the warehouse | BigQuery SQL, scheduled queries, Dataform | 2.1 |
| **ETL (batch)** | Extract → Transform → Load | Serverless for Apache Spark, Dataproc, Data Fusion, Dataflow | 2.2 |
| **ETL (streaming)** | Events → Transform in real time → Load | Pub/Sub, Dataflow, BigQuery, Bigtable | 2.3 |
| **Automation** | Event or schedule triggers the pipeline | Cloud Run functions, Eventarc, Cloud Scheduler, Workflows, Composer | 2.4 |

---

## ✅ By the End of Today You Can
- [ ] Explain the difference between **EL, ELT and ETL** and when to use each
- [ ] Build and run a **Dataform** SQL workflow
- [ ] Run a **serverless Spark** batch job that loads BigQuery
- [ ] Describe a **streaming pipeline** with Pub/Sub, Dataflow and BigQuery
- [ ] Automate data loading with **event-driven Cloud Run functions**
- [ ] Choose the right **automation tool** (Scheduler, Workflows, Composer, Cloud Run functions)

---

## 📚 Helpful Reference Guide

### ELT & Dataform
- [Dataform overview](https://cloud.google.com/dataform/docs/overview)
- [BigQuery multi-statement queries (scripting)](https://cloud.google.com/bigquery/docs/multi-statement-queries)
- [Scheduling queries in BigQuery](https://cloud.google.com/bigquery/docs/scheduling-queries)

### ETL: Batch
- [Serverless for Apache Spark overview](https://cloud.google.com/dataproc-serverless/docs/overview)
- [Dataproc overview](https://cloud.google.com/dataproc/docs/concepts/overview)
- [Cloud Data Fusion](https://cloud.google.com/data-fusion/docs)

### ETL: Streaming
- [Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [Dataflow overview](https://cloud.google.com/dataflow/docs/overview)
- [Dataflow templates](https://cloud.google.com/dataflow/docs/concepts/dataflow-templates)
- [Bigtable overview](https://cloud.google.com/bigtable/docs/overview)
- [Looker Studio](https://cloud.google.com/looker-studio)

### Automation
- [Cloud Run functions](https://cloud.google.com/functions/docs)
- [Eventarc](https://cloud.google.com/eventarc/docs)
- [Cloud Scheduler](https://cloud.google.com/scheduler/docs)
- [Workflows](https://cloud.google.com/workflows/docs)
- [Cloud Composer (Managed Airflow)](https://cloud.google.com/composer/docs)

### Official Training & Certification
- [Google Skills – Professional Data Engineer path](https://www.skills.google/paths/16)
- [Course: Introduction to Data Engineering on Google Cloud](https://www.skills.google/paths/16/course_templates/1157)
- [Professional Data Engineer certification](https://cloud.google.com/learn/certification/data-engineer)

---

## 💡 Tips for Today
- Reuse your Day 1 resources: the `-raw` bucket and BigQuery datasets.
- **Stop the Dataflow streaming job** after Lab 2.3. Streaming jobs keep running and use lab budget.
- Serverless Spark and Dataflow jobs take **2–5 minutes to start**; read the lab's concepts section while you wait.
- If a service API is disabled, enable it from **APIs & Services → Library**, or ask the trainer.

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Day 1](../Day-01/README.md) | ➡️ [Day 3](../Day-03/README.md)
