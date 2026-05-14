# k3s Migration Design

**Date:** 2026-05-15
**Scope:** Migrate all Docker Swarm services to k3s

---

## Context

Current stack runs on Docker Swarm with Traefik v3 as ingress, Cloudflare DNS challenge for TLS, and three app groups (voice-agent, connect, webhook). The cluster has mixed-architecture nodes (ARM and AMD).

**Target cluster topology:**
- 1 AMD VPS — k3s server (control plane), runs AMD workloads
- 1 ARM VPS — k3s agent, runs ARM workloads
- 1 AMD VPS — future k3s agent (not yet provisioned)

---

## What's Dropped

- **Portainer** — dropped in favour of plain `kubectl`
- Host-mounted `acme.json` — replaced by Traefik PVC
- Docker socket mount on Traefik — not needed; k3s Traefik uses k8s API
- Overlay networks — replaced by native k8s cluster networking

---

## Directory Structure

```
k3s/
  traefik/
    helmchartconfig.yaml     # patches k3s built-in Traefik
    cloudflare-secret.yaml   # CF_DNS_API_TOKEN secret
  apps/
    voice-agent/
      namespace.yaml
      secret.yaml
      deployment.yaml
      service.yaml
      ingressroute.yaml
    connect/
      namespace.yaml
      secret.yaml
      deployment.yaml
      service.yaml
      ingressroute.yaml
    webhook/
      namespace.yaml
      deployment-webhook.yaml
      deployment-redis.yaml
      deployment-echo-server.yaml
      service-redis.yaml
      service-echo-server.yaml
      ingressroute.yaml
```

---

## Section 1: Namespaces & Node Labels

Each app lives in its own namespace: `voice-agent`, `connect`, `webhook`. Traefik remains in `kube-system`.

Node labels set once at cluster join:
```bash
kubectl label node <amd-node>  arch=amd64
kubectl label node <arm-node>  arch=arm64
```

Deployments use `nodeSelector` to replace swarm placement constraints:
```yaml
nodeSelector:
  arch: arm64   # or amd64
```

---

## Section 2: Traefik Configuration

k3s ships Traefik as a Helm release. Configuration is patched via `HelmChartConfig` CRD in `kube-system` — no separate Traefik install.

**`traefik/cloudflare-secret.yaml`**
- Kind: `Secret`, namespace: `kube-system`
- Key: `CF_DNS_API_TOKEN`

**`traefik/helmchartconfig.yaml`** configures:
- ACME cert resolver `tlsResolver` with Cloudflare DNS challenge
  - email: `info@himanshu-gunwant.com`
  - resolvers: `1.1.1.1:53`, `1.0.0.1:53`
- `CF_DNS_API_TOKEN` injected via `envFrom` from cloudflare-secret
- HTTP→HTTPS redirect on `web` entrypoint
- Dashboard at `traefik.himanshu-gunwant.com` with TLS (`tlsResolver`)
- Persistence enabled for `acme.json` (PVC, 128Mi)
- `insecureSkipVerify: true` on serversTransport
- `providers.kubernetescrd.namespaces: []` — watch all namespaces

---

## Section 3: App Deployments

### voice-agent (namespace: `voice-agent`)
- Image: `hgunwant2312/voice-agent:latest`
- `nodeSelector: { arch: arm64 }`
- `envFrom` → Secret `voice-agent-env`
- 1 replica, `restartPolicy: Always`
- ClusterIP Service on port 3000

### connect (namespace: `connect`)
- Image: `hgunwant2312/zenodeck-connect:latest`
- `nodeSelector: { arch: arm64 }`
- `envFrom` → Secret `connect-env`
- 1 replica, `restartPolicy: Always`
- ClusterIP Service on port 3000

### webhook (namespace: `webhook`)
Three separate Deployments, all with `nodeSelector: { arch: amd64 }`:

| Deployment | Image | Port |
|---|---|---|
| webhook | `webhooksite/webhook.site` | 80 |
| redis | `redis:alpine` | 6379 |
| laravel-echo-server | `webhooksite/laravel-echo-server` | 6001 |

- `webhook` and `laravel-echo-server` env vars inline (no sensitive values)
- Inter-service DNS: `webhook_redis` → `redis` (within same namespace)
- `laravel-echo-server` not publicly exposed (ClusterIP only)

---

## Section 4: Ingress Routing

Using Traefik `IngressRoute` CRD in each app's namespace.

| App | Rule | TLS | Backend Port |
|---|---|---|---|
| voice-agent | `PathPrefix(\`/api/v1/voice-agent\`)` | `tlsResolver` | 3000 |
| connect | `PathPrefix(\`/api/v1/connect\`)` | `tlsResolver` | 3000 |
| webhook | `Host(\`webhook.himanshu-gunwant.com\`)` | `tlsResolver` | 80 |
| traefik dashboard | `Host(\`traefik.himanshu-gunwant.com\`)` | `tlsResolver` | built-in |

Traefik watches all namespaces for `IngressRoute` resources via `providers.kubernetescrd.namespaces: []`.

---

## Secrets Strategy

Plain Kubernetes Secrets (approach A). Secrets are applied manually to the cluster and **not committed to git**. The git repo contains placeholder/example secret files only.

```bash
# Example — apply secrets manually
kubectl apply -f k3s/traefik/cloudflare-secret.yaml
kubectl apply -f k3s/apps/voice-agent/secret.yaml
kubectl apply -f k3s/apps/connect/secret.yaml
```

---

## Apply Order

1. `k3s/traefik/` — cloudflare-secret, then helmchartconfig
2. `k3s/apps/voice-agent/` — namespace → secret → deployment → service → ingressroute
3. `k3s/apps/connect/` — same order
4. `k3s/apps/webhook/` — namespace → deployments → services → ingressroute
