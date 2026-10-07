# AWS_Web_Migration
An AWS migration proof of concept featuring WordPress on EC2, a LAMP stack, IAM-based log collection, and CloudWatch monitoring.
# AWS Web Migration

Deploying a WordPress application on Amazon EC2 with centralized Apache logging and CPU monitoring through Amazon CloudWatch.

## Project Overview

This project implements the target AWS environment for a web migration proof of concept. It combines a working LAMP application with IAM-based log collection and a tested CloudWatch CPU alarm.

The implementation covers server provisioning, Linux administration, application and database configuration, and monitoring. Transferring an existing on-premises application is outside the completed scope.

## Technology Stack

| Technology | Purpose |
|---|---|
| Amazon EC2 | Hosts the application on Ubuntu Linux |
| Apache | Serves web requests |
| PHP | Runs WordPress application code |
| MySQL | Stores WordPress content |
| WordPress | Provides the web application |
| AWS IAM | Grants the EC2 instance permissions to publish monitoring data |
| CloudWatch Agent | Collects Apache error logs |
| CloudWatch Logs | Stores collected logs with seven-day retention |
| CloudWatch Alarms | Evaluates EC2 CPU utilization |
| Amazon SNS | Configured as the alarm notification destination |

## Architecture

Apache, PHP, WordPress, and MySQL run on a single EC2 instance.

- Visitors access WordPress through Apache.
- WordPress reads and writes content in the local MySQL database.
- The CloudWatch agent forwards Apache error logs using the instance's IAM role.
- A CloudWatch alarm evaluates the EC2 CPUUtilization metric.
- Amazon SNS is configured for email notifications.

## Implementation

### Application Deployment

- Provisioned an Ubuntu EC2 instance.
- Connected to the server using SSH key authentication.
- Installed Apache, PHP, and MySQL.
- Created a dedicated WordPress database and database user.
- Configured WordPress database connectivity and file permissions.
- Enabled Apache and MySQL to start automatically at boot.
- Accessed the WordPress administration dashboard.

### Centralized Logging

- Attached an EC2 IAM role with CloudWatchAgentServerPolicy.
- Installed and configured the CloudWatch agent.
- Collected logs from `/var/log/apache2/error.log`.
- Published logs to `/wordpress/apache/error`.
- Used the EC2 instance ID as the log stream name.
- Configured seven-day log retention.
- Verified Apache log entries appeared in CloudWatch.

### CPU Alarm Testing

| Setting | Test configuration |
|---|---|
| Namespace | AWS/EC2 |
| Metric | CPUUtilization |
| Statistic | Average |
| Period | 5 minutes |
| Threshold | Greater than 2% |
| Datapoints to alarm | 1 out of 1 |

A temporary 2% threshold was used to validate alarm triggering under light activity. The alarm successfully entered the ALARM state.

The intended operating threshold is 80%; the 2% setting is for testing only.

## Validation Results

| Check | Result |
|---|---|
| WordPress dashboard accessible | Verified |
| CloudWatch agent running | Verified |
| Apache logs visible in CloudWatch | Verified |
| CPU alarm entered ALARM state | Verified |
| SNS email delivery | Not yet verified |

## Engineering Decisions

- Kept the application and database on one instance to limit complexity and cost.
- Used an IAM instance role instead of storing AWS access keys on the server.
- Limited log retention to seven days.
- Tested alarm behavior using a temporary low threshold.

## Current Limitations

- The single EC2 instance is a single point of failure.
- The deployment uses HTTP; HTTPS has not been configured.
- Backup restoration and high availability have not been implemented.
- Collected logs exposed PHP JIT memory-allocation warnings that remain to be investigated.
- Alarm triggering is verified; email delivery requires further validation.

## Skills Demonstrated

AWS EC2 provisioning, Ubuntu administration, SSH access, LAMP configuration, MySQL user management, IAM roles, centralized logging, and CloudWatch alarm testing.
