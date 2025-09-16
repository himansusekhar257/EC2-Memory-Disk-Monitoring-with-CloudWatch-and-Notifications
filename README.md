# 📘 EC2 Memory & Disk Monitoring with CloudWatch and Notifications

This guide walks you through setting up **Amazon CloudWatch Agent** on an EC2 instance to monitor **memory and disk utilization**, and configuring **CloudWatch Alarms** with **SNS notifications** when thresholds are crossed.

---

## 🚀 Steps to Configure Monitoring

### **Step 1: Create IAM Role for EC2**

1. Go to **IAM → Roles → Create role**.
2. Choose **EC2** as the trusted entity.
3. Attach the following policies:
   - `CloudWatchAgentServerPolicy`
   - `AmazonSSMManagedInstanceCore`
   - *(For demo purposes, you can use `CloudWatchFullAccess` + `AmazonSSMFullAccess`)*
4. Name the role: **EC2-CloudWatch-Role**.
5. Attach this role to your EC2 instance.

---

### **Step 2: Create SSM Parameter for CloudWatch Agent Config**

1. Go to **Systems Manager → Parameter Store → Create parameter**.
2. Set:
    - **Name:** `/alarm/AWS-CWAgentLinConfig`
    - **Type:** `String`
    - **Value:** (Paste the following JSON)

    ```json
    {
      "metrics": {
        "append_dimensions": {
          "InstanceId": "${aws:InstanceId}"
        },
        "metrics_collected": {
          "mem": {
            "measurement": [
              "mem_used_percent"
            ],
            "metrics_collection_interval": 60
          },
          "disk": {
            "measurement": [
              "disk_used_percent"
            ],
            "metrics_collection_interval": 60
          }
        }
      }
    }
    ```

---

### **Step 3: Install and Configure CloudWatch Agent**

Add the following to **EC2 User Data** (when launching the instance):

```bash
#!/bin/bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/linux/amd64/latest/AmazonCloudWatchAgent.zip
unzip AmazonCloudWatchAgent.zip
sudo ./install.sh
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:/alarm/AWS-CWAgentLinConfig -s
```

**Check if the agent is running:**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

> At this stage, metrics are pushed to CloudWatch, but no alerts/notifications are configured yet.

---

## 🔔 Steps to Configure Notifications

### **Step 4: Create SNS Topic and Subscription**

1. Go to **Amazon SNS → Topics → Create topic**.
2. **Type:** Standard
3. **Name:** `EC2-Memory-Disk-Alerts`
4. Create a **subscription**:
    - **Protocol:** Email
    - **Endpoint:** Your email address
5. **Confirm** the subscription via email (status must be *Confirmed*).

---

### **Step 5: Create CloudWatch Alarms**

1. Go to **CloudWatch → Alarms → Create alarm**.
2. **Select metric:**
    - **Namespace:** `CWAgent`
    - **Metric name:** 
        - `mem_used_percent` (for memory)
        - `disk_used_percent` (for disk)
3. **Configure alarm:**
    - Example: Trigger when `mem_used_percent` >= 80 for 1 datapoint within 5 minutes.
    - Example: Trigger when `disk_used_percent` >= 85 for 1 datapoint within 5 minutes.
4. Under **Notification settings**, choose SNS topic: `EC2-Memory-Disk-Alerts`.
5. **Create alarm**.

---

## 🧪 Step 6: Testing Alarms

- **Test SNS manually:**
    ```bash
    aws sns publish --topic-arn arn:aws:sns:REGION:ACCOUNT_ID:EC2-Memory-Disk-Alerts \
      --message "Test notification from CloudWatch SNS"
    ```

- **Test memory alarm:**
    ```bash
    sudo yum install stress -y   # Amazon Linux
    stress --vm 1 --vm-bytes 200M --timeout 120s
    ```

- **Test disk alarm:**
    ```bash
    dd if=/dev/zero of=/tmp/bigfile bs=1M count=10000
    rm /tmp/bigfile
    ```

> When thresholds are crossed, alarms move to **ALARM** state and SNS sends an email notification.

---

## ✅ Summary

- **CloudWatch Agent** collects memory & disk metrics.
- **CloudWatch Alarms** monitor metrics against thresholds.
- **SNS** notifies via email/SMS/Lambda when alarms are triggered.

---

**References:**
- [Amazon CloudWatch Agent Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [AWS SNS Documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
