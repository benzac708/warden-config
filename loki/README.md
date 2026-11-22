# Loki + Promtail (Helm)

Adds log aggregation to the k3d cluster so `warden` (and other apps) can query logs.

- Loki: `grafana/loki` chart 7.6.0, SingleBinary mode, filesystem storage on a 10Gi local-path PVC, 7-day retention.
- Promtail: `grafana/promtail` chart 6.17.1, DaemonSet shipping container logs to Loki.
- Namespace: `monitoring` (matches `warden`'s default `WARDEN_LOKI_URL=http://loki.monitoring.svc.cluster.local:3100`).

## Deploy

### Via Argo CD (recommended, reproducible)

```sh
kubectl apply -f loki/app-loki.yaml
kubectl apply -f loki/app-promtail.yaml
kubectl apply -f loki/app-datasource.yaml
# Argo CD syncs all three. Watch:
kubectl -n argocd get applications loki promtail loki-datasource
```

### Via Helm (manual)

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki -n monitoring -f loki/loki-values.yaml
helm install promtail grafana/promtail -n monitoring -f loki/promtail-values.yaml
```

## Verify

```sh
kubectl -n monitoring get pods -l app.kubernetes.io/name=loki
kubectl -n monitoring get pods -l app.kubernetes.io/name=promtail
# Push a test log line and query:
curl -s "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/query?query=%7Bnamespace%3D%22apps%22%7D" | head
```

## Grafana datasource

The datasource is managed by `loki/app-datasource.yaml` from
`loki/datasource/grafana-datasource.yaml`. The kube-prometheus-stack sidecar
loads the labeled ConfigMap. Do not add it by hand in the Grafana UI.