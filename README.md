# warden-config

Kustomize base and Argo CD application for [warden](https://github.com/benzac708/warden).

- `base/` — Deployment (two containers, shared emptyDir), Services, ServiceMonitor, ConfigMap, RBAC, NetworkPolicy
- `app.yaml` — Argo CD Application pointing at `base/`
- `cloudflared/ingress-snippet.txt` — rule to add to `platform-config`

The public page is served by the `warden-showcase` container only. The `warden-agent` container is internal and holds the LLM key. Neither container can write to the cluster.

## Prerequisites

Warden queries Loki for cluster logs. Loki is **not** part of kube-prometheus-stack.
The `loki`, `promtail`, and `loki-datasource` Argo CD Applications in `loki/`
install and configure it — see [`loki/README.md`](loki/README.md). Without Loki,
every warden report has confidence 0.0 and a DNS error.

The `warden` Argo CD Application is owned by the App-of-Apps in
`platform-config/bootstrap/root-app.yaml`. Do not apply an Application from this
repo; edit `platform-config/apps/warden.yaml` instead.

## LLM key

The LLM key is a `SealedSecret` in `base/sealed-secrets.yaml`. That is the only
supported source of truth. Rotate it with `kubeseal` and an Argo CD sync; do not
create `warden-llm` with `kubectl create secret`, because the two sources of
truth conflict under `prune: true` and `selfHeal: true`.

## Deploy (run on the VPS over Tailscale)

The tunnel route for `warden.zachara.dev` is already in
`platform-config/manifests/cloudflared/config.yaml` — do not add it again.

```sh
kubectl -n apps rollout status deploy/warden
kubectl -n apps get pods -l app=warden
curl -fsS https://warden.zachara.dev/healthz
curl -fsS https://warden.zachara.dev/report.json | head
```

The `warden` repo needs a `CONFIG_REPO_TOKEN` secret (an SSH key with write
access to this repo) before its CI can bump the image tag. Both containers are
pinned to a commit SHA, so a missing token leaves the last working image in place.

## Resource footprint

One pod, two containers, requests 75m CPU / 96Mi total, limits 300m CPU / 384Mi total. Fits the existing 3-node cluster; no new nodes and no PVC are needed.
