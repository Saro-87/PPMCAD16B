# Amazon CloudWatch - Hands-On Labs

## Session goal

Build and observe two monitoring paths:

`EC2 metrics -> CPU alarm -> SNS -> email`

`Application log -> CloudWatch Logs -> Logs Insights -> metric filter -> alarm -> dashboard`

**Environment:** One Amazon Linux 2023 EC2 instance in a training AWS account  
**Access method:** AWS Management Console and browser-based Session Manager only  
**Region:** Use the same AWS Region for every resource

> All AWS resources in this guide are created and managed through the AWS Management Console. Commands are used only inside the EC2 Session Manager terminal to configure the operating system and generate test activity.

---

## Before the session - Prepare the EC2 instance

### Step 1: Create an EC2 instance role

1. Open **IAM -> Roles -> Create role**.
2. For **Trusted entity type**, select **AWS service**.
3. For **Use case**, select **EC2**.
4. Attach these policies:
   - `CloudWatchAgentServerPolicy` - allows the agent to publish metrics and logs.
   - `AmazonSSMManagedInstanceCore` - allows browser-based Session Manager access.
5. Name the role `CloudWatchLabEC2Role` and create it.

> `CloudWatchAgentServerPolicy` already includes the permissions required to publish custom metrics and create and write to CloudWatch log groups and streams. No separate logs policy is required for this lab.

### Step 2: Launch the instance

1. Open **EC2 -> Instances -> Launch instances**.
2. Name the instance `cloudwatch-lab-ec2`.
3. Select **Amazon Linux 2023**.
4. Select `t2.micro` or `t3.micro`, depending on availability.
5. Under **Advanced details -> IAM instance profile**, select `CloudWatchLabEC2Role`.
6. Under **Advanced details -> CloudWatch monitoring**, enable **Detailed CloudWatch monitoring** so `CPUUtilization` is published at one-minute intervals.
7. Launch the instance.
8. Record its **Instance ID** and confirm that its state is **Running**.

> Detailed monitoring can incur CloudWatch charges. Disable or terminate the training instance during cleanup.

### Step 3: Connect through the console

1. Select the instance.
2. Choose **Connect -> Session Manager -> Connect**.
3. Keep this browser terminal open for the labs.

---

## Lab 1 - Explore EC2 default metrics

### Objective

Read a built-in EC2 metric and connect the CloudWatch metric model to a real resource.

### Step 1: Open the metric

1. Open **CloudWatch** in another browser tab.
2. Select **Metrics -> All metrics**.
3. Select **EC2 -> Per-Instance Metrics**.
4. Search using your instance ID.
5. Select `CPUUtilization`.

Identify:

- Namespace: `AWS/EC2`
- Metric name: `CPUUtilization`
- Dimension: `InstanceId=<your-instance-id>`
- Statistic: `Average`
- Period: start with `5 minutes`

### Step 2: Compare statistics and periods

In **Graphed metrics**:

1. Change **Statistic** between `Average` and `Maximum`.
2. Change **Period** between `1 minute` and `5 minutes`.
3. Change the graph time range to the last hour.

Discuss:

- Why can `Maximum` reveal a spike hidden by `Average`?
- Why might a longer period make a graph smoother but detection slower?
- Why is memory usage not available as a default EC2 metric?

### Success criteria

- You can identify the namespace, metric name, dimension, statistic, and period.
- You can explain the difference between a default EC2 metric and an agent metric.

---

## Lab 2 - Create an EC2 CPU alert with SNS email

### Objective

Create an alarm for high EC2 CPU utilization and send its state-change notification to an email address.

### Step 1: Start creating the alarm

1. Open **CloudWatch -> Alarms -> All alarms**.
2. Choose **Create alarm -> Select metric**.
3. Select **EC2 -> Per-Instance Metrics**.
4. Search using your instance ID.
5. Select `CPUUtilization`, then choose **Select metric**.

### Step 2: Configure the threshold

Use these training values:

- **Statistic:** `Average`
- **Period:** `1 minute`
- **Threshold type:** `Static`
- **Whenever CPUUtilization is:** `Greater/Equal`
- **Threshold value:** `50`
- **Datapoints to alarm:** `1 out of 1`
- **Missing data treatment:** `Treat missing data as missing`

Choose **Next**.

> A low threshold and one evaluation period make the transition easy to demonstrate. Production alarms normally use thresholds and evaluation periods based on the workload baseline.

### Step 3: Create the SNS topic and email subscription

Under **Notification**:

1. For **Alarm state trigger**, select **In alarm**.
2. Select **Create new topic**.
3. Enter the topic name `cloudwatch-lab-alerts`.
4. Enter an email address that you can access during the session.
5. Choose **Create topic**.
6. Open the AWS notification email and select **Confirm subscription**.

> The alarm can be created before confirmation, but SNS will not deliver notifications until the email subscription is confirmed.

### Step 4: Name and create the alarm

1. Choose **Next**.
2. Set **Alarm name** to `cloudwatch-lab-high-cpu`.
3. Add the description `EC2 CPU is at or above 50 percent`.
4. Choose **Next -> Create alarm**.

### Step 5: Generate CPU load from Session Manager

In the browser-based Session Manager terminal, run:

```bash
for i in 1 2; do timeout 300 bash -c 'while true; do :; done' & done
```

Return to **CloudWatch -> Alarms -> All alarms** and wait for the alarm to change from `OK` to `In alarm`. This can take several minutes because metric delivery and alarm evaluation are not instantaneous.

Inspect:

- The alarm graph and threshold line
- **State details**
- The **History** tab
- The SNS alert email

To stop the CPU test early, return to Session Manager and run:

```bash
pkill -f "while true"
```

### Success criteria

- The alarm changes to `In alarm` after CPU crosses the threshold.
- The confirmed email subscription receives an SNS notification.
- You can explain threshold, period, evaluation periods, and datapoints to alarm.

---

## Lab 3 - Install and configure the CloudWatch Agent

### Objective

Publish guest operating-system memory and disk metrics and collect application log file.

### Step 1: Install the agent

In the **Session Manager** browser terminal, run:

```bash
sudo dnf install -y amazon-cloudwatch-agent

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

### Step 2: Create a sample application log

```bash
sudo mkdir -p /var/log/cloudwatch-lab
sudo touch /var/log/cloudwatch-lab/app.log
sudo chmod 644 /var/log/cloudwatch-lab/app.log
```

### Step 3: Create the agent configuration

Copy this complete block into Session Manager:

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWWorkshop",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"]
      },
      "disk": {
        "measurement": ["used_percent"],
        "resources": ["/"]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/cloudwatch-lab/app.log",
            "log_group_name": "/training/cloudwatch/app",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
EOF
```

### Step 4: Start the agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

If the agent does not start, inspect its log:

```bash
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Step 5: Generate structured log events

```bash
for i in $(seq 1 20); do
  if (( i % 5 == 0 )); then
    echo "{\"level\":\"ERROR\",\"service\":\"checkout\",\"requestId\":\"req-$i\",\"latencyMs\":$((200 + i * 10))}" | sudo tee -a /var/log/cloudwatch-lab/app.log > /dev/null
  else
    echo "{\"level\":\"INFO\",\"service\":\"checkout\",\"requestId\":\"req-$i\",\"latencyMs\":$((40 + i))}" | sudo tee -a /var/log/cloudwatch-lab/app.log > /dev/null
  fi
done

tail /var/log/cloudwatch-lab/app.log
```

### Step 6: Verify metrics in the console

Wait two or three minutes, then:

1. Open **CloudWatch -> Metrics -> All metrics**.
2. Under **Custom namespaces**, select **CWWorkshop**.
3. Locate `mem_used_percent` and `disk_used_percent` for your instance.
4. Select the metrics and view them on the graph.

### Step 7: Verify logs and configure retention

1. Open **CloudWatch -> Logs -> Log groups**.
2. Open `/training/cloudwatch/app`.
3. Confirm that a log stream named with your instance ID exists.
4. Open the stream and locate the JSON events.
5. Return to the log-group details page.
6. Next to **Retention setting**, choose **Edit**.
7. Select **1 week (7 days)** and save.

### Success criteria

- The agent status is `running`.
- `mem_used_percent` and `disk_used_percent` appear in `CWWorkshop`.
- The application log group contains JSON events and has seven-day retention.

---

## Lab 4 - Investigate logs with Logs Insights

### Objective

Use focused queries to find application errors and summarize them over time.

### Step 1: Open Logs Insights

1. Open **CloudWatch -> Logs Insights**.
2. Select `/training/cloudwatch/app`.
3. Set the time range to the last 30 minutes.

### Step 2: Find recent errors

Enter the query and choose **Run query**:

```text
fields @timestamp, level, service, requestId, latencyMs
| filter level = "ERROR"
| sort @timestamp desc
| limit 20
```

### Step 3: Aggregate errors and latency

Replace the query and choose **Run query**:

```text
filter level = "ERROR"
| stats count(*) as errors, avg(latencyMs) as avgLatency by bin(5m)
| sort bin(5m) desc
```

### Success criteria

- The first query returns the generated `ERROR` events.
- The second query produces a time-binned summary.
- You can explain why a narrow time range reduces scanned data.

---

## Lab 5 - Convert application errors into a metric and alarm

### Objective

Use the AWS Console to count error logs continuously and create an alarm from that count.

### Step 1: Create the metric filter

1. Open **CloudWatch -> Logs -> Log groups**.
2. Select `/training/cloudwatch/app`.
3. Choose **Actions -> Create metric filter**.
4. Enter this **Filter pattern**:

```text
{ $.level = "ERROR" }
```

5. Choose **Test pattern** and confirm that the existing error events match.
6. Choose **Next**.
7. Set:
   - **Filter name:** `ErrorCount`
   - **Metric namespace:** `CWWorkshop`
   - **Metric name:** `ErrorCount`
   - **Metric value:** `1`
   - **Default value:** `0`
8. Choose **Next -> Create metric filter**.

### Step 2: Generate fresh matching events

Metric filters do not process old events. In Session Manager, generate new errors:

```bash
for i in 1 2 3; do
  echo "{\"level\":\"ERROR\",\"service\":\"checkout\",\"requestId\":\"alarm-$i\",\"latencyMs\":900}" | sudo tee -a /var/log/cloudwatch-lab/app.log > /dev/null
done
```

Wait one or two minutes.

### Step 3: Create the log-error alarm in the console

1. Open **CloudWatch -> Metrics -> All metrics -> Custom namespaces -> CWWorkshop**.
2. Locate and select `ErrorCount`.
3. Choose **Graphed metrics -> Create alarm**.
4. Configure:
   - **Statistic:** `Sum`
   - **Period:** `1 minute`
   - **Threshold:** `Greater/Equal` to `1`
   - **Datapoints to alarm:** `1 out of 1`
   - **Missing data treatment:** `Treat missing data as not breaching`
5. Choose **Next**.
6. For **Notification**, select the existing SNS topic `cloudwatch-lab-alerts`.
7. Choose **Next**.
8. Name the alarm `cloudwatch-lab-errors`.
9. Choose **Next -> Create alarm**.

Generate one more fresh error from Session Manager if the alarm needs to be triggered:

```bash
echo '{"level":"ERROR","service":"checkout","requestId":"final-trigger","latencyMs":1000}' | sudo tee -a /var/log/cloudwatch-lab/app.log
```

Open the alarm in the console and inspect its graph, state, and **History** tab.

### Success criteria

- The metric filter publishes `CWWorkshop/ErrorCount`.
- The alarm changes state after a fresh matching event arrives.
- The SNS email subscription receives the alarm notification.

---

## Lab 6 - Build the operations dashboard

### Objective

Create one view that answers: Is the instance healthy, and are application errors increasing?

1. Open **CloudWatch -> Dashboards -> Create dashboard**.
2. Name it `cloudwatch-lab-dashboard`.
3. Add these widgets:
   - Line: `AWS/EC2 -> CPUUtilization`
   - Line: `CWWorkshop -> mem_used_percent`
   - Line: `CWWorkshop -> disk_used_percent`
   - Number or line: `CWWorkshop -> ErrorCount`
   - Alarm status: `cloudwatch-lab-high-cpu` and `cloudwatch-lab-errors`
4. Set the time range to the last hour.
5. Save the dashboard.

### Success criteria

- The dashboard contains default and agent metrics.
- CPU state, error count, and alarm state are visible in one view.
- Every widget supports a clear operational question.

---

## Cleanup - AWS Console only

Perform cleanup only when your trainer confirms that the resources are no longer required.

1. **CloudWatch -> Alarms -> All alarms**
   - Select `cloudwatch-lab-high-cpu` and `cloudwatch-lab-errors`.
   - Choose **Actions -> Delete**.
2. **CloudWatch -> Dashboards**
   - Select `cloudwatch-lab-dashboard` and choose **Delete**.
3. **CloudWatch -> Logs -> Log groups -> /training/cloudwatch/app**
   - Open **Metric filters**, select `ErrorCount`, and delete it.
   - Return to **Log groups**, select `/training/cloudwatch/app`, and choose **Actions -> Delete log group**.
4. **Amazon SNS -> Topics**
   - Select `cloudwatch-lab-alerts` and choose **Delete**.
5. In the **Session Manager** terminal, stop the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop
```

6. **EC2 -> Instances**
   - Terminate `cloudwatch-lab-ec2` only if it was created specifically for this lab.
7. **IAM -> Roles**
   - Delete `CloudWatchLabEC2Role` only if it is not shared with another instance or lab.

---

## Final check

You should now be able to explain and demonstrate:

- The CloudWatch metric model
- Default EC2 metrics versus CloudWatch Agent metrics
- A CPU alarm with an SNS email notification
- Log groups, streams, and retention
- A Logs Insights investigation
- A log metric filter and alarm
- A compact operations dashboard
