### Tony's Rill Playground

#### GCloud BigQuery

1. Get Network Egress Cost

```sql
SELECT EXTRACT(date FROM usage_start_time) as date,
  sku.description,
  round(sum(usage.amount_in_pricing_units)) as usage_gib,
  round(SUM(cost)) as cost,
  (SELECT value FROM UNNEST(labels) WHERE key = 'goog-k8s-node-pool-name') as nodepool,
  (SELECT value FROM UNNEST(labels) WHERE key = 'environment') as env
FROM `rilldata.billing.gcp_billing_export_v1_01FFEC_72971C_1C273F`
WHERE
  -- sku.description LIKE "Netwoskurk Inter Zone Egress"
  sku.id = "DE9E-AFBC-A15A"
  AND usage_start_time > "2023-01-01"
  AND EXISTS (SELECT value FROM UNNEST(labels) WHERE key = 'service' AND value = 'gke')
GROUP BY date, sku.description, env, nodepool
ORDER BY date desc, cost desc
```
