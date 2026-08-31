# 01 — Pod → Deployment → Service

**Goal:** feel the difference between the three most basic K8s objects, in order,
instead of just reciting definitions.

## Why this order matters

- A **Pod** is the smallest deployable unit — one or more containers that share
  network/storage. But a bare Pod has no self-healing: kill it, it's gone.
- A **Deployment** wraps Pods in a ReplicaSet and adds self-healing, scaling,
  and rolling updates. This is what you'd actually run in production — never
  bare Pods.
- A **Service** gives that (unstable, replaceable) set of Pods a stable network
  identity. Pods die and get new IPs constantly; the Service IP/DNS name never
  changes.

You're building understanding in layers: object → what it's missing → next
object that fixes it.

## Prerequisites

- Docker Desktop with Kubernetes enabled (Settings → Kubernetes → Enable) —
  or minikube/kind if you prefer, commands below don't change.
- `kubectl` on your PATH — check with `kubectl version --client`.

## Step 1 — Bare Pod

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
```

Now kill it and watch what happens:

```bash
kubectl delete pod nginx-pod
kubectl get pods
```

**Notice:** it's just gone. Nothing brings it back. This is the exact gap
Deployments exist to solve — write that observation in your own notes, it's
the core lesson of this step.

Port-forward to confirm it actually serves traffic before you delete it again:

```bash
kubectl apply -f nginx-pod.yaml
kubectl port-forward pod/nginx-pod 8080:80
# visit http://localhost:8080, then Ctrl+C
```

## Step 2 — Deployment

```bash
kubectl delete pod nginx-pod
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods -l app=nginx
kubectl get replicasets
```

Now break one on purpose:

```bash
kubectl delete pod <one-of-the-nginx-pod-names>
kubectl get pods -l app=nginx
```

**Notice:** a replacement Pod appears within seconds, with a new name. The
Deployment's ReplicaSet noticed `replicas: 3` wasn't satisfied and fixed it.
This is self-healing — the thing a bare Pod could never do.

Try scaling live:

```bash
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx
```

## Step 3 — Service

Three Pods, but each has its own changing IP — nothing else in the cluster can
reliably talk to "nginx" without a stable address. That's what a Service is for.

```bash
kubectl apply -f nginx-service-clusterip.yaml
kubectl get svc nginx-service-clusterip
kubectl get endpoints nginx-service-clusterip
```

**Notice:** the Endpoints object lists all 5 Pod IPs. The Service is just a
label selector + load balancer in front of them — delete/recreate a Pod and
the Endpoints list updates automatically, but the Service IP never changes.

Test it from inside the cluster (ClusterIP isn't reachable from your laptop
directly):

```bash
kubectl run tmp-shell --rm -it --image=busybox -- /bin/sh
# inside the shell:
wget -qO- http://nginx-service-clusterip
exit
```

Now expose it to your laptop with NodePort instead:

```bash
kubectl apply -f nginx-service-nodeport.yaml
curl http://localhost:30080
```

## Step 4 — Clean up

```bash
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service-clusterip.yaml
kubectl delete -f nginx-service-nodeport.yaml
```

## What you should be able to explain after this

- Why bare Pods are never used directly in production
- What a ReplicaSet actually does under a Deployment
- Why Services exist even though Pods already have IPs
- The difference between ClusterIP (internal only) and NodePort (externally
  reachable via a fixed port on the node)

## Status

- [x] Ran through all 4 steps
- [x] Killed a Pod under a Deployment and watched it self-heal
- [x] Confirmed Service traffic survives Pod replacement
- [x] Notes written in own words

## What actually happened (real run notes)

- Bare `nginx-pod` deleted → nothing replaced it. Confirmed.
- Deployment pod deleted → ReplicaSet spun up a replacement within seconds,
  new pod name, same 3/3 desired count maintained.
- `kubectl get endpoints nginx-service-clusterip` showed all 5 live pod IPs —
  the Endpoints object updates itself automatically whenever pods matching
  the Service's label selector change.
- ClusterIP is only reachable *inside* the cluster — tested from a temporary
  `busybox` pod (`kubectl run tmp-shell --rm -it --image=busybox -- /bin/sh`),
  not from the WSL host directly.
- NodePort needed `kind-config.yaml` with `extraPortMappings` to actually
  reach `localhost:30080` from outside — a plain `kind create cluster` does
  not expose NodePort to the host machine the way Docker Desktop's built-in
  cluster does. Recreated the cluster with the port mapping to fix this.
