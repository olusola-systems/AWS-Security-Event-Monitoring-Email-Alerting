# Week 34: AWS Security Event Monitoring & Email Alerting

## Project Overview

This project implements an AWS security monitoring and email alerting workflow using AWS CloudTrail, Amazon EventBridge, and Amazon Simple Notification Service (SNS).

The solution monitors selected AWS Identity and Access Management (IAM) changes, including user and group creation or deletion, access key changes, and user permission modifications. When a matching event occurs, EventBridge routes the event to an SNS topic, which sends an email notification.

The project was built through the AWS Management Console and tested with real IAM management events.

---

## Project Objectives

- Capture AWS management activity using CloudTrail
- Detect selected IAM security changes using EventBridge event patterns
- Route matching events to an SNS topic
- Deliver security notifications by email
- Validate the complete workflow using real AWS API events
- Investigate and troubleshoot event delivery failures

---

## Architecture

<img width="1440" height="857" alt="Screenshot 2026-10-25 at 4 13 15 PM" src="https://github.com/user-attachments/assets/30e6f7ff-9e4c-4cc2-b0f4-9883cb0caa90" />

---

## AWS Services Used

| Service | Purpose |
|---|---|
| AWS IAM | Source of monitored identity and permission changes |
| AWS CloudTrail | Records AWS management API activity |
| Amazon EventBridge | Matches events against the configured event pattern |
| Amazon SNS | Publishes matching events to an email subscriber |
| Amazon S3 | Stores CloudTrail log files |
| Amazon CloudWatch | Provides EventBridge rule monitoring metrics |

---

## Configuration

**AWS Region:** `us-east-1`

**CloudTrail trail:** `week34-security-monitoring-trail`

**EventBridge rule:** `week34-iam-security-monitoring`

**Event bus:** `default`

**SNS topic:** `week34-aws-security-alerts`

The CloudTrail trail was configured to log management events, with both read and write management activity selected. Data events, Insights events, and network activity events were not enabled for this project.

The EventBridge rule was configured with the following event pattern:

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com"],
    "eventName": [
      "CreateUser",
      "DeleteUser",
      "CreateGroup",
      "DeleteGroup",
      "AttachUserPolicy",
      "DetachUserPolicy",
      "PutUserPolicy",
      "DeleteUserPolicy",
      "CreateAccessKey",
      "DeleteAccessKey",
      "CreateLoginProfile",
      "UpdateLoginProfile",
      "AddUserToGroup",
      "RemoveUserFromGroup"
    ]
  }
}
```

This pattern focuses on selected IAM identity, credential, group membership, and permission changes rather than sending notifications for every AWS event.

---

## Implementation

### CloudTrail Configuration

A dedicated CloudTrail trail was created to record management events. Logging was enabled and the trail was configured to store logs in an S3 bucket.

During troubleshooting, CloudTrail Event history confirmed that IAM events were being recorded, but automated email notifications were not arriving. Creating and enabling the dedicated trail was followed by a successful end-to-end test.

### EventBridge Rule

An EventBridge rule was created on the default event bus. Its custom JSON event pattern filters for the selected IAM API calls recorded through CloudTrail. The rule targets the SNS topic `week34-aws-security-alerts`.

### SNS Email Notifications

An SNS Standard topic named `week34-aws-security-alerts` was created. An email subscription was added and confirmed before testing the automated workflow.

SNS was first tested by publishing a manual message. The test email arrived successfully, confirming that the email subscription and notification destination were functioning correctly.

---

## Testing and Results

### Manual SNS Test

A manual test message was published directly to the SNS topic. The email arrived successfully, confirming the SNS email delivery path.

### IAM Group Creation Test

A temporary IAM group was created to generate a real `CreateGroup` management event. After the CloudTrail trail was enabled, an SNS notification arrived containing the CloudTrail event payload including the event name, IAM event source, AWS Region, and test group name.

### IAM Group Deletion Test

The temporary IAM group was deleted, generating a `DeleteGroup` event. A second automated email arrived containing the corresponding event details.

### Verified Outcome

| Test | Result |
|---|---|
| Manual SNS email publication | Passed |
| CloudTrail recording of IAM management events | Verified |
| EventBridge detection of `CreateGroup` | Passed |
| EventBridge detection of `DeleteGroup` | Passed |
| Automated SNS email delivery | Passed |

These tests verified the complete end-to-end workflow:

**IAM → CloudTrail → EventBridge → SNS → Email**

#Screenshots

CloudTrail trail details showing logging enabled, and Target SNS Topic.
<img width="2880" height="1800" alt="CloudTrail and EventBridge Targets" src="https://github.com/user-attachments/assets/12c42864-ba18-4e5a-ab76-9f84e37a963f" />

Email notification for the CreateGroup event and DeleteGroup event.
<img width="2880" height="1800" alt="Email notifications" src="https://github.com/user-attachments/assets/6df563c3-9951-4f55-b847-cde484624344" />

SNS topic and confirmed subscription, with personal details redacted.
<img width="2880" height="1800" alt="sns topic n email configuration" src="https://github.com/user-attachments/assets/5e5d5d5a-d218-4000-9153-1ef5923230d7" />

---

## Challenges and Troubleshooting

### Automated Emails Were Not Arriving

**Problem:** The SNS manual test succeeded, but creating and deleting the initial test group did not produce automated email notifications.

**Cause:** CloudTrail Event history showed the IAM API events were being recorded, but the dedicated CloudTrail trail had not yet been created and enabled for the full workflow.

**Resolution:** A CloudTrail trail was created with management read and write events enabled. After logging was confirmed active, a fresh IAM group creation generated an email notification and deleting the group generated a second alert.

---

### EventBridge Monitoring Metrics Showed No Data

**Problem:** The EventBridge Monitoring dashboard showed no data for MatchedEvents, Invocations, TriggeredRules, and FailedInvocations during initial troubleshooting.

**Cause:** The absence of dashboard data did not identify the exact failure point — it did not prove whether the rule had matched an event or attempted delivery.

**Resolution:** The event pattern, target configuration, CloudTrail logging status, and actual email payloads were each inspected independently. The EventBridge monitoring dashboard showed no data not because the rule was failing, but because no events had successfully passed through the complete workflow yet. Successful notifications for real `CreateGroup` and `DeleteGroup` events ultimately confirmed the workflow was functioning correctly.

This reinforced an important observability lesson: the absence of metric data is ambiguous. It can mean the rule is not matching events, or it can mean no events have occurred yet. Inspecting the complete path rather than relying on a single dashboard is the correct troubleshooting approach.

---

## Lessons Learned

This project demonstrated how AWS services can work together to build an event-driven security monitoring workflow. CloudTrail records management API activity, EventBridge filters events based on a structured JSON pattern, and SNS delivers notifications to a confirmed email subscriber. Each service has a distinct responsibility, and successful email delivery depends on the correct configuration of the entire path — not just each individual service in isolation.

One of the most important troubleshooting lessons was that seeing an event in CloudTrail Event history does not prove that the automated notification workflow is working. The event pattern, the rule target, the CloudTrail trail logging status, and the actual event payload each needed to be verified independently. Testing the complete workflow with real IAM group creation and deletion events provided stronger evidence than relying on configuration screens alone.

The distinction between a manual SNS test and an end-to-end test is significant. A manual SNS message confirms the notification destination is working, but it does not prove that EventBridge is detecting the intended events. A workflow can appear partially functional while the automated detection layer is not triggering at all. End-to-end testing with real events is the only way to confirm the complete path.

The EventBridge monitoring dashboard showing no data is ambiguous information. It can mean the rule is not matching events, or it can mean no matching events have occurred yet. Understanding what the absence of metric data does and does not tell you is an important observability skill — and a reminder that monitoring dashboards supplement investigation rather than replace it.

---

## Security Considerations

- Monitor only the IAM events required for the project scope
- Keep the SNS topic and email subscription restricted to authorized users
- Avoid including credentials, access keys, or sensitive account details in public screenshots
- Review CloudTrail S3 bucket access, encryption, and retention configuration
- Consider enabling multi-factor authentication and applying least-privilege access to administrative identities
- Treat email alerts as a detection mechanism, not a replacement for preventive controls or a complete incident response process

---

## Cost Considerations

The console indicated that no additional charges applied to management event logging for this trail because it was the first copy of those management events. CloudTrail S3 storage, EventBridge, and SNS usage may be subject to applicable service pricing depending on configuration and usage. Review AWS billing information and remove unused resources when the project is no longer needed.

---

## Future Improvements

- Expand monitoring to root account activity, suspicious console sign-ins, security group rule changes, and CloudTrail configuration changes
- Add structured alert formatting for more readable email notifications
- Integrate with a centralized logging or SIEM platform
- Add automated incident response triggered by high-severity events
- Apply infrastructure as code using Terraform

---

## Next Project

Week 35 — AWS Automated Security Incident Response

Week 35 builds directly on this project by adding automated investigation and response to the detection workflow. Instead of only sending an email notification when a security event occurs, a Lambda function will automatically investigate the event and perform a controlled response — such as disabling an access key, removing a user from a group, or tagging a resource for review.

The architecture extends the Week 34 pattern:

```text
IAM Security Change
        │
        ▼
CloudTrail → EventBridge
        │
        ▼
AWS Lambda
Automated Investigation & Response
        │
        ▼
SNS Email
Enriched Alert with Response Details
```

This introduces automated remediation alongside detection.

---

## Outcome

Successfully built and tested an end-to-end AWS security monitoring and email alerting workflow. Selected IAM management events are detected by EventBridge, routed through SNS, and delivered as email notifications. The complete workflow was validated with real `CreateGroup` and `DeleteGroup` events and confirmed end to end.
