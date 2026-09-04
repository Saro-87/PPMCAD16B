# Hands-On Lab - Designing AWS Architecture

## Lab outcome

Design and review an AWS architecture for a ticket-booking workload.

The completed lab should contain:

- A requirements sheet.
- An architecture diagram.
- Failure and recovery notes.
- A short architecture decision record.

The objective is to design from requirements first rather than selecting AWS services first.

---

# Scenario

A company is launching an online ticket-booking platform.

- Traffic is normally low but rises sharply when ticket sales open.
- Customers access the service from several countries.
- Static event images must load quickly.
- A customer must not receive two bookings for one payment.
- Payment is processed by an external provider.
- The service must survive the failure of one Availability Zone.
- Deleted or corrupted booking data must be recoverable.
- Application servers must not contain permanent session or business data.

Use one of the following for the architecture diagram:

- diagrams.net
- PowerPoint shapes
- Whiteboard
- Paper

---

# Lab 1 - Convert the scenario into requirements

Before selecting AWS services, first translate the business scenario into technical requirements.

This helps ensure that architecture choices are based on the workload needs rather than on familiar AWS services.

| Area | Requirement |
|---|---|
| Users and locations | Customers access the platform from several countries |
| Normal and peak demand | Traffic is normally low but rises sharply when ticket sales open |
| Availability | The service must survive the failure of one Availability Zone |
| Data consistency | Duplicate booking requests must not create duplicate confirmed bookings |
| Security and sensitive data | Booking/customer data and application secrets must be protected; payment is handled by an external provider |
| RPO | **Assumption - 15 minutes** |
| RTO | **To be filled during the lab** |
| External dependencies | External payment provider |
| Cost constraint | **To be filled during the lab** |

## What to complete

For the remaining fields, write a reasonable assumption.

Example:

RTO: Assumption - 2 hours

Cost constraint: Assumption - scale with demand instead of permanently provisioning for peak traffic

---

# Lab 2 - Draw the request path

Start with the high-level customer request flow before adding implementation details.

Add these zones to the diagram:

1. Customers.
2. DNS.
3. Edge delivery for static content.
4. Web application protection.
5. Public application entry point.
6. Replaceable compute across at least two Availability Zones.
7. Shared session or cache layer if required.
8. Durable booking database.
9. Asynchronous processing.
10. Monitoring and logging.

A possible request path to consider is:

```text
Customers
    |
    v
Route 53
    |
    v
CloudFront
    |
    +--------> S3 static event images
    |
    v
AWS WAF
    |
    v
ALB / API Gateway
    |
    v
Application compute
    |
    +--------> Cache / session store
    |
    +--------> Booking database
    |
    +--------> Queue / event service
```

This is only an example.

Your final architecture may use different AWS services if you can justify them.

---

# Lab 3 - Select AWS services

Choose AWS services for the main architecture responsibilities.

Most of the common mappings are pre-filled below. Two responsibilities are intentionally left open and should be completed during the lab.

| Responsibility | Selected AWS service | Reason |
|---|---|---|
| DNS | Amazon Route 53 | Provides DNS routing for the application |
| Global static content | Amazon CloudFront | Delivers static content closer to users through edge locations |
| Web application protection | AWS WAF | Helps protect the public application from common web attacks and unwanted traffic |
| Web/API entry point | Application Load Balancer or Amazon API Gateway | Provides the public entry point and routes requests to application compute |
| Compute | Amazon EC2 Auto Scaling / Amazon ECS / AWS Lambda | Runs application workloads and supports replaceable compute |
| Scaling | Auto Scaling | Adds or removes compute capacity based on demand |
| Relational or transactional data | Amazon RDS / Amazon Aurora | Stores booking and transactional data with relational consistency |
| Cache/session state | Amazon ElastiCache | Stores shared session or frequently accessed data outside application instances |
| Queue or event routing | **To be filled during the lab** | |
| Object storage | Amazon S3 | Stores static files and uploaded event content durably |
| Encryption | AWS KMS | Provides encryption key management for supported AWS services |
| Secrets | AWS Secrets Manager | Stores application credentials and secrets securely |
| Metrics and logs | Amazon CloudWatch | Provides metrics, logs, alarms and operational monitoring |
| Audit activity | AWS CloudTrail | Records AWS API and account activity for auditing |
| Backup and recovery | **To be filled during the lab** | |

## To be completed during the lab

Select appropriate AWS services for:

- Queue or event routing
- Backup and recovery

More than one answer may be valid.

For each choice, explain which workload requirement it supports.

---

# Lab 4 - Make the compute tier stateless

Application compute must be replaceable.

A stateless compute tier means that an application instance can fail or be replaced without losing the only copy of business data, session data, logs, configuration or secrets.

For each item, identify the external AWS service that stores it.

| Data / state | External service |
|---|---|
| User session | Amazon ElastiCache |
| Uploaded event image | Amazon S3 |
| Booking record | Amazon RDS / Amazon Aurora |
| Application logs | Amazon CloudWatch Logs |
| Application configuration | **To be filled during the lab** |
| Secrets | **To be filled during the lab** |

## To be completed during the lab

Choose appropriate AWS services for:

- Application configuration
- Secrets

The target design should allow an application instance to fail and be replaced without losing business data.

---

# Lab 5 - Design the booking consistency boundary

The booking path is the most important consistency-sensitive part of the system.

The architecture must prevent:

```text
Two confirmed bookings for the same seat
```

or:

```text
Two bookings for one successful payment
```

Do not solve this only with caching.

Identify how the system of record protects booking consistency.

Possible mechanisms to consider include:

```text
Database transaction

Unique constraint

Conditional write

Optimistic locking

Idempotency key

Seat reservation / locking strategy
```

Record your design:

```text
System of record:

Consistency mechanism:

Duplicate-request protection:

Seat reservation approach:

What happens when two users attempt the same seat simultaneously:
```

The exact mechanism depends on the database and architecture selected.

---

# Lab 6 - Design payment handling

Payment is processed by an external provider.

Add the payment provider to the architecture diagram.

Consider this failure:

```text
Application
    |
    | payment request
    v
Payment Provider
    |
    | payment succeeds
    X
Response times out
```

The application may not know whether payment succeeded.

Answer:

```text
How is payment retry made idempotent?

How is payment status reconciled?

What happens if payment succeeds but booking confirmation fails?

What happens if booking succeeds but payment confirmation is delayed?

Where is external payment reference stored?

How is a duplicate payment attempt prevented?
```

Do not assume that retrying the payment request is always safe.

---

# Lab 7 - Add asynchronous processing

Not every action should be completed in the customer's synchronous booking request.

After the booking is successfully confirmed, publish an event or message.

For example:

```text
Booking confirmed
        |
        +----> Confirmation notification
        |
        +----> Loyalty / downstream fulfillment
        |
        +----> Analytics
```

Avoid treating core seat reservation as casually asynchronous unless your design explicitly provides a reliable consistency mechanism.

---

## Design the messaging behavior

Record how the architecture handles:

```text
Duplicate messages:

Failed messages:

Retry behavior:

Dead-letter queue:

Slow consumer:

Consumer outage:

Event ordering where required:

Idempotent consumer behavior:
```

Add a dead-letter queue or equivalent failure destination where appropriate.

Example:

```text
Queue
  |
  v
Consumer
  |
  X repeated failures
  |
  v
DLQ
```

---

# Lab 8 - Apply security layers

Add the following controls to the architecture.

1. IAM role used by the application instead of long-lived access keys.
2. AWS WAF protecting the public-facing application path where appropriate.
3. Security group for the public application entry point where applicable.
4. Application security group accepting traffic only from the entry point.
5. Database security group accepting traffic only from the application tier.
6. KMS-backed encryption for stored data where required.
7. Secrets Manager or Systems Manager Parameter Store for secrets and configuration where appropriate.
8. A VPC endpoint for supported AWS services when private service access is required.
9. CloudTrail for AWS API and audit activity.
10. CloudWatch for workload metrics, logs, alarms and operational monitoring.

The intended network pattern is:

```text
Internet
   |
   v
Public entry point
   |
   v
Application tier
   |
   v
Database tier
```

The database should not be directly reachable from the internet.

---

# Lab 9 - Add observability

Do not add only a generic "CloudWatch" box.

Identify the signals you would monitor.

At minimum consider:

```text
Request rate

Request latency

HTTP 4xx

HTTP 5xx

Booking failures

Payment failures

Application errors

Queue depth

Age of oldest message

Database connections

Database load

Cache hit ratio

Failed authentication events
```

Record:

```text
Metric / signal:

Why it matters:

Alarm threshold or condition:

Operational response:
```

Choose at least five signals.

---

# Lab 10 - Validate Multi-AZ resilience

The scenario requires the service to survive the failure of one Availability Zone.

Verify that this requirement is addressed across the full architecture, not only the compute layer.

For every critical stateful component, answer:

```text
Is it deployed across Availability Zones?

What happens if one Availability Zone fails?

Is failover automatic?

Is data lost?

Is manual action required?
```

Consider components such as:

```text
Application compute

Load balancer

Relational database

Cache / session store

Messaging

Object storage
```

A design with compute in two Availability Zones but a single-AZ database does not meet the requirement.

---

# Lab 11 - Identify single points of failure

For each important component, complete:

```text
Component:

Failure mode:

What happens if it fails:

Is it Multi-AZ:

Is data lost:

Automatic failover:

Manual action required:

Recovery mechanism:
```

Challenge every box in the architecture with:

```text
What happens if this fails?
```

Modify the design wherever the answer exposes an unacceptable single point of failure.

---

# Lab 12 - Challenge the design

Answer each question directly on the diagram or in accompanying notes.

1. What happens when one Availability Zone fails?
2. What happens when one compute instance fails?
3. How is double booking prevented at the system-of-record level if the same request is received twice?
4. What happens if the payment provider times out after charging the customer?
5. How is payment retry made idempotent?
6. What prevents direct database access from the internet?
7. What becomes the next bottleneck when compute scales?
8. What happens if the queue consumer becomes slow?
9. What happens if the database reaches connection limits?
10. How is accidentally deleted booking data restored?
11. How is recovery validated?
12. Which component creates the largest continuous cost?
13. Which component is most difficult to scale?
14. Which component has the greatest business impact if unavailable?

Modify the design wherever an answer is incomplete.

---