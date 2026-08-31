# Kubernetes Learning Journey

Hands-on Kubernetes fundamentals, one project per folder, building from "bare Pod"
up to a fully autoscaling, self-healing application. Each project is small on
purpose — the goal is to *feel* why each object exists, not just memorize YAML.

## Cluster setup

Everything here runs in a dedicated, isolated **kind** cluster called
`k8s-basics` — kept separate from any other clusters on the machine (demo
apps, other tools) so nothing collides.

```bash
kind create cluster --name k8s-basics --config kind-config.yaml
kubectl config use-context kind-k8s-basics
kubectl create namespace basics
kubectl config set-context --current --namespace=basics
```

`kind-config.yaml` maps container port 30080 out to the host — without this,
`kind` clusters don't expose NodePort services to `localhost` by default
(unlike Docker Desktop's built-in cluster, which does this automatically).
This tripped me up in project 01 — worth remembering for Ingress later.

Always confirm the right context before running anything:

```bash
kubectl config current-context   # should say kind-k8s-basics
kubectl get pods                 # should be scoped to the basics namespace
```

## Projects

| # | Project | Concept |
|---|---------|---------|
| 01 | [Pod → Deployment → Service](./01-pod-deployment-service) | Why each object exists, layered on top of each other |
| 02 | ConfigMaps + Secrets | Externalizing config, env vs mounted volume |
| 03 | Probes | Liveness / Readiness / Startup |
| 04 | Multi-container Pod | Sidecar pattern, shared volume |
| 05 | StatefulSet + PVC | Stateful workloads, storage persistence |
| 06 | ResourceQuota + LimitRange | Resource management, OOMKill |
| 07 | RBAC | ServiceAccount, Role, RoleBinding |
| 08 | Ingress | Path/host-based routing |
| 09 | HPA | Autoscaling under real load |
| 10 | Rolling update + rollback | Safe deploys, incident recovery |

Progress log kept per-project in that project's own README.
