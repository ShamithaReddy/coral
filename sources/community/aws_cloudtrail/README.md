# AWS CloudTrail

Query AWS management events via
[AWS CloudTrail LookupEvents](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/API_LookupEvents.html).
Captures every AWS API call that creates, modifies, or deletes a resource —
regardless of whether the caller used CloudFormation, Terraform, CDK, GitHub
Actions, the AWS CLI, or the console.

The 90-day event history is available in every AWS account where CloudTrail is
enabled (default since 2019) with no additional setup or S3 log access required.

## Authentication

AWS CloudTrail uses AWS Signature Version 4. You need an IAM user or role with
the `cloudtrail:LookupEvents` permission.

### Step 1 — Create an IAM policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["cloudtrail:LookupEvents"],
      "Resource": "*"
    }
  ]
}
```

### Step 2 — Create access keys

Attach the policy to an IAM user or role and generate an access key pair.

### Step 3 — Add the source

```sh
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
coral source add --file sources/community/aws_cloudtrail/manifest.yaml
```

## Tables

| Table | Description | Required filters |
|---|---|---|
| `cloudtrail.management_events` | All write-only management events across all AWS services | `start_time`, `end_time` |
| `cloudtrail.lambda_events` | Lambda function changes (UpdateFunctionConfiguration, UpdateFunctionCode, etc.) | `start_time`, `end_time` |
| `cloudtrail.cloudformation_events` | CloudFormation stack operations (UpdateStack, ExecuteChangeSet, etc.) | `start_time`, `end_time` |
| `cloudtrail.ec2_events` | EC2 instance lifecycle and configuration changes (RunInstances, ModifyInstanceAttribute, etc.) | `start_time`, `end_time` |

All time filters are **Unix epoch seconds** (Int64).

## Example queries

### Find all infrastructure changes in the last 24 hours

```sql
SELECT event_name, event_source, resource_name, username, event_time
FROM cloudtrail.management_events
WHERE start_time = CAST(EXTRACT(EPOCH FROM NOW() - INTERVAL '24 hours') AS BIGINT)
  AND end_time   = CAST(EXTRACT(EPOCH FROM NOW()) AS BIGINT)
ORDER BY event_time DESC
LIMIT 50
```

### Find Lambda changes near a specific time (for cost spike investigation)

```sql
SELECT event_name, resource_name, username, event_time, cloudtrail_event
FROM cloudtrail.lambda_events
WHERE start_time = 1715558400
  AND end_time   = 1715644800
ORDER BY event_time DESC
```

### Find CloudFormation deployments and correlate with GitHub PRs

```sql
SELECT
    ct.resource_name  AS stack_name,
    ct.event_name     AS operation,
    ct.event_time     AS deploy_time,
    ct.username       AS deployed_by,
    g.number          AS pr_number,
    g.title           AS pr_title,
    g.author_login    AS pr_author,
    g.merged_at
FROM cloudtrail.cloudformation_events ct
JOIN github.pulls g
    ON g.merged_at <= ct.event_time
    AND g.merged_at >= ct.event_time - INTERVAL '30 minutes'
    AND g.owner = 'your-org'
    AND g.state = 'closed'
WHERE ct.start_time = CAST(EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days') AS BIGINT)
  AND ct.end_time   = CAST(EXTRACT(EPOCH FROM NOW()) AS BIGINT)
  AND ct.event_name IN ('UpdateStack', 'ExecuteChangeSet', 'CreateStack')
ORDER BY ct.event_time DESC
```

### Inspect what changed (e.g. Lambda memory)

```sql
SELECT
    resource_name,
    event_time,
    username,
    json_get_str(cloudtrail_event, 'requestParameters') AS request_params
FROM cloudtrail.lambda_events
WHERE start_time = 1715558400
  AND end_time   = 1715644800
  AND event_name = 'UpdateFunctionConfiguration'
```

## Notes

- **Time filters are Unix epoch seconds:** Convert with
  `CAST(EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days') AS BIGINT)`.
- **90-day retention:** LookupEvents only returns events from the last 90 days.
  For older data, query CloudTrail logs in S3 via Athena.
- **Rate limit:** 1 request per second per account. Queries that paginate
  through many pages may hit this limit.
- **CloudTrail must be enabled:** Most accounts have CloudTrail enabled by
  default. If not, events will be empty rather than returning an error.
- **`cloudtrail_event` is a JSON string:** Use `json_get_str` to extract nested
  fields. The `requestParameters` field shows what values were set. The
  `clientRequestToken` field (when set by CI/CD) can be a direct foreign key to
  the commit or PR that triggered the change.
- **Region matters:** Each region has its own CloudTrail event history. Query
  the region where your resources are deployed.

### `cloudtrail.ec2_events`

EC2 instance and resource write events. Captures instance launches,
terminations, type changes, and storage operations.

```sql
-- Find new EC2 instances launched in the last 30 days
SELECT event_name, resource_name, username, event_time
FROM cloudtrail.ec2_events
WHERE start_time = CAST(EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days') AS BIGINT)
  AND end_time   = CAST(EXTRACT(EPOCH FROM NOW()) AS BIGINT)
  AND event_name = 'RunInstances'
ORDER BY event_time DESC

-- Combine with Cost Explorer to see cost of instances launched this month
SELECT
    ec.resource_name  AS instance_id,
    ec.username       AS launched_by,
    ec.event_time     AS launch_time,
    ce.total_cost     AS monthly_cost_usd
FROM cloudtrail.ec2_events ec
JOIN aws_cost_explorer.cost_by_service ce
    ON ce.service = 'Amazon EC2'
    AND ce.time_period_start = '2025-05-01'
    AND ce.time_period_end   = '2025-06-01'
WHERE ec.start_time = CAST(EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days') AS BIGINT)
  AND ec.end_time   = CAST(EXTRACT(EPOCH FROM NOW()) AS BIGINT)
  AND ec.event_name = 'RunInstances'
```
