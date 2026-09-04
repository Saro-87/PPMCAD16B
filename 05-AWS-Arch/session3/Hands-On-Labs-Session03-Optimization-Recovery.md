# Hands-On Labs - Optimization and Recovery

## Lab outcome

Explore AWS cost-management and optimization tools, create a monthly cost budget, identify how AWS produces optimization recommendations, back up a small Amazon EBS volume with AWS Backup, verify the recovery point, and review or perform a restore.

By the end of the lab, you will understand these two operational loops:

```text
Optimization

Measure cost and utilization
        |
        v
Find opportunities
        |
        v
Evaluate risk and workload behavior
        |
        v
Optimize
        |
        v
Measure again
```

and:

```text
Recovery

Protect
   |
   v
Backup
   |
   v
Recovery point
   |
   v
Restore
   |
   v
Validate
```

> **Key idea:** Cost optimization is not simply choosing the cheapest resource, and backup is not the same as recovery. Optimization must preserve workload requirements, while recovery is proven only when restore and validation are successful.

---

# Prerequisites

- Access to the AWS Management Console.
- Billing and Cost Management console access for the training identity.
- Permission to use:
  - AWS Cost Explorer
  - AWS Budgets
  - AWS Compute Optimizer
  - AWS Trusted Advisor
  - Amazon EC2 / EBS
  - AWS Backup
  - IAM
  - Amazon CloudWatch metrics, where required for recommendation analysis
- Select one AWS Region and use it throughout the backup and restore exercises.

> **Note:** Billing and cost-management information is account-level rather than tied to a single Region. The selected Region matters mainly for regional resources such as EC2, EBS and AWS Backup.

---

# Part 1 - Optimization

# Lab 1 - Explore current AWS costs with Cost Explorer

AWS Cost Explorer helps you analyze AWS spending over time and break it down by dimensions such as service, account, Region and tags.

## Step 1 - Open Cost Explorer

1. Open **Billing and Cost Management**.
2. In the left navigation, choose **Cost Explorer**.
3. If Cost Explorer has never been enabled for the account, choose the option to enable it.

> After Cost Explorer is enabled, current-month cost data can take approximately 24 hours to become available. Older data can take longer to populate.

---

## Step 2 - Analyze cost by service

1. Set the date range to:

```text
Current month
```

2. Locate the grouping control.
3. Group the report by:

```text
Service
```

4. Identify the services contributing the most cost.

Examples could include:

```text
Amazon EC2
Amazon RDS
Amazon EBS
Amazon S3
Data Transfer
```

The exact services and values depend on the training account.

---

## Step 3 - Analyze cost by Region

Change the grouping to:

```text
Region
```

Observe whether cost is concentrated in one Region or distributed across multiple Regions.

If the training account uses several Regions, apply a filter for the Region being used by the class.

---

## Step 4 - Inspect the report controls

Locate controls such as:

```text
Date range
Granularity
Filters
Group by
Cost metric
```

Discuss how changing these controls changes the view of the same underlying cost data.

---

## Step 5 - Save the report

If the training identity is permitted to save Cost Explorer reports:

1. Choose the option to save the report.
2. Enter:

```text
training-account-monthly-cost
```

3. Save it.

> A new or lightly used AWS account may not contain enough historical data for meaningful analysis. If that happens, inspect the available Cost Explorer controls and continue with the remaining labs.

---

## Optimization checkpoint

Before moving on, answer:

```text
Which service currently contributes the most cost?

Which Region contributes the most cost?

Is the current-month cost trend increasing, decreasing or relatively stable?

What additional dimension would help explain the cost?
```

---

# Lab 2 - Understand cost allocation tags

Resource tags become much more useful for financial analysis when selected tag keys are activated as **cost allocation tags**.

For example:

```text
Environment = Production
Application = Orders
Owner = Platform-Team
CostCenter = CC101
```

can help answer:

```text
How much does Production cost?

How much does the Orders application cost?

Which team owns this spend?
```

---

## Step 1 - Locate cost allocation tags

1. Open **Billing and Cost Management**.
2. In the left navigation, choose **Cost allocation tags**.

Depending on the console layout, this is available under the cost-organization area.

3. Locate the section for user-defined cost allocation tags.

---

## Step 2 - Inspect available tags

Look for tag keys already created in the AWS account.

If a suitable training tag exists, select it.

Do not create or activate organization-wide billing tags unless permitted by the instructor.

---

## Step 3 - Understand activation

A normal AWS resource tag does not automatically become available for cost analysis.

Conceptually:

```text
Resource tag
     |
     v
Cost allocation tag activation
     |
     v
Cost Explorer / billing analysis
```

AWS notes that a newly created resource tag can take time to appear on the cost-allocation-tags page, and activation can also take time before the tag becomes available for billing analysis.

---

# Lab 3 - Inspect AWS Compute Optimizer

AWS Compute Optimizer analyzes resource configuration and utilization data and produces recommendations for supported resources.

Examples of supported recommendation areas include:

```text
EC2 instances
Auto Scaling groups
EBS volumes
Lambda functions
ECS services on Fargate
RDS DB instances
Idle resources
```

Availability depends on account configuration and supported resource types.

---

## Step 1 - Open Compute Optimizer

1. Open **AWS Compute Optimizer**.
2. If the account has not opted in, the console displays the getting-started experience.

If permitted by the instructor:

1. Choose **Get started**.
2. Review the account setup.
3. Opt in only for the scope approved for the training account.

> Recommendations are not produced instantly. Compute Optimizer needs sufficient utilization data before meaningful recommendations are available.

---

## Step 2 - Inspect EC2 recommendations

In the left navigation, choose:

```text
EC2 instances
```

If recommendations exist, open one.

Inspect:

```text
Current instance type
Finding
Recommended instance type
Performance risk
Estimated savings
Utilization graphs
```

Compute Optimizer can display multiple recommendation options for a resource rather than only one possible replacement.

---

## Step 3 - Understand the analysis window

For standard EC2 recommendations, Compute Optimizer uses recent historical utilization data. AWS documents a default analysis look-back period of **14 days** for EC2 recommendations.

An optional paid feature called **enhanced infrastructure metrics** can extend the analysis period.

Do not interpret a recommendation as a guarantee about future demand.

---

## Step 4 - Evaluate before changing anything

Before accepting a recommendation, record:

```text
Utilization period:

Observed peak workload:

Seasonal or batch behavior:

Availability requirement:

Expected workload growth:

Performance risk:

Architecture impact:

Estimated saving:
```

> **Do not resize a production workload only because a recommendation shows a lower-cost instance.** Historical utilization, peaks, failover capacity, workload growth and architectural requirements must also be considered.

---

# Lab 4 - Inspect Trusted Advisor

AWS Trusted Advisor provides checks across areas such as:

```text
Cost optimization
Performance
Security
Fault tolerance
Service limits
Operational excellence
```

The checks available to an account can depend on the AWS Support plan and other AWS services enabled in the account.

---

## Step 1 - Open Trusted Advisor

1. Open **AWS Trusted Advisor**.
2. In the navigation pane, choose:

```text
Cost optimization
```

3. Inspect any checks available to the training account.

Depending on the account, examples may relate to:

```text
EC2
EBS
RDS
Savings Plans
Idle or underutilized resources
```

---

## Step 2 - Inspect one recommendation

For one available check, identify:

```text
Resource

Current state

Recommended action

Potential saving

Last refresh time

Any dependencies or prerequisites
```

Do not make a change unless the instructor explicitly requests it.

---

# Lab 5 - Compare optimization tools

At this point you have seen several different tools.

Use this simplified distinction:

| Tool | Main purpose |
|---|---|
| **Cost Explorer** | Understand where money is being spent |
| **Cost allocation tags** | Attribute cost to applications, teams or environments |
| **Compute Optimizer** | Analyze resource utilization and recommend resource changes |
| **Trusted Advisor** | Surface checks and recommendations across cost and other operational categories |
| **AWS Budgets** | Alert when spend or usage approaches a defined threshold |

The tools are complementary rather than replacements for one another.

---

# Lab 6 - Create a monthly AWS cost budget

AWS Budgets allows you to define a spending target and generate alerts when actual or forecasted cost reaches a threshold.

## Step 1 - Start budget creation

1. Open:

   **Billing and Cost Management → Budgets**

2. Choose:

```text
Create budget
```

3. Choose the option to customize the budget.
4. Select:

```text
Cost budget
```

---

## Step 2 - Configure the budget

Configure:

| Setting | Value |
|---|---|
| Budget name | `architecture-training-budget` |
| Period | `Monthly` |
| Budget amount | A small value approved for the training account |

Do not use an arbitrary training value if the account owner has defined spending controls.

---

## Step 3 - Configure the alert

Create an alert threshold:

```text
80%
```

Configure it against:

```text
Actual cost
```

Use:

```text
% of budgeted amount
```

Enter an approved learner or trainer email address.

Review the configuration and create the budget.

---

## Step 4 - Understand budget behavior

Conceptually:

```text
Monthly budget
     |
     v
Actual AWS cost
     |
     | reaches 80%
     v
Budget alert
     |
     v
Email notification
```

> AWS Budgets alerts are not intended as an immediate real-time control. Do not wait during the class for the threshold to be reached.

---

# Part 2 - Recovery

# Lab 7 - Understand RPO and RTO

Before creating a backup, understand the two recovery objectives.

## RPO - Recovery Point Objective

**RPO** describes how much data loss the business can tolerate.

Example:

```text
RPO = 15 minutes
```

means the recovery strategy should normally allow the system to recover to a point no more than approximately 15 minutes before the disruption.

Conceptually:

```text
Failure at 10:00
        |
        v

Acceptable recovery point >= 09:45
```

---

## RTO - Recovery Time Objective

**RTO** describes the target time within which the service should be restored after a disruption.

Example:

```text
RTO = 2 hours
```

means the recovery process should be designed to restore the required service within the defined two-hour target.

---

## RPO vs RTO

```text
RPO -> How much data can we lose?

RTO -> How long can recovery take?
```

These objectives influence:

```text
Backup frequency
Replication strategy
Retention
Cross-Region design
Cross-account protection
Automation
Restore procedure
Validation frequency
```

---

# Lab 8 - Create a small EBS volume

The lab uses an EBS volume because it is simple to create, back up and restore.

## Step 1 - Create the volume

1. Open:

   **EC2 → Elastic Block Store → Volumes**

2. Choose:

```text
Create volume
```

3. Configure:

| Setting | Value |
|---|---|
| Volume type | `gp3` |
| Size | `1 GiB` |
| Availability Zone | Any AZ in the selected Region |

4. Add the tag:

```text
Name = backup-lab-volume
```

5. Choose **Create volume**.

---

## Step 2 - Validate

Confirm the volume state becomes:

```text
Available
```

The volume does not need to be attached to an EC2 instance for this introductory backup exercise.

> Because the volume contains no application data, this lab validates the AWS Backup workflow rather than file-level data integrity. A production recovery test must validate application or data correctness after restore.

---

# Lab 9 - Create an AWS Backup vault

An AWS Backup vault is a logical container used to organize recovery points managed by AWS Backup.

Conceptually:

```text
AWS Backup
    |
    v
Backup vault
    |
    +---- Recovery point A
    +---- Recovery point B
    +---- Recovery point C
```

---

## Step 1 - Create the vault

1. Open **AWS Backup**.
2. In the left navigation, choose:

```text
Backup vaults
```

3. Choose:

```text
Create backup vault
```

4. Enter:

```text
architecture-training-vault
```

5. For encryption, select the default AWS Backup KMS key unless the instructor has provided another approved key.
6. Create the vault.

---

# Lab 10 - Create an on-demand EBS backup

AWS Backup supports scheduled backup plans as well as backups initiated manually.

For the first exercise, use an **on-demand backup** so that the backup can be observed immediately.

---

## Step 1 - Start the backup

1. Open **AWS Backup**.
2. From the dashboard, choose:

```text
Create on-demand backup
```

You can also reach this action from **Protected resources**.

---

## Step 2 - Select the resource

Configure:

| Setting | Value |
|---|---|
| Resource type | `EBS` |
| Volume ID | The volume tagged `backup-lab-volume` |

Confirm that the selected volume is the one created in Lab 8.

---

## Step 3 - Configure the backup

For the backup window, choose:

```text
Create backup now
```

For retention, configure a short retention period permitted by the training-account policy.

For the backup vault, select:

```text
architecture-training-vault
```

For the IAM role:

- Use the existing AWS Backup default role if one is already available and approved.
- If the console offers to create the default role because one does not exist, allow AWS Backup to create it if permitted by the training policy.

Review the configuration and choose:

```text
Create on-demand backup
```

---

# Lab 11 - Monitor the backup job

## Step 1 - Open backup jobs

Open:

```text
AWS Backup → Jobs → Backup jobs
```

Locate the job for the EBS volume.

Observe the job status.

Possible states during the workflow can include:

```text
Created
Pending
Running
Completed
Failed
```

Wait until the job reaches:

```text
Completed
```

before validating the recovery point.

---

## Step 2 - Investigate failures if required

If the backup does not complete, inspect:

```text
Status
Status message
Resource
IAM role
Backup vault
Region
```

Typical troubleshooting questions include:

```text
Does the AWS Backup role have the required permissions?

Is the correct resource selected?

Is the resource in the expected Region?

Is the backup vault accessible?

Is a KMS permission preventing the backup?
```

Do not proceed to restore until a completed recovery point exists.

---

# Lab 12 - Validate the recovery point

A successful backup job should create a recovery point in the selected backup vault.

## Step 1 - Open the vault

Open:

```text
AWS Backup → Backup vaults → architecture-training-vault
```

Locate the recovery point for:

```text
backup-lab-volume
```

---

## Step 2 - Record recovery metadata

Record the values available in the console:

```text
Recovery point:

Resource type:

Resource ARN:

Creation time:

Status:

Expiry / lifecycle information:
```

The exact labels shown can vary by resource type and console view.

---

## What has been proven?

At this point you have proven:

```text
AWS Backup successfully created a recovery point.
```

You have **not yet proven**:

```text
The resource can be restored successfully.

The recovered data is valid.

The application works after recovery.

The required RTO can be achieved.
```

That distinction is important.

---

# Lab 13 - Review the EBS restore process

## Step 1 - Start restore

1. Select the EBS recovery point.
2. Choose:

```text
Restore
```

3. Review the restore settings presented for the EBS resource.

AWS Backup restores the recovery point as a new resource rather than overwriting the original EBS volume.

Conceptually:

```text
Original EBS volume
        |
        | backup
        v
Recovery point
        |
        | restore
        v
New EBS volume
```

---

## Step 2 - Review the restore configuration

Inspect the available configuration carefully.

Depending on the current EBS restore workflow, settings may include characteristics such as:

```text
Availability Zone
Volume type
Encryption
IAM restore role
```

Do not submit the restore yet if the instructor wants only a workflow review.

---

# Lab 14 - Perform and validate a restore

This lab is recommended if the training account permits restoration.

## Step 1 - Start the restore

From the recovery point:

1. Choose **Restore**.
2. Configure the target EBS settings approved for the training account.
3. Select the appropriate restore IAM role.
4. Start the restore job.

---

## Step 2 - Monitor the restore job

Open:

```text
AWS Backup → Jobs → Restore jobs
```

Find the restore job.

Wait until it reaches:

```text
Completed
```

---

## Step 3 - Find the restored volume

Open:

```text
EC2 → Elastic Block Store → Volumes
```

Locate the newly restored EBS volume.

Confirm that its state becomes:

```text
Available
```

---

## Step 4 - Validate expected configuration

Compare the restored resource with the recovery requirement.

Record:

```text
Expected size:

Restored size:

Expected volume type:

Restored volume type:

Expected encryption:

Restored encryption:

Expected Availability Zone:

Restored Availability Zone:

Validation result:
```

Do not assume every attribute, including tags, will necessarily be reproduced exactly unless the restore workflow and resource behavior explicitly support it.

---

# Recovery checkpoint

Consider an order system with:

```text
RPO = 15 minutes
RTO = 2 hours
```

Complete:

```text
Backup or replication frequency:

Backup type / replication mechanism:

Retention:

Primary Region:

Recovery Region:

Target account:

Recovery point selection process:

Restore steps:

Infrastructure dependencies:

Application startup steps:

Data validation steps:

Application validation steps:

DNS / traffic failover steps:

Estimated recovery duration:

Owner of the recovery decision:
```

---
