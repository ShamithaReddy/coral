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

### Step 4 — Add the source

```sh
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
# Optional — defaults shown:
export AWS_REGION="us-east-1"             # us-gov-west-1 for GovCloud, cn-northwest-1 for China
export AWS_ENDPOINT_SUFFIX="amazonaws.com"  # amazonaws.com.cn for China
coral source add --file sources/community/aws_cost_explorer/manifest.yaml
```

`AWS_REGION` and `AWS_ENDPOINT_SUFFIX` default to `us-east-1` and
`amazonaws.com`. Override them only for AWS GovCloud (US) or the China
partition. Coral does not read your AWS CLI profile, AWS SSO cache, or
`AWS_PROFILE` — credentials must be supplied via the four environment
variables above (or `coral source add --interactive`).

## Tables

| Table | Description | Required filters |
|---|---|---|
| `aws_cost_explorer.cost_by_service` | Spend grouped by AWS service | `time_period_start`, `time_period_end` |
| `aws_cost_explorer.cost_by_tag` | Spend grouped by a resource tag key | `time_period_start`, `time_period_end`, `tag_key` |
| `aws_cost_explorer.anomalies` | Cost anomalies detected by AWS's ML models | `date_interval_start`, `date_interval_end` |
| `aws_cost_explorer.cost_forecast` | Projected cost for a future or in-progress period | `time_period_start`, `time_period_end`, `granularity`, `metric` |
| `aws_cost_explorer.rightsizing_recommendations` | EC2 downsize/terminate recommendations | — |
| `aws_cost_explorer.savings_plans_coverage` | Savings Plans coverage by service | `time_period_start`, `time_period_end` |

`cost_by_service` and `cost_by_tag` accept an optional `metric` filter
(`UNBLENDED_COST`, `BLENDED_COST`, `AMORTIZED_COST`, `NET_AMORTIZED_COST`,
`NET_UNBLENDED_COST`, `USAGE_QUANTITY`, `NORMALIZED_USAGE_AMOUNT`) and an
optional `granularity` filter (`MONTHLY`, `DAILY`, `HOURLY`). Both default to
`UNBLENDED_COST` and `MONTHLY` when omitted. `HOURLY` is opt-in on the AWS
account and limited to the last 14 days. Per-metric typed columns
(`unblended_cost`, `blended_cost`, `amortized_cost`, `net_amortized_cost`,
`net_unblended_cost`, `usage_quantity`, `normalized_usage_amount`) are
populated only for the requested `metric`; the other six are NULL on the
same row. `period_total_cost` and `unit` always reflect the requested
metric.

For example, `metric = 'AMORTIZED_COST'` produces:

| period_start | period_total_cost | amortized_cost | unblended_cost | blended_cost |
|---|---|---|---|---|
| 2026-04-01   | 3421.78           | 3421.78        | NULL           | NULL         |

`unblended_cost` and the other five metric columns return NULL because the
API only returns the metric you asked for. Use the typed column matching
your filter, or the always-populated `period_total_cost`.

### `aws_cost_explorer.cost_by_service`

Returns one row per billing period with the aggregate cost and a `groups` JSON
array containing the per-service breakdown. `time_period_end` is **exclusive**
— use the first day of the following month to include all days of the target month.

```sql
SELECT
    period_start,
    period_total_cost,
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

Returns one row per billing period with the aggregate cost and a `groups` JSON
array containing the per-tag-value breakdown. The AWS-managed tag
`aws:cloudformation:stack-name` is automatically activated — use it to
attribute costs to CloudFormation stacks.

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

Projects cost for a future or in-progress period. `time_period_start` must be
today or later — past dates return `ValidationException`. `DAILY` allows up
to 3 months ahead; `MONTHLY` up to 18 months. Supported metrics are narrower
than `cost_by_service` — only `UNBLENDED_COST`, `BLENDED_COST`,
`AMORTIZED_COST`, `NET_AMORTIZED_COST`, and `NET_UNBLENDED_COST`.

```sql
SELECT
    period_start,
    CAST(mean_value AS DOUBLE)                       AS expected_usd,
    CAST(prediction_interval_upper_bound AS DOUBLE)  AS conservative_usd
FROM aws_cost_explorer.cost_forecast
WHERE time_period_start         = '2026-05-29'
  AND time_period_end           = '2026-06-01'
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
signal to review commitment purchases. Returns
`DataUnavailableException` from AWS when the account has no Savings Plans
in the requested period — that is account state, not a query error.

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
  start date to be within the last 13 months. `anomalies` retains up to
  90 days. `cost_forecast` requires `time_period_start` to be today or
  later. The example dates above stay valid as long as the underlying
  retention window includes them — refresh the dates if you re-run after
  a long gap.
- **String columns to cast:** every numeric-looking column on every table
  in this source is `Utf8` and must be cast before arithmetic. This
  includes all `*_cost` and `*_amount` columns, `period_total_cost`,
  `mean_value`, `prediction_interval_lower_bound`,
  `prediction_interval_upper_bound`, `total_impact`,
  `total_impact_percentage`, `max_impact`, `total_actual_spend`,
  `total_expected_spend`, `monthly_cost`, `estimated_monthly_savings`,
  `coverage_pct`, `cpu_utilization_pct`, `memory_utilization_pct`, and
  `usage_quantity` / `normalized_usage_amount`. Use
  `CAST(<column> AS DOUBLE)` before any comparison or arithmetic.
  Boolean (`actions_enabled`, `estimated`), Float64
  (`current_score`, `max_score`), and Int64 columns project as their
  declared types.
- **Tag activation:** User-defined tags must be activated under
  **Billing → Cost allocation tags** before they appear in `cost_by_tag`.
  The `aws:cloudformation:stack-name` tag is auto-activated.
- **Cost Explorer must be enabled:** The API returns `OptInRequired` if Cost
  Explorer has never been activated in the account.
