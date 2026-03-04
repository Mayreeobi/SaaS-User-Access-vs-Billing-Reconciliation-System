# SaaS User Access vs Billing Reconciliation System

[![SQL](https://img.shields.io/badge/SQL-SQLServer-blue)](https://github.com/Mayreeobi/Revenue-Reconciliation-System/blob/main/revenue%20reconciliation.sql)
[![Python](https://img.shields.io/badge/Python-3.8%2B-yellow)](https://github.com/Mayreeobi/Revenue-Reconciliation-System/blob/main/data%20reconcilation%20generator.ipynb)

> Automated SQL-based system that identifies and quantifies **monthly recurring revenue (MRR) leakage** caused by mismatches between user access and billing records in a SaaS environment.

#### [View SQL Script](https://github.com/Mayreeobi/Revenue-Reconciliation-System/blob/main/revenue%20reconciliation.sql)
---

## Table of Contents

- [Situation](#situation)
- [Task](#task)
- [Action](#action)
- [Result](#result)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)

---


## Situation

In SaaS companies, **user access and billing systems often drift out of sync**.  
When this happens, companies lose revenue, overcharge customers, or create compliance risks.

Examples:
- Users accessing paid features without an active subscription
- Deleted or inactive users still being billed
- Customers charged multiple times
- Trials that expire but retain full access

Manual reconciliation is slow, error-prone, and often done quarterly; long after revenue is lost.

---

## Task

Design and build an end-to-end automated reconciliation system that could:

- Cross-reference user access records against billing subscriptions at scale
- Detect and categorize every type of billing discrepancy, not just the obvious ones
- Quantify the financial impact of each discrepancy in real dollar terms
- Prioritize issues so finance teams know exactly what to fix first
- Replace the monthly manual audit with a push-button report

Scope: 500 users analyzed across 7 distinct issue categories.

---

## Action

### 1. Database Design (SQL Server)

Built a normalized relational database with two core tables; `user_accounts` and `billing_subscriptions`- connected via foreign key relationships and optimized with proper indexing for query performance.

Created **9 analytical SQL views** on top of that schema, each one implementing the reconciliation logic for a specific issue type. Separating concerns this way keeps the business rules readable, testable, and easy to update independently when billing logic changes.

### 2. 🧩 Seven Issue Categories: Logic, Impact, and Fix

---

#### Issue 1: Free Riders
**Definition:** Active users on paid plans with no active billing subscription - getting access without paying for it.

```sql
SELECT u.user_id, u.email, u.plan_type,
       CASE u.plan_type
           WHEN 'Starter'    THEN 29
           WHEN 'Pro'        THEN 99
           WHEN 'Enterprise' THEN 299
       END AS expected_monthly_revenue
FROM user_accounts u
LEFT JOIN billing_subscriptions b ON u.user_id = b.user_id
WHERE u.account_status = 'active'
  AND u.plan_type != 'Free'
  AND b.subscription_id IS NULL;
```

**Monthly Impact:** $3,971 in lost revenue  

**Typical Causes:** Payment declined but access not revoked, trial conversion failed, integration bug between provisioning and billing systems  

**Recommended Action:** Create subscription or suspend access within 24 hours

---

#### Issue 2: Ghost Subscriptions
**Definition:** Active billing records with no corresponding user account.(Charging for something that no longer exists.)

```sql
SELECT b.*
FROM billing_subscriptions b
LEFT JOIN user_accounts u 
  ON b.user_id = u.user_id
WHERE b.status IN ('active', 'trialing', 'past_due')
  AND u.user_id IS NULL;
```

**Monthly Impact:** $2,280 in refund liability    

**Typical Causes:** User deleted their account but cancellation didn't process, data sync failure, orphaned records from migrations    

**Recommended Action:** Cancel subscription and issue prorated refund

---

#### Issue 3: Plan Mismatches
**Definition:** User's access level doesn't match what they're being billed for.

```sql
SELECT u.user_id, u.plan_type,
       b.[plan] AS billing_plan
FROM user_accounts u
INNER JOIN billing_subscriptions b   
  ON u.user_id = b.user_id
WHERE u.plan_type != b.[plan]
  AND u.plan_type != 'Free';
```

**Monthly Impact:** $6,500 (under-charging = revenue loss; over-charging = refund liability + churn risk)   

**Typical Causes:** User upgraded in-app but billing wasn't updated, plan migration ran partially  

**Recommended Action:**  
 Under-charging → update billing, document if grandfathered.   
Over-charging → issue credit and fix immediately

---

#### Issue 4: Status Mismatches
**Definition:** Account status and billing status are out of sync in ways that cost money.

```sql
SELECT u.user_id, u.account_status, b.status
FROM user_accounts u
INNER JOIN billing_subscriptions b 
   ON u.user_id = b.user_id
WHERE (u.account_status = 'active'              AND b.status IN ('canceled', 'past_due'))
   OR (u.account_status IN ('inactive','deleted') AND b.status = 'active');
```

**Monthly Impact:** $4,241 in unrecovered revenue

**Critical Combinations:**
- Active user + Canceled billing → payment failure not enforced
- Active user + Past due billing → access granted despite non-payment
- Deleted user + Active billing → billing not canceled on account close
- Inactive user + Active billing → paying for zero usage

**Recommended Action:** Active/canceled → suspend access. Deleted/active → cancel + refund. Inactive/active → offer pause or cancellation.

---

#### Issue 5: Duplicate Subscriptions
**Definition:** A single user with multiple active billing subscriptions.

```sql
SELECT user_id,
       COUNT(*) AS subscription_count
FROM billing_subscriptions
WHERE status IN ('active', 'trialing')
GROUP BY user_id
HAVING COUNT(*) > 1;
```

**Monthly Impact:** $1,012 in overcharges + chargeback risk 

**Typical Causes:** User updated payment method by creating a new subscription instead of editing the existing one, system bug during checkout or plan upgrade  

**Recommended Action:** Cancel the duplicate, issue a credit, keep the highest-tier subscription

---

#### Issue 6: Free Plans with Active Billing
**Definition:** Users on the Free plan with an active billing subscription — charging customers for a tier that costs nothing.

```sql
SELECT u.user_id, b.billing_amount
FROM user_accounts u
INNER JOIN billing_subscriptions b   
   ON u.user_id = b.user_id
WHERE u.plan_type = 'Free'
  AND b.status = 'active';
```

**Monthly Impact:** Immediate refund risk regardless of amount  

**Recommended Action:** Cancel subscription and issue a full refund; same day

---

#### Issue 7: Trial Expiration Issues
**Definition:** Trials that ended but the user still has active paid access.

```sql
SELECT u.user_id, b.current_period_end
FROM user_accounts u
INNER JOIN billing_subscriptions b 
  ON u.user_id = b.user_id
WHERE b.status = 'trialing'
  AND b.current_period_end < CAST(GETDATE() AS DATE)
  AND u.account_status = 'active';
```

**Monthly Impact:** $1,265 in free access beyond the trial window 

**Recommended Action:** Convert to paid or suspend access.  
Only flag trials expired > 1 day to avoid catching conversions still in processing.

---

### 3. Business Rules and Healthy vs. Problem States

**Healthy states the system ignores:**
- Active user + Active billing + Matching plans ✅
- Free user + No billing ✅
- Deleted user + No billing ✅
- Inactive user + Canceled billing ✅

**Problem states the system flags:**
- Active user + No billing + Paid plan ❌ Free rider
- Active user + Canceled billing ❌ Payment not enforced
- Active billing + No user ❌ Ghost subscription
- User plan ≠ Billing plan ❌ Plan mismatch
- Multiple active subscriptions per user ❌ Duplicate billing

**Grace period logic built into queries:**
- Payment failures: 3-day grace period before flagging for suspension
- Trial conversions: only flag trials expired > 24 hours
- Account deletions: only flag ghost subscriptions from deletions > 7 days old

---

### 4. Prioritized Action Framework

Rather than handing finance a list of 171 problems with no direction, the system outputs a tiered action plan:

**Priority 1: Fix Today**  
Free users being charged, deleted users with active billing, duplicate subscriptions. These carry legal and compliance risk.

**Priority 2: Fix This Week**  
Free riders on paid plans, payment failures not enforced, ghost subscriptions. Direct, recoverable revenue loss.

**Priority 3: Fix This Month**  
Plan mismatches, expired trials. Revenue optimization and customer satisfaction improvements.

**Priority 4: Monitor and Prevent**  
Set alerts for new issues as they emerge, add reconciliation checks to nightly jobs, automate suspension triggers at key lifecycle events (signup, upgrade, cancellation).

---

### 5. Python ETL Pipeline

Developed an automated data pipeline using Pandas that handles three things: generates a realistic SaaS dataset that mirrors real-world billing complexity, populates the SQL database with validated data, and exports reconciliation results for downstream reporting.  
Data validation and error handling run at each stage so quality issues are caught before they reach the analysis.

---

## Result

| Metric | Value |
|---|---|
| Annual revenue at risk identified | **$260,388** |
| Monthly revenue at risk | **$21,699** |
| Billing discrepancies found | **171 across 7 categories** |
| Quick wins (recoverable in under 1 week) | **$3,292/month** |
| Monthly manual work eliminated | **20 - 40 hours** |
| System break-even point | **Recovering just 2% of identified leakage** |



**Full breakdown:**

| Issue Type | Count | Monthly Impact |
|---|---|---|
| Plan Mismatches (over + under charging) | 39 | $6,500 |
| Status Mismatches | 49 | $4,241 |
| Free Riders | 29 | $3,971 |
| Inactive Users with Active Billing | 20 | $2,430 |
| Ghost Subscriptions | 20 | $2,280 |
| Trial Expiration Issues | 15 | $1,265 |
| Duplicate Subscriptions | 18 | $1,012 |
| Free Users with Billing | 1 | $0 |
| **Total** | **171** | **$21,699/month → $260,388/year** |

Beyond the numbers, this system shifts finance from reactive quarterly audits to continuous automated monitoring. The team gets a prioritized action list the moment discrepancies appear; not months after they started compounding.

Finance gets clarity on what to fix and in what order. Product managers can spot patterns in plan mismatches that point to UX or pricing problems. Customer success can identify users who should be on a different plan before billing confusion drives them to churn.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| SQL Server | Database design, 9 analytical views, reconciliation logic |
| Python (Pandas) | ETL pipeline, data generation, validation |
| Excel | Data export and spot-check validation |

**SQL concepts used:** CTEs, multi-table JOINs, LEFT JOINs for gap detection, CASE statements, window functions, subqueries, HAVING clauses

---

## Project Structure

```
saas-user-billing-reconciliation/
│
├── sql/
│   ├── schema/
│   │   ├── create_user_accounts.sql
│   │   └── create_billing_subscriptions.sql
│   ├── views/
│   │   ├── vw_free_riders.sql
│   │   ├── vw_ghost_subscriptions.sql
│   │   ├── vw_plan_mismatches.sql
│   │   ├── vw_status_mismatches.sql
│   │   ├── vw_duplicate_billing.sql
│   │   ├── vw_trial_expiration.sql
│   │   ├── vw_billing_errors.sql
│   │   ├── vw_revenue_impact_summary.sql
│   │   └── vw_priority_action_list.sql
│   └── queries/
│       └── reconciliation_report.sql
│
├── python/
│   ├── data_generator.py
│
└── README.md
```
---
## 🔗 Project Assets 

[**SQL Queries**](https://github.com/Mayreeobi/Revenue-Reconciliation-System/blob/main/revenue%20reconciliation.sql)  | [**Python Data Generator**](https://github.com/Mayreeobi/Revenue-Reconciliation-System/blob/main/data%20reconcilation%20generator.ipynb)


---

*Built by Chinyere Obi - Data Analyst focused on turning messy operational data into clear financial decisions.*


