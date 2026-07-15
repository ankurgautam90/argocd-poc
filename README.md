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

Datastores are single-node with 50 Gi PVCs (adjust for production). PostgreSQL and Temporal
are intentionally **not** included (DB is external; Temporal added separately when needed).

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
