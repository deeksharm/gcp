# Instructor checklist — Day 3

## Before session
- [ ] Confirm Nuvepro portal credentials work in Incognito; assign **one authorized project per learner**.
- [ ] Confirm project billing and remaining budget; set `us-central1` / `US` where appropriate.
- [ ] Confirm required APIs and admin permissions for this day's paid managed services; pre-provision limited services if training time is short.
- [ ] Run each SQL example in a **test project**; confirm expected rows, schemas, and any regional / model availability differences.
- [ ] Test service-specific instructions separately; no SQL fallback should be presented as proof of successful managed-service execution.
- [ ] Have a backup plan for denied IAM, quotas and organization policies.

## During session
- [ ] Explain architecture and data flow before commands.
- [ ] Learners run SQL, inspect outputs, then complete authorized Console and CLI extension steps.
- [ ] Capture service-specific screenshots and errors for evidence.
- [ ] Discuss production costs, least privilege, observability and reliability.

## After session
- [ ] Cancel long-running Dataflow jobs and Composer/Data Fusion/Bigtable/Cloud SQL resources created during demonstrations.
- [ ] Remove dedicated Pub/Sub/Cloud Storage / BigQuery training resources only where scoped to this exercise.
- [ ] Confirm no service-account keys, secrets, or credentials were committed to GitHub.
- [ ] Mark advanced features that were not run in the client account as **not executed**.
