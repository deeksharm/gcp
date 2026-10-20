# GCP Data Engineer Lab – Access Guide

How to log in to your **GCP Data Engineer Account Lab** for the Professional Data Engineer training.

---

## Before You Start

- Your trainer will give you **portal login credentials** (username and password) for the lab portal.
- Use **Google Chrome** in an **Incognito / Private window**. This stops your personal Google account from clashing with the lab account.
- Lab portal URL: **https://cloudlabs.nuvepro.com/company/AlchemySolutions/home**

---

## Step-by-Step Login

### Step 1: Log in to the lab portal
1. Open **https://cloudlabs.nuvepro.com/company/AlchemySolutions/home**
2. Sign in with the **portal credentials** your trainer gave you.

### Step 2: Open your lab
1. On the home page, find the card **"GCP Data Engineer Account Lab"**.
2. Check the **Expires On** date shown on the card.
3. Click **View Lab**.

### Step 3: Start the lab
1. In the **Lab Control Panel**, click the green **Start** button.
2. Wait for **Latest Status** to change to **"Start – Complete"**.
   - Click the **refresh icon** next to the status to update it.
   - This can take a few minutes.

### Step 4: Note your access details
Once the lab has started, the **Access Details** section on the left shows:

| Field | What it is |
|---|---|
| **loginId** | Your Google Cloud login email (ends with `@alchemysolutionsgcp.nuvelabs.com`) |
| **loginpassword** | Your Google Cloud password. Click the **eye icon** to view it or the **copy icon** to copy it |
| **serviceAccountKeyJson** | Service account key. **Only use it when a lab specifically asks for it** |

### Step 5: Sign in to Google Cloud
1. On the right side of the panel, click **signinLink**.
2. A Google sign-in page opens.
3. Enter the **loginId** as the email and click **Next**.
4. Enter the **loginpassword** and click **Next**.
5. If prompted, accept the Google Cloud terms of service.
6. You're now in the **Google Cloud Console**. Check that the lab project is selected in the project drop-down at the top.

---

## Useful Panel Sections

Under **More Details** on the right side of the Lab Control Panel:

- **Policies**: what you can and can't do in the lab account
- **Instructions**: lab-specific notes from the provider
- **Other Details**: extra account information
- **Budget Quota**: your remaining lab budget. **Check it regularly**

---

## Lab Status Meanings

| Status | Meaning |
|---|---|
| **Start – Complete** | The lab is running and ready to use |
| **Cleanup – Complete** | The lab has been stopped and cleaned up. Click **Start** to use it again |

---

## Important Rules

- **Don't click Stop** unless your trainer asks you to. Stopping can clean up your resources.
- **Never share** your loginId, password or service account key with anyone.
- **Never upload** the service account key to GitHub, email or chat.
- **Delete resources you don't need** after each lab (VMs, Dataproc clusters, Dataflow jobs, Composer environments) to save budget.
- Labs are available only until the **Expires On** date on the lab card.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Status stuck on "Starting" | Wait 2–3 minutes and click the refresh icon |
| Google sign-in shows your personal account | Use an Incognito window, or sign out of other Google accounts |
| "Wrong password" error | Copy the password with the copy icon instead of typing it |
| Can't see the project in the Console | Click the project drop-down and select the lab project |
| "Permission denied" for a service | Check **Policies** in the panel and tell the trainer |
| Lab shows "Cleanup – Complete" | Click **Start** again and wait for "Start – Complete" |

If the problem continues, contact your trainer with a screenshot of the error.

---

**Trainer:** Deeksha Sharma
**LinkedIn:** [linkedin.com/in/deekshasharma191196](https://www.linkedin.com/in/deekshasharma191196/)
**Email:** [deekshasharma191196@gmail.com](mailto:deekshasharma191196@gmail.com)
