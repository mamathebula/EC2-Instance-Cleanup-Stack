# EC2 Instance Cleanup Stack

CloudFormation stack that automatically terminates EC2 instances running for more than 1 hour that are not tagged with `do-not-delete`.

## How It Works

- Lambda runs every 30 minutes (configurable)
- Checks all EC2 instances in the region (pending, running, stopping, stopped)
- If an instance has been running for more than 1 hour and has no tag containing `do-not-delete` → terminates it
- Sends an email notification listing terminated and protected instances

## How to Protect an Instance

Add any tag with a value containing `do-not-delete` to the instance. For example:

- Tag Key: `Name`, Value: `my-server-do-not-delete`
- Tag Key: `Protection`, Value: `do-not-delete`

The check is case-insensitive, so `Do-Not-Delete` or `DO-NOT-DELETE` also works.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ExcludeString` | `do-not-delete` | Instances with this string in any tag value are protected |
| `NotificationEmail` | *(required)* | Email for termination notifications |
| `ScheduleExpression` | `rate(30 minutes)` | How often the cleanup runs |

## Deployment

### AWS Console

1. Go to CloudFormation → Create Stack → Upload a template file
2. Upload `ec2-instance-cleanup.yaml`
3. Enter your email address
4. Acknowledge IAM resource creation
5. Create stack
6. Confirm the SNS subscription email you receive

### AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name ec2-instance-cleanup \
  --template-body file://ec2-instance-cleanup.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
  --capabilities CAPABILITY_NAMED_IAM
```

### CloudShell

1. Log into the AWS Console
2. Open CloudShell (terminal icon, top right)
3. Make sure you're in the correct region
4. Click Actions → Upload file → select `ec2-instance-cleanup.yaml`
5. If re-uploading, delete the old file first:

```bash
rm ec2-instance-cleanup.yaml
```

6. Run:

```bash
aws cloudformation create-stack \
  --stack-name ec2-instance-cleanup \
  --template-body file://ec2-instance-cleanup.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
  --capabilities CAPABILITY_NAMED_IAM
```

## What Gets Created

| Resource | Type | Description |
|----------|------|-------------|
| Lambda Function | `AWS::Lambda::Function` | Python 3.13, 256 MB memory, 5 min timeout — runs the cleanup logic |
| IAM Role | `AWS::IAM::Role` | EC2 describe/terminate and SNS publish permissions only |
| SNS Topic | `AWS::SNS::Topic` | Sends email notifications when instances are terminated |
| EventBridge Rule | `AWS::Events::Rule` | Triggers the Lambda on schedule (default every 30 minutes) |
| Lambda Permission | `AWS::Lambda::Permission` | Allows EventBridge to invoke the Lambda function |

## Email Notification Example

```
EC2 Instance Cleanup Report
Region: us-east-1
Time: 2026-03-19 14:30:00 UTC

TERMINATED (2):
  - i-0abc123 (test-server) - running for 3:45:00
  - i-0def456 (dev-instance) - running for 12:30:00

PROTECTED (1):
  - i-0ghi789 (prod-server-do-not-delete)
```

## Important Notes

- The Lambda checks ALL instances (running, stopped, pending, stopping) in the region, including ones created before the stack was deployed
- Tag your instances with `do-not-delete` BEFORE deploying this stack
- The cleanup stack itself is not affected (it only targets EC2 instances, not CloudFormation stacks)
- To test manually: go to Lambda console → find the function → click Test → send `{}` as the event
- Logs are available in CloudWatch at `/aws/lambda/{stack-name}-function`
