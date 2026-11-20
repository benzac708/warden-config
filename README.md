# warden-config

Kustomize base and Argo CD application for [warden](https://github.com/benzac708/warden).

- `base/` — Deployment (two containers, shared emptyDir), Services, ServiceMonitor, ConfigMap, RBAC, NetworkPolicy
- `app.yaml` — Argo CD Application pointing at `base/`
- `cloudflared/ingress-snippet.txt` — rule to add to `platform-config`

The public page is served by the `warden-showcase` container only. The `warden-agent` container is internal and holds the LLM key. Neither container can write to the cluster.

## Prerequisites

Warden queries Loki for cluster logs. Loki is **not** part of kube-prometheus-stack,
so install it first — see [`loki/`](loki/README.md) for the Helm/Argo CD setup.
Without Loki, every warden report has confidence 0.0 and a DNS error.

## Deploy (run on the VPS over Tailscale)

```sh
# 1. Create the LLM key secret (one time). Use a key with a free model quota.
read -rs -p 'LLM API key: ' KEY; echo
kubectl -n apps create secret generic warden-llm \
  --from-literal=api-key="$KEY" --dry-run=client -o yaml | kubectl apply -f -
unset KEY

# 2. Register the Argo CD Application (one time).
kubectl apply -f app.yaml

# 3. Add the tunnel hostname (see cloudflared/ingress-snippet.txt) and the DNS
#    record in the Cloudflare dashboard, then let Argo CD sync platform-config.

# 4. Verify.
kubectl -n apps rollout status deploy/warden
kubectl -n apps get pods -l app=warden
curl -fsS https://warden.zachara.dev/healthz
curl -fsS https://warden.zachara.dev/report.json | head
```

Prerequisite: the `warden` repo needs a `CONFIG_REPO_TOKEN` secret (an SSH key with write access to this repo) before its CI can bump the image tag. The initial manifest uses the `:v1` tag, so the pod still runs if that secret is missing.

## Resource footprint

One pod, two containers, requests 75m CPU / 96Mi total, limits 300m CPU / 384Mi total. Fits the existing 3-node cluster; no new nodes and no PVC are needed.
