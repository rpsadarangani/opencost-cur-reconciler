# opencost-cur-reconciler

Reconcile [OpenCost](https://opencost.io) Kubernetes cost data against your **actual AWS bill**.

OpenCost prices nodes using the AWS list pricing API (optionally spot data feed). It does
**not** know about your Savings Plans, Reserved Instances, or EDP discounts — workload
costs can be 2–3× higher than what you actually pay. Cost-to-bill reconciliation is a paid
Kubecost feature.

This project closes that gap with a tiny CronJob:

```
CronJob (every 6h)
   │  AssumeRole (optional, cross-account)
   ▼
Athena query over your CUR 2.0 table
   per EC2 instance, per hour:
     SavingsPlanCoveredUsage → savings_plan_effective_cost
     DiscountedUsage (RI)    → reservation_effective_cost
     Spot / On-Demand        → unblended_cost
   ▼
push to VictoriaMetrics (Prometheus import API)
   aws_node_effective_hourly_cost{provider_id, cluster, account, instance_type, purchase_option}
```

Join against OpenCost's `node_total_hourly_cost` (which carries `provider_id`) in Grafana
and your dashboards show **invoice-exact** node costs — SP, RI, EDP and spot all included.

## Requirements

- CUR 2.0 (or legacy CUR) queryable via Athena, with **resource IDs enabled**
- VictoriaMetrics (single or cluster) — anything serving `/api/v1/import/prometheus`.
  Vanilla Prometheus lacks a backfill import API.
- IRSA (IAM Roles for Service Accounts) on EKS, with Athena/Glue/S3 read access to your
  CUR — directly or via a cross-account `AssumeRole` hop (`curRoleArn`)
- OpenCost deployed per cluster shipping metrics to the same VictoriaMetrics

## Install

```sh
helm install opencost-cur-reconciler ./chart/opencost-cur-reconciler \
  -f my-values.yaml -n opencost-system
```

See [`examples/values-example.yaml`](examples/values-example.yaml). Minimal config:

```yaml
athena:
  region: us-east-1
  workgroup: primary
  database: athenacurcfn_mycur     # or your CUR 2.0 data-export database
  table: mycur
accountClusterMap: "111111111111=prod-cluster,222222222222=dev-cluster"
vmImportUrl: http://vminsert.monitoring.svc:8480/insert/0/prometheus/api/v1/import/prometheus
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111111111111:role/cur-reconciler
# optional second hop if Athena lives in another account:
# curRoleArn: arn:aws:iam::333333333333:role/cur-read
```

## Example Grafana queries

Actual cost per cluster (reconciled):

```promql
sum by (cluster) (aws_node_effective_hourly_cost)
```

Estimate-vs-actual drift (how wrong list pricing is for you):

```promql
sum by (cluster) (label_replace(node_total_hourly_cost, "iid", "$1", "provider_id", ".*/(i-.*)"))
/ on (cluster) sum by (cluster) (aws_node_effective_hourly_cost)
```

Reconciled namespace cost (allocation share × actual node cost):

```promql
sum by (namespace) (
  container_cpu_allocation * on (cluster, node) group_left()
  (avg by (cluster, node) (node_cpu_hourly_cost))
) * on (cluster) group_left()
( sum by (cluster) (aws_node_effective_hourly_cost)
/ sum by (cluster) (avg by (cluster,node) (node_total_hourly_cost)) )
```

## Notes

- CUR data lags ~12–24h; the job re-pushes a sliding `lookbackDays` window (idempotent —
  VictoriaMetrics dedups identical samples).
- Only EC2 instance costs (`BoxUsage`/`SpotUsage`) are reconciled. EBS, network, RDS etc.
  remain visible through OpenCost Cloud Costs.
- The CronJob image is the stock `aws-cli` image — no custom image to build or trust.

## License

Apache-2.0
