
# Day 1: Foundations & Introduction to Data Engineering

**Google Cloud Professional Data Engineer – 8-Day Training Program**

> 🔐 New to the lab? Follow the [Lab Access Guide](../README.md) to sign in to your GCP lab account first.

---

## 🎯 What You Will Learn Today

### Part A: GCP Core Foundations
- How Google Cloud is organised: **regions, zones and multi-regions**
- The **resource hierarchy**: Organization → Folders → Projects → Resources
- **IAM**: who can do what on which resource (basic, predefined and custom roles)
- **Service accounts** and the principle of **least privilege**
- **Cloud Storage** classes (Standard, Nearline, Coldline, Archive), lifecycle rules and versioning
- Using the **Cloud Console** and **Cloud Shell** (`gcloud`, `bq`, `gcloud storage`)

### Part B: Introduction to Data Engineering on Google Cloud
- The **role of a data engineer** and the data lifecycle: Ingest → Store → Process → Analyse → Serve
- Data sources, sinks, formats (CSV, JSON, Avro, Parquet, Iceberg) and storage options
- Data lake vs data warehouse vs lakehouse
- **Data replication & migration** tools: Storage Transfer Service, Transfer Appliance, BigQuery Data Transfer Service, Database Migration Service and **Datastream**

---

## 🧪 Today's Labs

| # | Lab | What you'll do |
|---|---|---|
| 1.1 | Tour of Google Cloud Console & Cloud Shell | Explore the Console, check your project and IAM, run your first `gcloud` commands |
| 1.2 | IAM Custom Roles & Service Accounts | Create, update and delete custom roles; assign one to a service account |
| 1.3 | Loading Data into BigQuery | Load data from Cloud Storage, local files and public datasets; create an external table |
| 1.4 | Datastream: PostgreSQL Replication to BigQuery | Replicate a Cloud SQL PostgreSQL database into BigQuery in near real time |

---

## ✅ By the End of Today You Can
- [ ] Navigate the Google Cloud Console and Cloud Shell confidently
- [ ] Explain the resource hierarchy and how IAM inheritance works
- [ ] Choose the right IAM role type and Cloud Storage class for a scenario
- [ ] Load and query data in BigQuery
- [ ] Pick the right replication or migration tool for a given requirement

---

## 📚 Helpful Reference Guide

### GCP Foundations
- [Google Cloud overview](https://cloud.google.com/docs/overview)
- [Resource hierarchy](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy)
- [Regions and zones](https://cloud.google.com/about/locations)
- [Cloud Shell](https://cloud.google.com/shell/docs)
- [gcloud CLI reference](https://cloud.google.com/sdk/gcloud/reference)

### IAM & Service Accounts
- [IAM overview](https://cloud.google.com/iam/docs/overview)
- [Understanding roles](https://cloud.google.com/iam/docs/understanding-roles)
- [Creating custom roles](https://cloud.google.com/iam/docs/creating-custom-roles)
- [Service accounts overview](https://cloud.google.com/iam/docs/service-account-overview)

### Cloud Storage
- [Storage classes](https://cloud.google.com/storage/docs/storage-classes)
- [Object lifecycle management](https://cloud.google.com/storage/docs/lifecycle)

### BigQuery
- [What is BigQuery?](https://cloud.google.com/bigquery/docs/introduction)
- [Loading data into BigQuery](https://cloud.google.com/bigquery/docs/loading-data)
- [External tables](https://cloud.google.com/bigquery/docs/external-tables)
- [BigQuery public datasets](https://cloud.google.com/bigquery/public-data)

### Replication & Migration
- [Datastream overview](https://cloud.google.com/datastream/docs/overview)
- [Database Migration Service](https://cloud.google.com/database-migration/docs)
- [Storage Transfer Service](https://cloud.google.com/storage-transfer/docs/overview)
- [Transfer Appliance](https://cloud.google.com/transfer-appliance/docs)
- [BigQuery Data Transfer Service](https://cloud.google.com/bigquery/docs/dts-introduction)

### Official Training & Certification
- [Professional Data Engineer certification](https://cloud.google.com/learn/certification/data-engineer)
- [Google Skills – Professional Data Engineer path](https://www.skills.google/paths/16)
- [Course: Introduction to Data Engineering on Google Cloud](https://www.skills.google/paths/16/course_templates/1157)

---

## 💡 Tips for Today
- Use an **Incognito window** for the lab so it doesn't clash with your personal Google account.
- Note your **Project ID**; you'll use it in every lab.
- **Delete the Cloud SQL instance** after Lab 1.4 to save lab budget.

---

**Trainer:** Deeksha Sharma · [LinkedIn](https://www.linkedin.com/in/deekshasharma191196/) · [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)

⬅️ [Lab Access Guide](../README.md) | ➡️ [Day 2](../Day-02/README.md)
