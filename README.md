# Daily Spend Guardrail

This Declarative Automation Bundle deploys a daily Databricks SQL Alert that:

- estimates the previous usage date's USD list cost from `system.billing.usage` and `system.billing.list_prices`;
- alerts when the daily total is greater than a configurable threshold;
- includes the top five workspace/SKU contributors in the result by default; and
- only notifies the subscriber—it does not stop clusters, warehouses, jobs, or other compute.

It also deploys an AI/BI dashboard with yesterday-versus-prior-day spend, seven- and thirty-day KPIs, a filterable 90-day trend, account and workspace selectors, and yesterday's top five workspace/SKU contributors.

The estimate uses published list prices. It is not an invoice and does not include contract discounts, credits, taxes, or every possible billing adjustment.

## Prerequisites

- Databricks CLI `0.292.0` or newer and an authenticated profile.
- A Pro or Serverless SQL warehouse with permission to use it.
- Access to `system.billing.usage` and `system.billing.list_prices`.
- Permission to create SQL Alerts.
- An existing workspace group for alert access. DABs do not manage workspace SCIM groups.

## Configuration

Copy a sample environment file. Real `.env` files are gitignored:

```bash
cp .env.deployment.example .env.production
```

Replace every placeholder, authenticate the selected profile, and find an appropriate warehouse:

```bash
set -a
source .env.production
set +a
databricks auth login --profile "$DATABRICKS_PROFILE"
databricks warehouses list --profile "$DATABRICKS_PROFILE"
```

Bundle variables use Databricks' `BUNDLE_VAR_<name>` environment convention. `set -a` exports the values loaded from the environment file. Every Databricks CLI command passes the selected profile explicitly.

## Variables

Bundle variables and defaults are defined in `databricks.yml`:

| Variable | Default | Purpose |
|---|---:|---|
| `deployment_label` | `Workspace` | short label included in resource names |
| `warehouse_id` | required | SQL warehouse that evaluates the alert |
| `daily_budget_usd` | `1000` | notification threshold in USD |
| `top_n` | `5` | contributors included in the result |
| `alert_cron` | `0 0 9 * * ?` | daily Quartz schedule |
| `alert_timezone` | `America/Denver` | schedule timezone |
| `alert_pause_status` | `UNPAUSED` | `PAUSED` during testing or `UNPAUSED` when approved |
| `alert_recipient_email` | required | workspace user email receiving notifications |
| `alert_access_group` | required | existing group granted `CAN_RUN` |

To use Slack, Teams, PagerDuty, or a webhook instead of email, replace the subscription with a pre-created notification destination:

```yaml
subscriptions:
  - destination_id: 00000000-0000-0000-0000-000000000000
```

The SQL Alert is defined in `resources/daily_spend.alert.yml`. After deployment, workspace users with permission can also view and manage it under **SQL → Alerts**. Dashboard subscriptions are delivery schedules for dashboard snapshots, not spend-threshold alerts.

The configured workspace group receives `CAN_RUN` permission on the alert. Databricks SQL Alert subscriptions do not expand workspace groups into recipients: they accept only a workspace user email or a notification destination ID. The bundle therefore sends email to `alert_recipient_email` while the group controls alert access. To send to a team, use a mail distribution-list address associated with a workspace user or create a supported notification destination and set its `destination_id`.

The dashboard source is `src/dashboards/daily_spend.lvdash.json`. Its datasets use `system.billing` as the catalog/schema context configured in `resources/daily_spend.dashboard.yml`.

## Deployment

Create the group named by `BUNDLE_VAR_alert_access_group` in **Admin Settings → Identity and access → Groups**, then add the intended users. This is a one-time workspace prerequisite outside the DAB.

Validate the complete bundle:

```bash
databricks bundle validate --strict \
  --target "$DATABRICKS_TARGET" \
  --profile "$DATABRICKS_PROFILE"
```

Deploy the dashboard and paused alert separately during acceptance testing:

```bash
databricks bundle deploy \
  --target "$DATABRICKS_TARGET" \
  --select dashboards.daily_spend_dashboard \
  --profile "$DATABRICKS_PROFILE"

databricks bundle deploy \
  --target "$DATABRICKS_TARGET" \
  --select alerts.daily_spend_guardrail \
  --profile "$DATABRICKS_PROFILE"
```

Review dashboard results, threshold, schedule, warehouse, recipient, group membership, and the alert query. Then change `BUNDLE_VAR_alert_pause_status=UNPAUSED` and redeploy only the alert:

```bash
databricks bundle validate --strict \
  --target "$DATABRICKS_TARGET" \
  --profile "$DATABRICKS_PROFILE"

databricks bundle deploy \
  --target "$DATABRICKS_TARGET" \
  --select alerts.daily_spend_guardrail \
  --profile "$DATABRICKS_PROFILE"
```

## Behavior

The alert runs once daily and evaluates the previous `usage_date`, which gives billing records time to arrive. If no rows exist, the alert state is `OK`. If spend stays over the threshold, `retrigger_seconds: 1` allows the daily evaluation to notify again. The schedule—not the retrigger setting—limits evaluations to once per day.

For stronger coverage of late-arriving corrections, add a second alert that checks a rolling multi-day window. Keep that separate from this alert so each notification still names one unambiguous usage date.

Compute policies can constrain how users create compute, but they do not make this alert a hard budget stop. See `COMPUTE_POLICIES.md`.
