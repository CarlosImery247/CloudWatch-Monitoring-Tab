# AWS Lab: Monitoring Infrastructure with CloudWatch, EventBridge and AWS Config

Hands-on lab completed on **16 Nov 2026** in **us-west-2 (Oregon)**. I used the CloudWatch agent, CloudWatch Logs, CloudWatch Metrics and alarms, EventBridge, SNS and AWS Config to monitor an EC2 web server.

> Screenshots have been redacted (account IDs, instance and resource IDs, IP addresses, email addresses and ARNs are blurred). The original lab guide is AWS training material and is not reproduced here; this repo contains my own notes and configuration.

## What I built

| # | Task | Services |
|---|------|----------|
| 1 | Install and configure the CloudWatch agent | Systems Manager Run Command, Parameter Store |
| 2 | Ship Apache logs, create a 404 metric filter and alarm | CloudWatch Logs, CloudWatch Alarms, SNS |
| 3 | Inspect OS-level metrics (disk, memory, swap) | CloudWatch Metrics (`CWAgent` namespace) |
| 4 | Real-time notification when an instance stops or terminates | EventBridge, SNS |
| 5 | Compliance checks for tags and unattached EBS volumes | AWS Config |

The agent configuration I stored in Parameter Store is in [`config/cloudwatch-agent-config.json`](config/cloudwatch-agent-config.json).

## Walkthrough

### Task 1: Install and configure the CloudWatch agent

Installed the agent with the `AWS-ConfigureAWSPackage` document, stored a JSON config in Parameter Store (`Monitor-Web-Server`), then started the agent with `AmazonCloudWatch-ManageAgent` (configure, ec2, ssm source, restart).

**Run Command output: AWS-ConfigureAWSPackage step 1 completes successfully**

![Run Command output: AWS-ConfigureAWSPackage step 1 completes successfully](images/01-ssm-install-agent-step1.png)

**Output confirming `AmazonCloudWatchAgent` was installed**

![Output confirming `AmazonCloudWatchAgent` was installed](images/02-ssm-install-agent-output.png)

**Creating the `Monitor-Web-Server` parameter in Parameter Store**

![Creating the `Monitor-Web-Server` parameter in Parameter Store](images/03-parameter-store-create.png)

**Selecting the `AmazonCloudWatch-ManageAgent` document**

![Selecting the `AmazonCloudWatch-ManageAgent` document](images/04-run-command-manage-agent.png)

**Command parameters: configure / ec2 / ssm / `Monitor-Web-Server` / restart**

![Command parameters: configure / ec2 / ssm / `Monitor-Web-Server` / restart](images/05-manage-agent-parameters.png)

**Targeting the Web Server instance manually**

![Targeting the Web Server instance manually](images/06-target-selection.png)

**Command status: Success**

![Command status: Success](images/07-command-status-success.png)


### Task 2: Application logs, metric filter and alarm

Generated 404s by requesting nonexistent pages, confirmed the access log in CloudWatch Logs, created a metric filter (`status_code=404`) that feeds `LogMetrics/404Errors`, and an alarm that fires at 5 or more 404s per minute.

**Web server test page reachable on its public IP**

![Web server test page reachable on its public IP](images/08-apache-test-page.png)

**Requesting `/start` returns 404 Not Found (generates log data)**

![Requesting `/start` returns 404 Not Found (generates log data)](images/09-not-found-page.png)

**Log stream in `HttpAccessLog` (named after the instance ID)**

![Log stream in `HttpAccessLog` (named after the instance ID)](images/10-log-streams.png)

**Access log events delivered to CloudWatch Logs**

![Access log events delivered to CloudWatch Logs](images/11-log-events.png)

**Expanded event showing the `GET /start` request with status 404**

![Expanded event showing the `GET /start` request with status 404](images/12-log-event-404.png)

**Filter pattern `[ip, id, user, timestamp, request, status_code=404, size]` and test**

![Filter pattern `[ip, id, user, timestamp, request, status_code=404, size]` and test](images/13-metric-filter-pattern-test.png)

**Test results: 2 of 5 sample events matched with status 404**

![Test results: 2 of 5 sample events matched with status 404](images/14-metric-filter-test-results.png)

**Assigning metric `LogMetrics / 404Errors` with value 1**

![Assigning metric `LogMetrics / 404Errors` with value 1](images/15-metric-filter-assign-metric.png)

**Review and create the metric filter**

![Review and create the metric filter](images/16-metric-filter-review.png)

**Metric filter created**

![Metric filter created](images/17-metric-filter-created.png)

**Alarm condition: `404Errors` >= 5 per 1 minute**

![Alarm condition: `404Errors` >= 5 per 1 minute](images/18-alarm-conditions.png)

**Alarm notification to an SNS topic**

![Alarm notification to an SNS topic](images/19-alarm-notification.png)

**Alarm name and description**

![Alarm name and description](images/20-alarm-details.png)

**Preview and create: metric and period**

![Preview and create: metric and period](images/21-alarm-preview.png)

**Review of conditions, actions and details**

![Review of conditions, actions and details](images/22-alarm-review-actions.png)

**Confirming the SNS email subscription**

![Confirming the SNS email subscription](images/23-sns-confirm-subscription.png)

**Alarm in the ALARM state on the CloudWatch overview**

![Alarm in the ALARM state on the CloudWatch overview](images/24-alarm-in-state.png)

**Alarm notification email**

![Alarm notification email](images/25-alarm-email.png)


### Task 3: Instance metrics

The agent publishes metrics that EC2 cannot see from outside the VM (memory, swap, disk used).

**Web Server instance in the EC2 console (Monitoring tab source)**

![Web Server instance in the EC2 console (Monitoring tab source)](images/26-ec2-instance-list.png)

**Custom namespaces `CWAgent` and `LogMetrics` in CloudWatch Metrics**

![Custom namespaces `CWAgent` and `LogMetrics` in CloudWatch Metrics](images/27-metrics-namespaces.png)

**Disk metrics collected by the agent (device, fstype, host, path)**

![Disk metrics collected by the agent (device, fstype, host, path)](images/28-cwagent-disk-metrics.png)

**Memory and swap metrics collected by the agent (by host)**

![Memory and swap metrics collected by the agent (by host)](images/29-cwagent-memory-metrics.png)


### Task 4: Real-time notifications

Created an EventBridge rule matching EC2 state changes to `stopped` or `terminated` and routed it to an SNS topic, then stopped the instance to test it.

**EventBridge default event bus**

![EventBridge default event bus](images/30-eventbridge-default-bus.png)

**Event pattern rules on the default bus, with Create rule**

![Event pattern rules on the default bus, with Create rule](images/31-eventbridge-create-rule.png)

**Advanced builder: building the event pattern**

![Advanced builder: building the event pattern](images/32-eventbridge-advanced-builder.png)

**EC2 Instance State-change Notification for `stopped` and `terminated`**

![EC2 Instance State-change Notification for `stopped` and `terminated`](images/33-eventbridge-event-pattern.png)

**Target: SNS topic `Default_CloudWatch_Alarms_Topic`**

![Target: SNS topic `Default_CloudWatch_Alarms_Topic`](images/34-eventbridge-select-target.png)

**Rule `Instance_Stopped_Terminated` created**

![Rule `Instance_Stopped_Terminated` created](images/35-eventbridge-rule-created.png)

**SNS topic with the confirmed email subscription**

![SNS topic with the confirmed email subscription](images/36-sns-topic-subscription.png)

**Stopping the Web Server instance to trigger the rule**

![Stopping the Web Server instance to trigger the rule](images/37-stop-instance.png)

**State-change notification email (JSON body)**

![State-change notification email (JSON body)](images/38-state-change-email.png)


### Task 5: AWS Config compliance

Enabled the `required-tags` rule (tag key `project`) and `ec2-volume-inuse-check`.

**AWS Config: selecting the `required-tags` managed rule**

![AWS Config: selecting the `required-tags` managed rule](images/39-config-required-tags-rule.png)

**AWS Config: selecting `ec2-volume-inuse-check`**

![AWS Config: selecting `ec2-volume-inuse-check`](images/40-config-volume-inuse-rule.png)

**Both Config rules active**

![Both Config rules active](images/41-config-rules-list.png)

**Non-compliant resources for `required-tags`**

![Non-compliant resources for `required-tags`](images/42-config-noncompliant-resources.png)

**Non-compliant volume for `ec2-volume-inuse-check`**

![Non-compliant volume for `ec2-volume-inuse-check`](images/43-config-noncompliant-volume.png)


## Troubleshooting and lessons learned

The lab guide is older than the current console, so a few things moved:

- **Log groups** are now under *CloudWatch > Logs > Log management*.
- **Metrics** are now under *CloudWatch > Metrics*, and the agent metrics appear in the `CWAgent` custom namespace.
- **CloudWatch Events** is now **Amazon EventBridge**. Go to *Event buses > default > Create rule*, then use the **Advanced builder** and enter the lab values.
- **AWS Config** may need first-time setup through the online guide before rules can be added.
- The agent output step may show a skipped Windows-only step; that is expected on a Linux AMI.
- The alarm briefly shows *Insufficient data* until the first metric datapoint arrives.

## Repo layout

```
.
├── README.md
├── config/cloudwatch-agent-config.json
└── images/            # redacted screenshots, numbered in lab order
```
