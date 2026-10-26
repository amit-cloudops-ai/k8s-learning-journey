# Kubernetes Learning Journey

Hands-on Kubernetes fundamentals, one project per folder, building from "bare Pod"
up to a fully autoscaling, self-healing application. Each lab is small on
purpose — the goal is to understand why each object exists, not just memorize YAML.

## Cluster setup

Everything here runs in a dedicated, isolated **kind** cluster called
`k8s-basics` — kept separate from any other clusters on the machine so nothing collides.

```bash
kind create cluster --name k8s-basics --config kind-config.yaml
kubectl config use-context kind-k8s-basics
kubectl create namespace basics
kubectl config set-context --current --namespace=basics
```

`kind-config.yaml` maps container port 30080 out to the host — useful for reaching NodePort services from `localhost`.

Always confirm the active context before running commands:

```bash
kubectl config current-context   # should say kind-k8s-basics
kubectl get pods                 # should be scoped to the basics namespace
```

## Labs

| # | Lab | Concept | Status |
|---|---|---|---|
| 01 | [Pod → Deployment → Service](./01-pod-deployment-service) | Workloads, replicas, stable service networking | Available |
| 02 | [ConfigMaps + Secrets](./02-configmaps-secrets) | Externalized configuration and sensitive values | Available |
| 03 | [Probes](./03-probes) | Liveness, readiness, and startup checks | Available |
| 04 | [Multi-container Pod](./04-multi-container-pod) | Sidecar pattern and shared volumes | Available |
| 05 | [StatefulSet + PVC](./05-statefulset-pvc) | Stateful workloads and persistent storage | Available |
| 06 | [ResourceQuota + LimitRange](./06-resourcequota-limitrange) | Resource governance and limits | Available |
| 07 | RBAC | ServiceAccounts, Roles, and RoleBindings | Planned |
| 08 | Ingress | Host/path routing and ingress controllers | Planned |
| 09 | HPA | Scaling workloads under load | Planned |
| 10 | Rolling updates + rollback | Safer releases and recovery | Planned |

Only labs marked **Available** have folders in this repository. Planned labs will be added as they are implemented.

## Suggested workflow

1. Read the lab's README before applying manifests.
2. Check the current Kubernetes context and namespace.
3. Apply the lab resources and verify the expected behavior with `kubectl`.
4. Inspect events and logs, then clean up the lab resources before moving on.

## Cleanup

When finished with the learning environment, remove the dedicated cluster:

```bash
kind delete cluster --name k8s-basics
```

Progress notes and lab-specific commands belong in each lab's own README.
