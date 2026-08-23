## AWS Application Request Flow

**Headline: A user request to `https://demo.com` is first resolved through DNS to the ALB, then the ALB forwards the request to healthy EC2 instances, which communicate with RDS when database access is required.**

### Application architecture

```text
User
  |
  v
Route 53
  |
  v
Application Load Balancer
  |
  v
Target Group
  |
  +--------+
  |        |
 EC2      EC2
  \        /
   \      /
    RDS MySQL
```

EC2 instances run inside an **Auto Scaling Group**, the ALB distributes traffic across healthy instances, and the EC2 application connects to RDS MySQL for database operations.

---

# Q1. What happens when a user opens `https://demo.com`?

### 1. Browser receives the URL

The user enters:

```text
https://demo.com
```

The browser needs to find the IP address associated with `demo.com`.

### 2. Local name resolution is checked

The operating system may first check local name-resolution sources, including the hosts file:

Linux:

```text
/etc/hosts
```

Windows:

```text
C:\Windows\System32\drivers\etc\hosts
```

If there is no matching entry, normal DNS resolution continues.

### 3. DNS resolver is contacted

The system sends the DNS query to its configured DNS resolver, often provided by:

* ISP
* Organization
* Router
* Public DNS service

If the resolver already has the answer cached, it returns it immediately.

Otherwise, DNS resolution continues.

---

# 4. DNS hierarchy is followed

For:

```text
demo.com
```

DNS resolution conceptually follows:

```text
Root DNS
   |
   v
.com TLD DNS
   |
   v
Authoritative Name Servers for demo.com
```

Here:

```text
.com
```

is the **Top-Level Domain (TLD)**.

Examples:

```text
.com
.in
.org
.edu
```

If the domain is hosted in Route 53, the domain's authoritative name servers point to the Route 53 hosted zone.

---

# 5. Route 53 resolves `demo.com`

In Route 53, you would normally have a record such as:

```text
demo.com
    |
    v
Alias Record
    |
    v
Application Load Balancer
```

For an AWS ALB, an **Alias A/AAAA record** is generally used rather than manually storing the load balancer's IP addresses.

---

# 6. Browser connects to the ALB

DNS returns the address associated with the ALB.

Because the URL is:

```text
https://demo.com
```

the browser establishes an HTTPS connection to the ALB, normally on:

```text
TCP 443
```

The ALB listener receives the request.

Example:

```text
ALB Listener
443 / HTTPS
```

---

# 7. ALB forwards traffic to the Target Group

The ALB listener has a rule such as:

```text
HTTPS : 443
    |
    v
Target Group
```

The target group contains the EC2 instances running the application:

```text
Target Group
   |
   +--> EC2-1
   |
   +--> EC2-2
```

The ALB sends traffic only to targets that are considered healthy by its health checks.

---

# 8. EC2 processes the application request

One of the EC2 instances receives the request.

For example:

```text
User
 |
Route 53
 |
ALB
 |
Target Group
 |
EC2
```

The application running on EC2 processes the request.

If it needs information from the database:

```text
EC2
 |
 v
RDS MySQL
```

RDS is **not directly accessed by the browser or ALB**.

The flow is:

```text
User
  |
  v
Route 53
  |
  v
ALB
  |
  v
EC2 Application
  |
  v
RDS MySQL
```

---

# 9. Response returns to the user

The response follows the reverse application path:

```text
RDS
 |
 v
EC2
 |
 v
ALB
 |
 v
Internet
 |
 v
Browser
```

The browser then displays the application page.

---

# Q2. How should the VPC be designed?

A good starting architecture is:

```text
                     Internet
                        |
                        v
                       IGW
                        |
              +-------------------+
              |       VPC         |
              |                   |
              | Public Subnets    |
              |                   |
              |  AZ-A      AZ-B   |
              |   |          |    |
              |   +--- ALB ---+   |
              |                   |
              | Private Subnets   |
              |                   |
              |  AZ-A      AZ-B   |
              |  EC2        EC2   |
              |    \        /     |
              |     \      /      |
              |      RDS MySQL    |
              +-------------------+
```

## 1. Decide the VPC CIDR

First estimate:

* Number of application servers
* Expected Auto Scaling growth
* Number of environments
* Number of AWS services requiring IP addresses
* Future expansion

Then choose an appropriate VPC CIDR.

Example:

```text
VPC
10.0.0.0/16
```

For a lab this gives plenty of room, although production CIDR planning should also consider connectivity with existing networks to avoid overlaps.

---

# 2. Create subnets across two Availability Zones

For HA:

```text
VPC: 10.0.0.0/16
```

Example:

```text
AZ-A
Public Subnet 1  : 10.0.1.0/24
Private Subnet 1 : 10.0.11.0/24

AZ-B
Public Subnet 2  : 10.0.2.0/24
Private Subnet 2 : 10.0.12.0/24
```

Public subnets:

```text
ALB
NAT Gateway, if required
```

Private subnets:

```text
EC2 application servers
RDS
```

---

# 3. Create an Internet Gateway

Create an:

```text
Internet Gateway
```

and attach it to the VPC.

```text
Internet
   |
   v
IGW
   |
   v
VPC
```

---

# 4. Configure public subnet routing

Create a public route table.

Add:

```text
Destination       Target

10.0.0.0/16       local
0.0.0.0/0         Internet Gateway
```

Associate this route table with:

```text
Public Subnet 1
Public Subnet 2
```

A subnet becomes effectively public when its route table has a route to the Internet Gateway and the resource has the required public reachability.

---

# 5. Configure private subnet routing

The EC2 instances should not be directly accessible from the Internet.

If the application servers need outbound Internet access, for example to download packages, place a NAT Gateway in a public subnet.

Flow:

```text
Private EC2
    |
    v
Private Route Table
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

Private route table:

```text
Destination       Target

10.0.0.0/16       local
0.0.0.0/0         NAT Gateway
```

---

# 6. Place the ALB in public subnets

The internet-facing ALB should use:

```text
Public Subnet 1
Public Subnet 2
```

across two Availability Zones.

```text
Internet
   |
   v
ALB
 / \
AZ-A AZ-B
```

---

# 7. Place EC2 in private subnets

The Auto Scaling Group launches EC2 instances into:

```text
Private Subnet 1
Private Subnet 2
```

The instances normally do not need public IP addresses.

```text
ALB
 |
 v
EC2 private IP
```

---

# 8. Place RDS in private subnets

Create an RDS DB subnet group containing private subnets across at least two Availability Zones.

Conceptually:

```text
Private Subnet AZ-A
        +
Private Subnet AZ-B
        |
        v
RDS DB Subnet Group
```

For production HA, RDS can then be configured as **Multi-AZ** if required.

---

# 9. Configure Security Groups

### ALB Security Group

Inbound:

```text
443 from 0.0.0.0/0
```

Optionally:

```text
80 from 0.0.0.0/0
```

if you want HTTP to redirect to HTTPS.

### EC2 Security Group

Inbound application port only from the **ALB security group**.

Example:

```text
Port 80
Source: ALB-SG
```

or, if your application runs on 8080:

```text
Port 8080
Source: ALB-SG
```

Do not expose the application port directly to the Internet.

### RDS Security Group

Allow MySQL only from the EC2 application security group:

```text
Port 3306
Source: EC2-SG
```

So the security path becomes:

```text
Internet
   |
  443
   |
 ALB-SG
   |
  80/8080
   |
 EC2-SG
   |
  3306
   |
 RDS-SG
```

---

# Final architecture

```text
                         Internet
                            |
                         Route 53
                            |
                            v
                    Internet Gateway
                            |
             +--------------+--------------+
             |                             |
      Public Subnet AZ-A            Public Subnet AZ-B
             |                             |
             +-------------+---------------+
                           |
                           ALB
                           |
                       Target Group
                      /            \
                     /              \
            Private Subnet      Private Subnet
                 AZ-A               AZ-B
                  |                  |
                EC2-1              EC2-2
                  \                  /
                   \                /
                     RDS MySQL
```

The core design principle is simple:

```text
PUBLIC
------
ALB
NAT Gateway

PRIVATE
-------
EC2
RDS
```

And the request path is:

```text
demo.com
   ↓
DNS / Route 53
   ↓
ALB
   ↓
Target Group
   ↓
EC2
   ↓
RDS
   ↓
EC2
   ↓
ALB
   ↓
User
```

One small correction to your original sequence: **Route 53 does not receive the actual web request.** Route 53 only participates in the **DNS lookup**. Once DNS resolution is complete, the browser connects directly to the ALB address returned through DNS.
