# Compute Policies and Spend Guardrails

Compute policies are preventive configuration controls for classic all-purpose and jobs compute. They complement this daily notification, but they do not stop compute when account spend reaches a dollar threshold.

## Recommended Controls

- Cap `dbus_per_hour` to bound the maximum DBU rate of a cluster.
- Limit autoscaling maximum workers and disallow oversized node types.
- Require short auto-termination for interactive compute.
- Restrict runtimes to supported LTS releases.
- Require cost-allocation tags such as `CostCenter`, `Project`, and `Owner`.
- Use job compute for scheduled workloads instead of persistent all-purpose compute.
- Review SQL warehouse size, scaling range, and auto-stop separately; compute policies do not govern SQL warehouses or serverless compute.

## Example Policy Definition

This illustrative policy limits newly created or edited clusters. Tune the limits to workload requirements before deploying it.

```json
{
  "dbus_per_hour": {
    "type": "range",
    "maxValue": 40
  },
  "spark_version": {
    "type": "allowlist",
    "values": ["auto:latest-lts"],
    "defaultValue": "auto:latest-lts"
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 10,
    "maxValue": 30,
    "defaultValue": 15
  },
  "autoscale.min_workers": {
    "type": "fixed",
    "value": 1,
    "hidden": true
  },
  "autoscale.max_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 8,
    "defaultValue": 4
  },
  "custom_tags.CostCenter": {
    "type": "regex",
    "pattern": ".+"
  },
  "custom_tags.Owner": {
    "type": "regex",
    "pattern": ".+"
  }
}
```

Policies can be managed as DAB `cluster_policies` resources, through the Databricks CLI/API, or in the workspace UI. Assign policy permissions separately so users can select the policy but cannot edit it.

## What Policies Do Not Do

- They do not terminate running compute when daily spend crosses the alert threshold.
- They do not guarantee a monthly invoice amount.
- They do not govern SQL warehouses, serverless jobs, model serving, apps, or every other serverless product.
- They do not replace budgets, billing system tables, usage dashboards, or SQL Alerts.

Use layered controls: compute policies to constrain creation, tags and budget policies for attribution, auto-stop settings to reduce idle cost, and this bundle for daily notification and contributor visibility.
