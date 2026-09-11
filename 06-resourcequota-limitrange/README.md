# Project 06 — ResourceQuota + LimitRange

Demonstrates how Kubernetes prevents any single container from consuming
unlimited CPU/memory, and how it caps total resource usage across an entire
namespace.

![Uploading cpu_request_vs_limit.png…]()

## 1. The problem this solves

Nothing built in earlier projects stopped a Pod from using as much CPU or
memory as it wanted. On a shared cluster (many teams, one set of servers),
one careless or buggy Pod could consume all available resources and starve
every other Pod on the same node. This project adds two different layers of
guardrails:

- **LimitRange** — rules applied to each individual container: default
  request/limit values if none are given, and a maximum any container is
  allowed to ask for.
- **ResourceQuota** — a rule for an entire namespace as a whole: total
  CPU/memory usage summed across every Pod in the namespace cannot exceed
  a fixed cap, and the number of Pods is capped too.

Analogy: LimitRange is a per-employee spending limit. ResourceQuota is the
whole department's budget cap. Different scope, same goal — stop usage from
running away unchecked.

## 2. Request vs limit, explained from scratch

Every container can set two separate numbers for CPU and memory:

- **request** = the guaranteed minimum reserved for this container. The
  scheduler uses this number to decide which node has room for the Pod.
- **limit** = the absolute maximum the container is allowed to use. Going
  over the CPU limit gets the container throttled (slowed down, not
  killed). Going over the memory limit gets the container killed
  (`OOMKilled`).

A container can burst above its request, using spare capacity, but it can
never cross its limit.

## 3. Files in this project

| File | Purpose |
|---|---|
| `limitrange.yaml` | Per-container default request/limit + max ceiling |
| `resourcequota.yaml` | Whole-namespace cap on total CPU/memory/pod count |
| `no-resources-pod.yaml` | A Pod with no resources set, to prove LimitRange auto-fills defaults |
| `oversized-pod.yaml` | A Pod that deliberately exceeds the LimitRange max, to prove it gets rejected |

All resources in this project run in a dedicated namespace, `resource-demo`,
created with:

```bash
kubectl create namespace resource-demo
```

This keeps the quota/limit rules isolated from every other project in this
repo.

## 4. Step-by-step

Apply the LimitRange and ResourceQuota:

```bash
kubectl apply -f limitrange.yaml
kubectl apply -f resourcequota.yaml
```

Check what was set:

```bash
kubectl describe limitrange container-limits -n resource-demo
kubectl describe resourcequota namespace-quota -n resource-demo
```

Create a Pod with **no** `resources:` block at all:

```bash
kubectl apply -f no-resources-pod.yaml
```

Inspect it — despite specifying nothing, it was assigned resource values:

```bash
kubectl get pod no-resources-pod -n resource-demo -o yaml | grep -A 6 resources:
```

Expected: `requests: cpu: 100m, memory: 64Mi` and `limits: cpu: 200m, memory: 128Mi`
— the exact defaults from the LimitRange, injected automatically.

Recheck the ResourceQuota — it now shows real usage:

```bash
kubectl describe resourcequota namespace-quota -n resource-demo
```

`pods` should read `1/5`, with CPU/memory `Used` matching the Pod's assigned values.

## 5. Proving the max ceiling is enforced

Try to create a container that asks for more than the LimitRange's `max`
(`500m` CPU / `256Mi` memory):

```bash
kubectl apply -f oversized-pod.yaml
```

Expected result — an immediate rejection, not a Pod that gets created and
fails later:

```
Error from server (Forbidden): error when creating "oversized-pod.yaml":
pods "oversized-pod" is forbidden: [maximum cpu usage per Container is 500m,
but limit is 700m, maximum memory usage per Container is 256Mi, but limit
is 400Mi]
```

## 6. Why this matters — admission control

This rejection happens at **admission time**, before the Pod is ever stored
in the cluster or scheduled to a node. When you run `kubectl apply`, the
request first passes through an admission controller that checks it against
rules like the LimitRange. A violation gets bounced immediately with a
`Forbidden` error — no `Pending` state, no failed scheduling attempt, no
Pod object left behind.

This is different from something like a probe misconfiguration
(Project 03), where the Pod gets created successfully and only fails later,
once it's actually running. LimitRange/ResourceQuota violations are
rejected at the front door — this is what makes them an active enforcement
mechanism, not just a monitoring tool that reports bad behavior after the
fact.
