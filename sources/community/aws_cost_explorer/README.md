# AWS Cost Explorer

Query AWS billing data including cost by service, cost by resource tag,
detected cost anomalies, cost forecasts, EC2 rightsizing recommendations, and
Savings Plans coverage from
[AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/).

## Authentication

AWS Cost Explorer uses AWS Signature Version 4. You need an IAM user with
read-only Cost Explorer permissions.

### Step 1 — Create an IAM policy

Create a policy with the following permissions (all read-only):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetAnomalies",
        "ce:GetCostForecast",
        "ce:GetRightsizingRecommendation",
        "ce:GetSavingsPlansCoverage"
      ],
      "Resource": "*"
    }
  ]
}
```

Cost Explorer IAM actions only support `Resource: "*"` — resource-level
scoping is not available.

### Step 2 — Enable Cost Explorer

Cost Explorer must be activated before the API returns data. In the AWS
console, go to **Billing and Cost Management → Cost Explorer → Enable**.
After activation, the API becomes available within 24 hours.

### Step 3 — Create access keys

Attach the policy to an IAM user and generate an access key pair.

> **Long-term IAM keys only.** This source signs requests with the
> `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` pair as-is. Temporary
> credentials that require a session token — STS `AssumeRole`, IAM
> Identity Center / SSO, EC2 instance roles, EKS / IRSA — are **not
> supported** because there is no manifest path to thread
> `AWS_SESSION_TOKEN` through the SigV4 authenticator yet. Access keys
> whose ID starts with `ASIA` are session-scoped and will fail with
> `InvalidClientTokenId` or `SignatureDoesNotMatch`. Use a long-term IAM
> user key (the ID starts with `AKIA`).

### Step 4 — Add the source

```sh
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
# Optional — defaults shown:
export AWS_REGION="us-east-1"             # cn-northwest-1 for the China partition
export AWS_ENDPOINT_SUFFIX="amazonaws.com"  # amazonaws.com.cn for the China partition
coral source add --file sources/community/aws_cost_explorer/manifest.yaml
```

`AWS_REGION` and `AWS_ENDPOINT_SUFFIX` default to `us-east-1` and
`amazonaws.com`. Override them only for the China partition. Cost
Explorer is a global service with one regional endpoint per partition,
and AWS GovCloud (US) does not expose a Cost Explorer endpoint. Coral
does not read your AWS CLI profile, AWS SSO cache, or `AWS_PROFILE` —
credentials must be supplied via the four environment variables above
(or `coral source add --interactive`).

## Tables

| Table | Description | Required filters |
|---|---|---|
| `aws_cost_explorer.cost_by_service` | Spend grouped by AWS service | `time_period_start`, `time_period_end` |
| `aws_cost_explorer.cost_by_tag` | Spend grouped by a resource tag key | `time_period_start`, `time_period_end`, `tag_key` |
| `aws_cost_explorer.anomalies` | Cost anomalies detected by AWS's ML models | `date_interval_start`, `date_interval_end` |
| `aws_cost_explorer.cost_forecast` | Projected cost for an in-progress or future period | `time_period_start`, `time_period_end`, `granularity`, `metric` |
| `aws_cost_explorer.rightsizing_recommendations` | EC2 downsize/terminate recommendations | — |
| `aws_cost_explorer.savings_plans_coverage` | Savings Plans coverage by service | `time_period_start`, `time_period_end` |

`cost_by_service` and `cost_by_tag` accept an optional `metric` filter
(`UnblendedCost`, `BlendedCost`, `AmortizedCost`, `NetAmortizedCost`,
`NetUnblendedCost`, `UsageQuantity`, `NormalizedUsageAmount`) and an
optional `granularity` filter (`MONTHLY`, `DAILY`, `HOURLY`). Both default to
`UnblendedCost` and `MONTHLY` when omitted. `HOURLY` is opt-in at the
AWS payer-account level and limited to the last 14 days; it also
requires `time_period_start` and `time_period_end` to be passed as
ISO-8601 timestamps (e.g. `'2026-05-29T00:00:00Z'`) rather than
`YYYY-MM-DD` dates, or AWS will reject the request with
`ValidationException` (`"Time period is invalid. Valid format is
yyyy-MM-ddThh:mm:ssZ."`).

Both tables hardcode a `GroupBy` on the request — `SERVICE` for
`cost_by_service`, the supplied `tag_key` for `cost_by_tag` — and AWS
returns the response `Total` block as an empty object whenever a
`GroupBy` is set. The cost values therefore live exclusively under the
`groups` JSON array; there is no scalar period-total column. Sum across
the entries in `groups` to compute a period-level aggregate.

### `aws_cost_explorer.cost_by_service`

Returns one row per billing period with a `groups` JSON array containing the
per-service breakdown. `time_period_end` is **exclusive** — use the first day
of the following month to include all days of the target month.

```sql
SELECT
    period_start,
    estimated,
    groups
FROM aws_cost_explorer.cost_by_service
WHERE time_period_start = '2026-05-01'
  AND time_period_end   = '2026-06-01'
```

To work with individual services, use `json_get_json` on the `groups` column:

```sql
SELECT
    period_start,
    json_get_json(groups, 0)                                                    AS first_service_json,
    json_get_str(json_get_json(groups, 0), 'Keys', 0)                          AS first_service_name,
    json_get_str(json_get_json(groups, 0), 'Metrics', 'UnblendedCost', 'Amount') AS first_service_cost
FROM aws_cost_explorer.cost_by_service
WHERE time_period_start = '2026-05-01'
  AND time_period_end   = '2026-06-01'
```

### `aws_cost_explorer.cost_by_tag`

Returns one row per billing period with a `groups` JSON array containing the
per-tag-value breakdown. To attribute costs to CloudFormation stacks, group
by the AWS-generated tag `aws:cloudformation:stack-name`. Both user-defined
tags and AWS-generated tags must be activated under
**Billing → Cost Allocation Tags** before they return any rows. Activation
is non-retroactive (older billing periods stay empty) and only the
management account can activate tags in AWS Organizations.

```sql
SELECT
    period_start,
    groups
FROM aws_cost_explorer.cost_by_tag
WHERE time_period_start = '2026-05-01'
  AND time_period_end   = '2026-06-01'
  AND tag_key           = 'aws:cloudformation:stack-name'
```

### `aws_cost_explorer.anomalies`

Returns cost anomalies detected by AWS's ML baseline. AWS retains anomalies
for up to 90 days. `anomaly_end_date` is `NULL` for ongoing spikes.
`date_interval_end` has an upper bound of "yesterday" — AWS rejects
end dates of today or later with `ValidationException` (the live error
reads `"Latest supported detectionDate for GetRecentAnomalies is …"`).
Optional filters: `total_impact_min` (whole-dollar threshold),
`feedback` (`YES`, `NO`, `PLANNED_ACTIVITY` — user-supplied classification),
and `monitor_arn` (restrict to a specific Cost Anomaly Monitor).

```sql
SELECT
    anomaly_start_date,
    dimension_value AS service,
    CAST(total_impact AS DOUBLE) AS impact_usd,
    CAST(total_impact_percentage AS DOUBLE) AS pct_above_expected
FROM aws_cost_explorer.anomalies
WHERE date_interval_start = '2026-04-01'
  AND date_interval_end   = '2026-05-29'
  AND total_impact_min    = 50
ORDER BY CAST(total_impact AS DOUBLE) DESC
```

To inspect root causes, drill into the `root_causes` JSON array:

```sql
SELECT
    anomaly_id,
    json_get_str(json_get_json(root_causes, 0), 'Service')     AS top_root_service,
    json_get_str(json_get_json(root_causes, 0), 'Region')      AS top_root_region,
    json_get_str(json_get_json(root_causes, 0), 'UsageType')   AS top_root_usage_type
FROM aws_cost_explorer.anomalies
WHERE date_interval_start = '2026-04-01'
  AND date_interval_end   = '2026-05-29'
```

### `aws_cost_explorer.cost_forecast`

Projects cost for an in-progress or future period. `time_period_start` has
a hard lower bound of "yesterday" — AWS rejects earlier starts with
`ValidationException` (`"Earliest supported Start is …"`). Future starts
are accepted in practice even though AWS docs say the start "must be equal
to or no later than the current date"; in live tests AWS returns
`DataUnavailableException` ("Insufficient amount of historical data")
rather than rejecting the start date itself when there is no history to
forecast from. `time_period_end` has its own ceiling: AWS rejects ends
more than ~3 months ahead at `DAILY` granularity (or ~18 months ahead
at `MONTHLY`) with `ValidationException`
(`"Latest supported End is …"`). The forecast covers `time_period_start`
through `time_period_end` exclusive; AWS blends actuals up to today with
a prediction beyond. Supported metrics are narrower than `cost_by_service`
— only `UNBLENDED_COST`, `BLENDED_COST`, `AMORTIZED_COST`,
`NET_AMORTIZED_COST`, and `NET_UNBLENDED_COST`.

```sql
-- Forecast from yesterday through end of next month. Use yesterday's
-- date as the start because earlier starts are rejected by AWS.
SELECT
    period_start,
    CAST(mean_value AS DOUBLE)                       AS expected_usd,
    CAST(prediction_interval_upper_bound AS DOUBLE)  AS conservative_usd
FROM aws_cost_explorer.cost_forecast
WHERE time_period_start         = '2026-05-30'
  AND time_period_end           = '2026-07-01'
  AND granularity               = 'DAILY'
  AND metric                    = 'UNBLENDED_COST'
  AND prediction_interval_level = 80
```

### `aws_cost_explorer.rightsizing_recommendations`

Returns EC2 instances AWS recommends downsizing or terminating based on 14
days of CloudWatch CPU data. Memory-based recommendations require the
CloudWatch agent.

```sql
SELECT
    instance_id,
    instance_type,
    CAST(monthly_cost AS DOUBLE)             AS current_monthly_usd,
    recommendation_type,
    target_instance_type,
    CAST(estimated_monthly_savings AS DOUBLE) AS savings_usd
FROM aws_cost_explorer.rightsizing_recommendations
WHERE recommendation_type = 'MODIFY'
ORDER BY CAST(estimated_monthly_savings AS DOUBLE) DESC
```

### `aws_cost_explorer.savings_plans_coverage`

Shows how much eligible on-demand spend is covered by an active Savings
Plan. Coverage below 80% with significant `on_demand_cost` is a strong
signal to review commitment purchases. AWS returns
`DataUnavailableException` (HTTP 400) when the account has no Savings
Plans in the requested period; this surfaces as a query error rather
than zero rows. Treat that error as account state, not a manifest
problem. `time_period_end` is exclusive and AWS rejects end dates after
today with `ValidationException`
(`"end date should NOT be after [today]"`); use today's date for an
in-progress month.

`coverage_pct` is computed by AWS as
`spend_covered_by_savings_plans / total_cost * 100`, where
`total_cost = on_demand_cost + spend_covered_by_savings_plans`.

```sql
SELECT
    service,
    CAST(coverage_pct AS DOUBLE)                     AS coverage_pct,
    CAST(on_demand_cost AS DOUBLE)                   AS uncovered_usd,
    CAST(spend_covered_by_savings_plans AS DOUBLE)   AS covered_usd,
    CAST(total_cost AS DOUBLE)                       AS total_usd
FROM aws_cost_explorer.savings_plans_coverage
WHERE time_period_start = '2026-04-01'
  AND time_period_end   = '2026-05-01'
  AND CAST(coverage_pct AS DOUBLE) < 80
ORDER BY CAST(on_demand_cost AS DOUBLE) DESC
```

## Notes

- **Data lag:** Cost Explorer data has a ~24-hour lag. The current open month
  is marked `estimated = true`.
- **End date is exclusive:** `time_period_end = '2026-06-01'` includes all of
  May 2026 but no June data.
- **Data retention:** `cost_by_service` and `cost_by_tag` keep 14 months of
  daily data by default and up to 38 months of monthly data when the
  multi-year opt-in is enabled. `savings_plans_coverage` requires the
  start date to be within the last 13 months and the end date to be
  today or earlier (AWS rejects future end dates). `anomalies` retains
  up to 90 days and `date_interval_end` is bounded at "yesterday" by
  AWS. `cost_forecast` requires `time_period_start` to be yesterday or
  later — AWS rejects earlier starts. The example dates above stay
  valid as long as the underlying retention window includes them —
  refresh the dates if you re-run after a long gap.
- **String columns to cast:** every numeric-looking column on every table
  in this source is `Utf8` and must be cast before arithmetic. This
  includes `mean_value`, `prediction_interval_lower_bound`,
  `prediction_interval_upper_bound`, `total_impact`,
  `total_impact_percentage`, `max_impact`, `total_actual_spend`,
  `total_expected_spend`, `monthly_cost`, `estimated_monthly_savings`,
  `cpu_utilization_pct`, `memory_utilization_pct`, `coverage_pct`,
  `on_demand_cost`, `spend_covered_by_savings_plans`, and `total_cost`.
  Per-period totals on `cost_by_service` and `cost_by_tag` live inside
  the `groups` JSON array — extract with `json_get_str` and cast the
  result the same way. Use `CAST(<column> AS DOUBLE)` before any
  comparison or arithmetic. Boolean (`estimated`), Float64
  (`current_score`, `max_score`), and Int64 columns project as their
  declared types.
- **Tag activation:** Both user-defined and AWS-generated cost allocation
  tags (including `aws:cloudformation:stack-name`) must be activated under
  **Billing → Cost Allocation Tags** before they appear in `cost_by_tag`.
  Activation is non-retroactive — older billing periods stay empty even
  after activation. In AWS Organizations, only the management account can
  activate cost allocation tags.
- **Cost Explorer must be enabled:** queries fail with an
  `AccessDeniedException` referencing `ce:GetCostAndUsage` (or whichever
  operation the table calls) until Cost Explorer is activated for the
  account in **Billing → Cost Explorer**.
