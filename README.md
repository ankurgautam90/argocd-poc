# argocd-poc — GitOps stack (app-of-apps)

A portable GitOps repo. Point **any ArgoCD** at it and apply the root app — it deploys the
whole stack to **whatever cluster that ArgoCD runs in** (every app targets the in-cluster
API `https://kubernetes.default.svc`, so nothing is tied to one cluster).

## Deploy to any cluster (one command)

1. Have a Kubernetes cluster with **ArgoCD installed**, a **default StorageClass**, and a
   **LoadBalancer** provider (for Kong / Rancher).
2. Bootstrap — the only manual step, ever:
   ```bash
   kubectl apply -f bootstrap/root-app.yaml
   ```
3. Done. `root` (app-of-apps) watches `apps/` and deploys everything in wave order.
   From then on: **edit a file + `git push`** and ArgoCD reconciles.

## Layout

```
bootstrap/root-app.yaml     # app-of-apps — apply once
apps/<name>.yaml            # one ArgoCD Application per component
workloads/<name>/           # plain manifests / CRs the Applications point at
```

## What deploys (and in what order)

| Wave | Components |
|------|-----------|
| 0 | cert-manager, ECK operator, cass-operator, Gateway API CRDs, Argo Rollouts |
| 1 | Kong (ingress/gateway), Rancher (+ Fleet) |
| 2 | Cassandra (`CassandraDatacenter`), Elasticsearch (`Elasticsearch`) |
| 3 | Temporal (server + web, wired to the Cassandra + Elasticsearch above) |
| 4 | Demos: `rollout-demo`, `mp-demo`, `cross-cluster-demo` |

Datastores are single-node with 50 Gi PVCs (adjust for production).

## Demos & workloads (wave 4)

These are the capability demos, each a self-contained ArgoCD Application over `workloads/<name>/`.

### `rollout-demo` — progressive delivery (canary)
`workloads/rollout-demo/` — an Argo Rollouts **canary** driven through the **Kong Gateway (Gateway API)**.
- `demo-gw` Rollout (blue/green image), `demo-stable` + `demo-canary` Services, `demo-route` HTTPRoute, the Kong `Gateway`/`GatewayClass`.
- Argo Rollouts writes the traffic split (20→50→100) into the HTTPRoute; **Kong enforces the exact %**.
- The App `ignoreDifferences` the HTTPRoute weights + the Services' `rollouts-pod-template-hash` selector (those are runtime-owned by the rollout controller — don't let ArgoCD fight it).
- Host `canary.<KONG_IP>.sslip.io` · dashboard `rollouts.<KONG_IP>.sslip.io`.

### `mp-demo` — one LB IP, four protocols
`workloads/mp-demo/` — proves **HTTP + HTTPS + gRPC + WSS on the single Kong LB IP** (ports 80 + 443 only).
- `echo` (whoami) → HTTP/HTTPS · `ws-test` → WSS · `grpcbin` → gRPC-over-TLS.
- Hosts: `echo.` / `ws.` / `grpc.<KONG_IP>.sslip.io`. Kong demuxes by SNI + Host + protocol.

### `cross-cluster-demo` — Submariner cross-cluster networking
`workloads/cross-cluster-demo/` — services on one cluster reachable from the other by DNS, over an IPsec tunnel.
- **`cluster-b/`** (managed by this ArgoCD): `nginx` + `grpcbin` (exported via `ServiceExport`) + `demo-client`.
- **`cluster-a/`** (applied to the *other* cluster — separate ArgoCD or `kubectl apply`): `whoami` (exported) + `connectivity-checker` + `demo-client`.
- Any pod reaches a remote service at **`<svc>.cross-cluster-demo.svc.clusterset.local`** (Lighthouse MCS DNS).

## Cross-cluster (Submariner) — installed out-of-band

Submariner itself (operator + broker + gateway) is **not** GitOps-managed here — it's installed with
`subctl` because the broker-join uses a one-time token/secret. One cluster hosts the broker; both join.

Hard requirements for the tunnel + no-NAT routing:
- **Non-overlapping pod & service CIDRs** across the two clusters (else use Submariner Globalnet + NAT).
- Each gateway node needs a **routable IP** (floating IP), the Neutron **port-security** relaxed or
  `allowed-address-pairs` set for the remote CIDRs, and SG rules for **UDP 500 + 4500 + ESP**.

```bash
subctl deploy-broker --kubeconfig cluster-b.kubeconfig
subctl join --kubeconfig cluster-b.kubeconfig broker-info.subm --clusterid cluster-b --cable-driver libreswan
subctl join --kubeconfig cluster-a.kubeconfig broker-info.subm --clusterid cluster-a --cable-driver libreswan
subctl export service <name> -n cross-cluster-demo   # make a service clusterset-reachable
```

## Environment-specific values to set

Everything else is generic; only these depend on your environment:

1. **Rancher hostname** — `apps/rancher.yaml` → `hostname:` → a real DNS name that resolves
   to Kong's LoadBalancer IP. Also change `bootstrapPassword`.
2. **StorageClass** — `workloads/cassandra/…` and `workloads/elasticsearch/…` use
   `storageClassName: default`. If your cluster's default SC has a different name, change it
   (or delete the line to use the cluster's default automatically) **before first deploy**
   (it's immutable afterward).
3. **repoURL** — only if you fork: update `repoURL` in `bootstrap/root-app.yaml` and every
   `apps/*.yaml` to your fork.

## Deploy to a *remote* cluster (optional)

The default targets the local cluster. To drive a remote cluster from a central ArgoCD:
register it (`argocd cluster add <context>`) and set each app's
`spec.destination.server` to that cluster's API URL (or use an ApplicationSet cluster
generator to fan out to many).
