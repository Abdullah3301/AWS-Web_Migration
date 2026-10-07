# AWS Web Migration

A web migration combining **AWS infrastructure, Linux administration, application deployment, and operational monitoring**.

Deployed WordPress on **Amazon EC2 running Ubuntu**, configured the **Apache, PHP, and MySQL** stack, and administered the server through **SSH key-based access**. The implementation includes database-scoped application permissions, Linux file and service management, and an **EC2 IAM role** for publishing Apache error logs to **Amazon CloudWatch**. Monitoring was validated through collected log events and a CPU alarm triggered using a temporary test threshold.

The completed scope covers the target AWS environment and monitoring setup. Transfer of an existing on-premises application and dataset has not yet been demonstrated. 

<img width="1197" height="776" alt="Screenshot 2026-10-06 205322" src="https://github.com/user-attachments/assets/95da22e1-d09f-4852-ad02-8f54b60d23df" />

## Architecture

Apache, PHP, WordPress, and MySQL run on a single EC2 instance. Application logs are forwarded to CloudWatch using the instance’s IAM role, while a separate alarm evaluates the standard EC2 CPU utilization metric.

| Path | Flow |
|---|---|
| Web requests | Browser → Apache → WordPress/PHP → MySQL |
| Administration | Administrator → SSH → Ubuntu |
| Application logs | Apache error log → CloudWatch agent → CloudWatch Logs |
| Monitoring | EC2 CPU metric → CloudWatch alarm → SNS notification topic |

The CloudWatch agent collects application logs. The CPU alarm uses the metric published by EC2 and does not depend on the agent.

## Technology Stack

| Technology | Purpose |
|---|---|
| **Amazon EC2** | Hosts the application and local database |
| **Ubuntu Linux** | Server operating system |
| **SSH and key pairs** | Encrypted remote terminal access |
| **EC2 Security Groups** | Instance-level network access control |
| **Apache** | HTTP web server |
| **PHP** | WordPress application runtime |
| **MySQL and SQL** | Application data storage, database creation, and user permissions |
| **WordPress** | Web application |
| **APT** | Package installation and updates |
| **systemd / systemctl** | Service startup, boot configuration, and status checks |
| **AWS IAM** | Role-based permissions for the CloudWatch agent |
| **CloudWatch Agent** | Collects Apache error logs |
| **CloudWatch Logs** | Centralized log storage with seven-day retention |
| **CloudWatch Alarms** | Threshold-based CPU monitoring |
| **Amazon SNS** | Configured notification destination; email delivery remains unverified |

## Implementation

### 1. Infrastructure and Server Administration

- Provisioned an Ubuntu EC2 instance.
- Connected through SSH using key-based authentication.
- Updated system packages using APT.
- Used `sudo` for Linux administrative operations.

### 2. Web Stack Configuration

- Installed Apache, PHP, the PHP–MySQL integration package, and MySQL.
- Started Apache and MySQL using `systemctl`.
- Enabled both services to start automatically at boot.
- Checked service status before configuring the application.
- Ran the MySQL security configuration utility.

### 3. Application and Database Setup

- Downloaded and extracted WordPress.
- Placed application files in Apache’s document root.
- Configured file ownership and permissions.
- Created a dedicated WordPress database.
- Created a local MySQL application user with privileges scoped to that database.
- Configured database connectivity in `wp-config.php`.
- Completed browser-based setup and accessed the WordPress dashboard.

### 4. IAM Integration

- Created an IAM role trusted by the EC2 service.
- Attached the AWS-managed `CloudWatchAgentServerPolicy`.
- Associated the role with the EC2 instance.
- Used role-provided temporary credentials for the agent’s AWS access.

The instance role authorizes requests to AWS services. Linux `sudo` permissions separately control administrative operations inside Ubuntu.

### 5. Centralized Logging

- Installed the Amazon CloudWatch agent on Ubuntu.
- Configured collection of `/var/log/apache2/error.log`.
- Published events to the `/wordpress/apache/error` log group.
- Used the EC2 instance ID as the log stream name.
- Configured seven-day retention.
- Verified the agent’s running status and confirmed log delivery in the CloudWatch console.

### 6. CPU Alarm Validation

Configured an alarm against the instance’s `CPUUtilization` metric.

| Setting | Test Configuration |
|---|---|
| Namespace | `AWS/EC2` |
| Metric | `CPUUtilization` |
| Statistic | Average |
| Period | 5 minutes |
| Comparison | Greater than |
| Temporary test threshold | 2% |
| Datapoints to alarm | 1 out of 1 |
| Notification destination | Amazon SNS topic |

The temporary 2% threshold allowed alarm behavior to be tested under light activity. CloudWatch successfully transitioned the alarm into the **ALARM** state when the five-minute CPU average exceeded the threshold.

The intended operating threshold is **80%**. The screenshot below records the **2% test configuration**, not a high-load performance test.

## CloudWatch Agent Configuration

The following configuration forwards Apache error logs to CloudWatch and applies seven-day retention:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/apache2/error.log",
            "log_group_name": "/wordpress/apache/error",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          }
        ]
      }
    }
  }
}
```

Configuration file location:

```text
/opt/aws/amazon-cloudwatch-agent/etc/wordpress-logs.json
```

Load the configuration and start the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/wordpress-logs.json \
  -s
```

Check agent status:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

## Deployment and Monitoring

### WordPress Application

WordPress administration dashboard after deployment.

<img width="1887" height="895" alt="Screenshot 2026-10-06 124717" src="https://github.com/user-attachments/assets/1bac8b25-b51e-40f9-84f2-ad6e9c409641" />


### Centralized Application Logs

Apache error logs collected from EC2 and displayed in CloudWatch.

<img width="2053" height="766" alt="Privacy-redacted CloudWatch log screenshot" src="https://github.com/user-attachments/assets/d367c625-0a2e-43e8-a5e4-979d07cda021" />

### CPU Alarm Validation

CloudWatch alarm in the **ALARM** state during the temporary 2% threshold test.

<img width="1895" height="607" alt="Screenshot 2026-10-06 195316" src="https://github.com/user-attachments/assets/8d852ae1-2395-4930-84f9-65bcb57bdffb" />


## Validation Results

| Check | Result |
|---|---|
| WordPress administration dashboard accessible | Verified |
| CloudWatch agent running and configured | Verified |
| Apache log events arriving in CloudWatch | Verified |
| CPU alarm entering ALARM state | Verified using the temporary 2% threshold |
| SNS email notification delivery | Not yet verified |

## Engineering Decisions

**Focused deployment:** Application and database components share one EC2 instance to keep the proof of concept manageable and limit infrastructure cost. This introduces a single point of failure.

**Role-based AWS access:** The CloudWatch agent uses an EC2 IAM role instead of stored AWS access keys. The implementation uses an AWS-managed policy; permissions could be narrowed further through a custom policy scoped to the required log resources.

**Database-scoped permissions:** WordPress connects through a dedicated MySQL user with access to its application database.

**Bounded log retention:** Seven-day retention limits how long logs are stored. Log ingestion volume still affects cost.

**Explicit monitoring validation:** A temporary low threshold verified the alarm’s state transition without requiring a sustained high-CPU workload.

## Troubleshooting Observations

Centralized logging exposed recurring PHP warnings about JIT memory allocation. These messages confirmed that application-level events were reaching CloudWatch.

The cause and remediation of these warnings remain to be investigated. Alarm triggering was also verified independently of email delivery, which remains an outstanding validation item.

## Scope and Limitations

- Application and database components run on a single EC2 instance.
- The deployment uses HTTP; HTTPS has not been configured.
- High availability and automated scaling are outside the implemented scope.
- Backup restoration has not been tested.
- Existing on-premises application and data transfer have not been demonstrated.
- SNS email delivery remains unverified.
- PHP JIT memory-allocation warnings require further investigation.
