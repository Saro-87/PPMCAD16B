# Hands-On Labs - AWS Architecture

## Lab outcome

Build a small, resilient web tier across two Availability Zones and access it through an Application Load Balancer.

The lab uses public subnets for the EC2 instances so that the web server packages can be installed without a NAT Gateway. The instance security group still accepts web traffic only from the load balancer. In production, application instances would normally use private subnets.

## Prerequisites

- Access to the AWS Management Console.
- Permission to manage VPC, EC2, security groups, target groups and load balancers.
- Select one AWS Region and use it throughout the lab.
- Use small instances such as `t3.micro` where available.

## Lab 1 - Create the VPC

1. Open **VPC → Your VPCs → Create VPC**.
2. Select **VPC only**.
3. Enter:
   - Name: `architecture-lab-vpc`
   - IPv4 CIDR: `10.20.0.0/16`
4. Create the VPC.
5. Open **Subnets → Create subnet**.
6. Create two subnets in different Availability Zones:

| Name | Availability Zone | CIDR |
|---|---|---|
| `public-web-a` | First available AZ | `10.20.1.0/24` |
| `public-web-b` | Second available AZ | `10.20.2.0/24` |

7. Select both subnets and enable **Auto-assign public IPv4 address**.

## Lab 2 - Add internet routing

1. Open **Internet gateways → Create internet gateway**.
2. Name it `architecture-lab-igw`.
3. Create it and attach it to `architecture-lab-vpc`.
4. Open **Route tables → Create route table**.
5. Name it `public-web-rt` and select the lab VPC.
6. Open the route table and add:

```text
Destination: 0.0.0.0/0
Target: architecture-lab-igw
```

7. Associate `public-web-a` and `public-web-b` with this route table.

### Validate

Confirm that both subnets use `public-web-rt` and that the route table contains the Internet Gateway route.

## Lab 3 - Create tier-based security groups

### Load balancer security group

1. Open **EC2 → Security Groups → Create security group**.
2. Name it `alb-sg` and select the lab VPC.
3. Add inbound rule:

```text
HTTP | TCP | 80 | Anywhere-IPv4
```

### Web server security group

1. Create another security group named `web-sg`.
2. Add inbound rule:

```text
HTTP | TCP | 80 | Source: alb-sg
```

Do not add SSH or HTTP access from `0.0.0.0/0` to `web-sg`.

## Lab 4 - Launch two web servers

1. Open **EC2 → Instances → Launch instances**.
2. Launch the first instance with:
   - Name: `web-a`
   - AMI: Amazon Linux
   - Instance type: `t3.micro` or another small allowed type
   - Subnet: `public-web-a`
   - Security group: `web-sg`
   - Key pair: **Proceed without a key pair**
3. Under **Advanced details → User data**, enter:

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo '<h1>Response from web-a</h1>' > /var/www/html/index.html
```

4. Repeat for `web-b` in `public-web-b` and change the page text to:

```html
<h1>Response from web-b</h1>
```

5. Wait until both instances show **Running** and have passed status checks.

## Lab 5 - Create the target group

1. Open **EC2 → Target Groups → Create target group**.
2. Select **Instances**.
3. Enter:
   - Name: `web-tg`
   - Protocol: HTTP
   - Port: 80
   - VPC: `architecture-lab-vpc`
   - Health check path: `/`
4. Register `web-a` and `web-b` on port 80.
5. Create the target group.

## Lab 6 - Create the Application Load Balancer

1. Open **EC2 → Load Balancers → Create load balancer**.
2. Select **Application Load Balancer**.
3. Enter:
   - Name: `architecture-lab-alb`
   - Scheme: Internet-facing
   - IP address type: IPv4
4. Select the lab VPC and both public subnets.
5. Select `alb-sg`.
6. Configure an HTTP listener on port 80 forwarding to `web-tg`.
7. Create the load balancer.

### Validate

1. Wait until both targets show **Healthy**.
2. Copy the load balancer DNS name and open it in a browser.
3. Refresh several times and confirm that responses can come from both servers.
4. Try opening an instance public IP directly. HTTP should fail because `web-sg` allows port 80 only from `alb-sg`.

## Lab 7 - Observe failure handling

1. Stop either `web-a` or `web-b`.
2. Observe the target health in `web-tg`.
3. Continue refreshing the load balancer URL.
4. Confirm that the load balancer sends requests only to the remaining healthy target.
5. Start the stopped instance and wait for it to become healthy again.

## Architecture extension

Identify where these production components would be placed without creating them:

- EC2 Auto Scaling group spanning both AZs.
- Private application subnets.
- Private database subnets and an RDS Multi-AZ deployment.
- S3 for static files.
- IAM role for application access to AWS services.
- CloudWatch metrics, logs and alarms.

## Cleanup

Delete resources in this order:

1. Application Load Balancer.
2. Target group.
3. Both EC2 instances.
4. `web-sg` and `alb-sg` after dependencies disappear.
5. Internet Gateway after detaching it.
6. Route table and subnets.
7. Lab VPC.
