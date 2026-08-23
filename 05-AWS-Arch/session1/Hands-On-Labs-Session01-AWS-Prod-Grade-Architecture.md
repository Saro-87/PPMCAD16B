**Headline:** Build a secure AWS web application with only the Application Load Balancer exposed publicly, while the application servers and RDS database remain in private subnets.

# Hands-On Lab - Secure AWS Three-Tier Architecture

## Lab outcome

Build this architecture:

```text
                         INTERNET
                            |
                            |
                     Internet Gateway
                            |
              +-------------+-------------+
              |                           |
        public-web-a                 public-web-b
        10.20.1.0/24                10.20.2.0/24
              |                           |
              +----------- ALB -----------+
                          alb-sg
                            |
                         HTTP 80
                            |
              +-------------+-------------+
              |                           |
          web-a                         web-b
     private-app-a                 private-app-b
     10.20.11.0/24                10.20.12.0/24
          app-sg                        app-sg
              \                           /
               \                         /
                    MySQL TCP 3306
                          |
                        db-sg
                          |
                      Amazon RDS
                  Public access: No
```

The ALB is internet-facing, while the EC2 instances use private IP addresses. An ALB target registered by instance ID receives traffic through the instance's primary private IP address.

RDS will also remain private and will accept database connections only from the application security group. AWS recommends keeping RDS databases private when public access is not required.

---

# Lab 1 - Create the VPC

1. Open **VPC → Your VPCs → Create VPC**.
2. Select **VPC only**.
3. Enter:

   * Name: `secure-app-vpc`
   * IPv4 CIDR: `10.20.0.0/16`
4. Create the VPC.

---

# Lab 2 - Create the subnets

Open **VPC → Subnets → Create subnet**.

Create four subnets across two Availability Zones:

| Name            | Availability Zone |            CIDR | Purpose |
| --------------- | ----------------- | --------------: | ------- |
| `public-web-a`  | First AZ          |  `10.20.1.0/24` | ALB     |
| `public-web-b`  | Second AZ         |  `10.20.2.0/24` | ALB     |
| `private-app-a` | First AZ          | `10.20.11.0/24` | EC2     |
| `private-app-b` | Second AZ         | `10.20.12.0/24` | EC2     |

Enable **Auto-assign public IPv4 address** only on:

```text
public-web-a
public-web-b
```

Do not enable it for the private application subnets.

---

# Lab 3 - Add internet routing

## Create the Internet Gateway

1. Open **Internet gateways → Create internet gateway**.
2. Name it:

```text
secure-app-igw
```

3. Attach it to:

```text
secure-app-vpc
```

## Create the public route table

1. Open **Route tables → Create route table**.
2. Enter:

   * Name: `public-rt`
   * VPC: `secure-app-vpc`
3. Add:

```text
Destination: 0.0.0.0/0
Target: secure-app-igw
```

4. Associate:

```text
public-web-a
public-web-b
```

A subnet uses its associated route table to determine where traffic is sent.

---

# Lab 4 - Create a NAT Gateway for the private EC2 instances

The EC2 instances need outbound internet access to install the web server, but they should not have public IP addresses.

1. Open **VPC → NAT gateways → Create NAT gateway**.
2. Enter:

   * Name: `secure-app-nat`
   * Subnet: `public-web-a`
   * Connectivity type: **Public**
3. Allocate an Elastic IP.
4. Create the NAT Gateway.
5. Wait until its state becomes **Available**.

Create another route table:

```text
Name: private-app-rt
VPC: secure-app-vpc
```

Add:

```text
Destination: 0.0.0.0/0
Target: secure-app-nat
```

Associate:

```text
private-app-a
private-app-b
```

AWS supports routing `0.0.0.0/0` from a private subnet to a NAT Gateway for outbound internet access.

For this lab, one NAT Gateway is sufficient. A more resilient production design can use one NAT Gateway per Availability Zone.

---

# Lab 5 - Create security groups

Create three security groups.

## ALB security group

Create:

```text
Name: alb-sg
VPC: secure-app-vpc
```

Inbound:

```text
HTTP | TCP | 80 | Anywhere-IPv4
```

---

## Application security group

Create:

```text
Name: app-sg
VPC: secure-app-vpc
```

Inbound:

```text
HTTP | TCP | 80 | Source: alb-sg
```

Do not allow:

```text
HTTP | 80 | 0.0.0.0/0
```

and do not add SSH access.

---

## Database security group

Create:

```text
Name: db-sg
VPC: secure-app-vpc
```

Inbound:

```text
MySQL/Aurora | TCP | 3306 | Source: app-sg
```

The security path should therefore be:

```text
Internet
   |
   | HTTP 80
   v
 alb-sg
   |
   | HTTP 80
   v
 app-sg
   |
   | MySQL 3306
   v
 db-sg
```

---

# Lab 6 - Create the RDS database

First create a DB subnet group.

Open:

**RDS → Subnet groups → Create DB subnet group**

Enter:

```text
Name: secure-app-db-subnets
VPC: secure-app-vpc
```

For this small lab, use the two private subnets:

```text
private-app-a
private-app-b
```

A DB subnet group used by RDS spans subnets across Availability Zones.

Now open:

**RDS → Databases → Create database**

Select:

```text
Standard create
Engine: MySQL
```

Configure:

```text
DB instance identifier: secure-app-db
Master username: admin
```

Set your own password.

Under **Connectivity**:

```text
VPC: secure-app-vpc

DB subnet group:
secure-app-db-subnets

Public access:
No

Security group:
db-sg
```

Create the database.

Wait until:

```text
Status: Available
```

Then copy the **RDS endpoint**.

AWS recommends `Public access = No` when the database does not need direct internet access.

---

# Lab 7 - Launch the first application server

Open:

**EC2 → Instances → Launch instances**

Configure:

```text
Name: web-a
AMI: Amazon Linux
Instance type: t3.micro

VPC: secure-app-vpc
Subnet: private-app-a

Auto-assign public IP: Disable

Security group:
app-sg

Key pair:
Proceed without a key pair
```

Under **Advanced details → User data**, enter the following.

Replace:

```text
RDS-ENDPOINT
```

with the endpoint copied from RDS.

```bash
#!/bin/bash

dnf install -y httpd php php-mysqli

systemctl enable --now httpd

cat > /var/www/html/index.php <<'EOF'
<?php

$server = "RDS-ENDPOINT";
$username = "admin";
$password = "YOUR-RDS-PASSWORD";

$conn = new mysqli($server, $username, $password);

echo "<h1>Secure AWS Application</h1>";
echo "<h2>Response from web-a</h2>";

if ($conn->connect_error) {
    echo "<p>Database connection: FAILED</p>";
} else {
    echo "<p>Database connection: SUCCESS</p>";
}

?>
EOF
```

Replace:

```text
YOUR-RDS-PASSWORD
```

with the database password used earlier.

> This direct password placement is intentionally simplified for the lab. A production application should not store database credentials directly in EC2 user data.

---

# Lab 8 - Launch the second application server

Repeat the previous lab with:

```text
Name: web-b
Subnet: private-app-b
```

Use:

```bash
#!/bin/bash

dnf install -y httpd php php-mysqli

systemctl enable --now httpd

cat > /var/www/html/index.php <<'EOF'
<?php

$server = "RDS-ENDPOINT";
$username = "admin";
$password = "YOUR-RDS-PASSWORD";

$conn = new mysqli($server, $username, $password);

echo "<h1>Secure AWS Application</h1>";
echo "<h2>Response from web-b</h2>";

if ($conn->connect_error) {
    echo "<p>Database connection: FAILED</p>";
} else {
    echo "<p>Database connection: SUCCESS</p>";
}

?>
EOF
```

Wait until both instances are **Running** and have passed their EC2 status checks.

Confirm that neither EC2 instance has a public IPv4 address.

---

# Lab 9 - Create the target group

Open:

**EC2 → Target Groups → Create target group**

Select:

```text
Target type: Instances
```

Enter:

```text
Name: secure-app-tg
Protocol: HTTP
Port: 80
VPC: secure-app-vpc
Health check path: /
```

Register:

```text
web-a
web-b
```

on port:

```text
80
```

Create the target group.

When EC2 instances are registered by instance ID, the ALB sends traffic to their private IP addresses.

---

# Lab 10 - Create the Application Load Balancer

Open:

**EC2 → Load Balancers → Create load balancer**

Select:

**Application Load Balancer**

Enter:

```text
Name: secure-app-alb
Scheme: Internet-facing
IP address type: IPv4
```

Select:

```text
VPC: secure-app-vpc
```

Select both public subnets:

```text
public-web-a
public-web-b
```

Select:

```text
Security group: alb-sg
```

Create listener:

```text
HTTP : 80
```

Forward to:

```text
secure-app-tg
```

Create the load balancer.

---

# Lab 11 - Validate the application

Open:

**EC2 → Target Groups → secure-app-tg**

Wait until both targets show:

```text
Healthy
```

Open:

**EC2 → Load Balancers → secure-app-alb**

Copy the ALB DNS name and open it in a browser.

You should see either:

```text
Secure AWS Application

Response from web-a

Database connection: SUCCESS
```

or:

```text
Secure AWS Application

Response from web-b

Database connection: SUCCESS
```

Refresh several times and confirm that requests can reach both servers.

---

# Lab 12 - Validate the security architecture

### EC2

Open each instance and confirm:

```text
Public IPv4 address: None
```

The EC2 instances should therefore not be directly accessible using an internet-facing public address.

### RDS

Open:

**RDS → secure-app-db**

Confirm:

```text
Publicly accessible: No
```

AWS recommends private RDS connectivity when public database access is unnecessary.

### Security groups

Confirm:

```text
alb-sg
80 ← Internet

app-sg
80 ← alb-sg

db-sg
3306 ← app-sg
```

---

# Lab 13 - Observe failure handling

1. Stop `web-a`.
2. Open **Target Groups → secure-app-tg**.
3. Observe its health status.
4. Continue accessing the ALB DNS name.
5. Confirm that the application continues responding from `web-b`.
6. Start `web-a`.
7. Wait until it becomes healthy again.

---


## Checkpoint

The lab is complete when:

* The **ALB is the only internet-facing application component**.
* `web-a` and `web-b` have no public IPv4 addresses.
* Both EC2 instances are healthy ALB targets.
* The application page is accessible through the ALB DNS name.
* The page shows **Database connection: SUCCESS**.
* RDS has **Public access = No**.
* `app-sg` accepts HTTP only from `alb-sg`.
* `db-sg` accepts MySQL only from `app-sg`.

## Cleanup

Delete resources in this order:

1. Application Load Balancer.
2. Target group.
3. Both EC2 instances.
4. RDS database.
5. RDS DB subnet group.
6. `db-sg`, `app-sg`, and `alb-sg`.
7. NAT Gateway.
8. Release the NAT Gateway Elastic IP.
9. Route tables.
10. Subnets.
11. Internet Gateway after detaching it.
12. VPC.
