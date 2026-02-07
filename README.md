# AWS Logs Exporter

## Contributors

**Vinay Selvaraj** - [LinkedIn](https://linkedin.com/in/vinayselvaraj) | [X](https://x.com/vinayselvaraj)

---

Automatically exports all CloudWatch Log Groups to S3 on a recurring schedule. Deployed as a single CloudFormation stack, it uses an AWS Step Functions state machine triggered by EventBridge to enumerate every log group in the account and region, then creates export tasks that write log data to a KMS-encrypted S3 bucket.

## Why Export CloudWatch Logs to S3?

- **Cost reduction** - CloudWatch Logs storage costs $0.03/GB/month. S3 Standard costs $0.023/GB/month and S3 Glacier costs as little as $0.0036/GB/month. For large log volumes, the savings are significant.
- **Long-term retention** - S3 lifecycle policies let you automatically transition logs to cheaper storage classes (Intelligent-Tiering, Glacier, Deep Archive) or expire them after a set period, giving you flexible retention strategies beyond what CloudWatch offers.
- **Analytics** - Logs in S3 can be queried directly with Amazon Athena, processed with AWS Glue, or loaded into OpenSearch. This enables ad-hoc analysis, dashboarding, and correlation across log groups without CloudWatch Logs Insights query costs.
- **Compliance and archival** - Many compliance frameworks require durable, immutable log archives. S3 Object Lock and versioning provide write-once-read-many (WORM) guarantees that satisfy these requirements.
- **Cross-account and cross-region access** - S3 bucket policies and replication rules make it straightforward to centralize logs from multiple accounts and regions into a single location.

## Architecture

```
EventBridge Rule (hourly)
        |
        v
Step Functions State Machine
        |
        ├── DescribeLogGroups (paginated)
        |
        └── For each log group (sequential):
                └── CreateExportTask --> S3 Bucket (KMS encrypted)
```

The state machine:
1. Calculates the time range for the previous hour
2. Paginates through all CloudWatch Log Groups in the account/region
3. Creates an export task for each log group, writing to the S3 bucket
4. Handles `LimitExceededException` with exponential backoff and jitter (the CreateExportTask API allows only one active export task at a time)

Exported logs are organized in S3 by date, hour, and log group name:

```
s3://<bucket>/YYYY_MM_DD/HH/<log-group-name>/
```

## Resources Created

| Resource | Description |
|---|---|
| KMS Key + Alias | Encrypts objects in the S3 bucket |
| S3 Bucket | Stores exported log data with lifecycle expiration |
| S3 Bucket Policy | Grants the CloudWatch Logs service permission to write |
| Step Functions State Machine | Orchestrates the log export workflow |
| EventBridge Rule | Triggers the state machine on a schedule |
| IAM Roles (x2) | Execution roles for Step Functions and EventBridge |
| CloudWatch Log Group | Captures state machine execution logs |

## Prerequisites

- AWS CLI configured with credentials that have permission to create the resources above
- An AWS account with CloudWatch Log Groups you want to export

## Deployment

Deploy with default settings:

```bash
aws cloudformation deploy \
  --stack-name log-exporter \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

Deploy with custom parameters:

```bash
aws cloudformation deploy \
  --stack-name log-exporter \
  --template-file template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    BucketPrefix=my-logs \
    ExpirationInDays=90 \
    ScheduleExpression="rate(6 hours)"
```

## Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `BucketPrefix` | String | `logs` | Prefix for the S3 bucket name. The full bucket name will be `<prefix>-<account-id>-<region>`. |
| `ExpirationInDays` | Number | `30` | Number of days before exported log objects are automatically deleted from S3. |
| `KmsKeyAliasName` | String | `alias/logs-bucket-key` | Alias name for the KMS key used to encrypt the S3 bucket. |
| `ScheduleExpression` | String | `rate(1 hour)` | EventBridge schedule expression that controls how often the export runs. Accepts `rate()` or `cron()` syntax. |
| `StateMachineLogLevel` | String | `ALL` | Logging level for the Step Functions state machine. Allowed values: `ALL`, `ERROR`, `FATAL`, `OFF`. |
| `StateMachineLogRetentionInDays` | Number | `1` | Retention period for the state machine's own CloudWatch log group. Must be a valid CloudWatch retention value. |

### Examples

**Export every 6 hours and keep logs for 90 days:**

```bash
--parameter-overrides ScheduleExpression="rate(6 hours)" ExpirationInDays=90
```

**Export daily at midnight UTC:**

```bash
--parameter-overrides ScheduleExpression="cron(0 0 * * ? *)"
```

**Reduce costs by turning off state machine execution logging:**

```bash
--parameter-overrides StateMachineLogLevel=OFF
```

## Outputs

| Output | Description |
|---|---|
| `BucketName` | Name of the S3 bucket where logs are exported |
| `BucketArn` | ARN of the S3 bucket |

Both outputs are exported for cross-stack reference using the names `<stack-name>-BucketName` and `<stack-name>-BucketArn`.

## Querying Exported Logs with Athena

Once logs are in S3, you can query them with Amazon Athena. Create a table pointing at the bucket and use standard SQL to search across all your log groups.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name log-exporter
```

Note: The S3 bucket must be empty before the stack can be deleted. Empty it first:

```bash
aws s3 rm s3://<bucket-name> --recursive
```
