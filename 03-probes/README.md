# 03 — Probes (Liveness / Readiness / Startup)

**Goal:** teach Kubernetes to actually check whether your app is working,
instead of assuming "process started" means "app is healthy."

## Why this matters — the three probes, plainly

- **startupProbe** — "has the app finished starting up yet?" Used for apps
  that take a while to boot. While this probe hasn't succeeded, Kubernetes
  holds off on running liveness/readiness checks, so a slow-starting app
  doesn't get killed for "failing" a check it hasn't had time to pass yet.
- **readinessProbe** — "is the app ready to receive traffic right now?" A Pod
  that fails this check isn't restarted — it's just pulled out of the
  Service's Endpoints list (project 01) until it passes again. This is the
  probe that connects directly back to what you learned about Endpoints.
- **livenessProbe** — "is the app still working, or has it silently broken?"
  A Pod that fails this repeatedly gets restarted. This is the one that
  actually kills and recreates the container.

The three exist because "started" and "healthy" and "ready for traffic" are
three genuinely different questions, and real apps can be in different
states for each at different times.

## Step 1 — startup + readiness on a slow-starting app

`probe-demo-pod.yaml` deliberately sleeps 20 seconds before nginx actually
starts — simulating a real app with a slow boot (loading data, connecting to
a database, warming a cache).

```bash
kubectl apply -f probe-demo-pod.yaml
kubectl get pod probe-demo-pod --watch
```

Watch the `READY` column. For the first ~20 seconds it'll show `0/1` even
though the Pod's `STATUS` says `Running` — that's the difference between
"container process exists" and "passed its readiness check." Once nginx
actually starts responding, it flips to `1/1`. Ctrl+C to stop watching.

Confirm what actually happened:

```bash
kubectl describe pod probe-demo-pod
```

Look at the **Events** section at the bottom — you'll see the startup probe
retrying every 3 seconds until it succeeds, and only then readiness/liveness
checks beginning.

**Why this matters for real apps:** without a startupProbe, a slow-booting
app can get killed by liveness checks before it ever finishes starting —
Kubernetes would misread "still booting" as "broken" and restart it in a
loop, and it would never actually come up. This is a real, common
misconfiguration in production.

## Step 2 — force a liveness failure on purpose

`probe-broken-pod.yaml` points its liveness probe at a path that doesn't
exist (`/this-path-does-not-exist` — nginx returns a 404 there, which counts
as a failed check).

```bash
kubectl apply -f probe-broken-pod.yaml
kubectl get pod probe-broken-pod --watch
```

Watch the `RESTARTS` column. After 3 failed checks (15 seconds, since
`periodSeconds: 5`), Kubernetes will restart the container. Watch it happen
live, then Ctrl+C.

Confirm it in the events:

```bash
kubectl describe pod probe-broken-pod
```

Look for `Liveness probe failed` followed by `Killing` and `Started` in the
Events — that's Kubernetes catching a "broken but still running" app and
fixing it automatically, the same self-healing instinct from project 01, but
triggered by a health check instead of the Pod being deleted outright.

## Step 3 — connect readiness back to Services

This is the part that ties directly into project 01. Apply a readiness-only
version behind a Service and watch a failing readiness check pull the Pod
out of rotation without killing it:

```bash
kubectl expose pod probe-demo-pod --port=80 --name=probe-demo-svc
kubectl get endpoints probe-demo-svc
```

While the Pod is still in its 20-second startup sleep, check the Endpoints —
it'll be empty, because a Pod that hasn't passed its readiness probe never
gets added to a Service's Endpoints list in the first place. Once it passes,
re-run `kubectl get endpoints probe-demo-svc` and you'll see the Pod's IP
appear.

## Step 4 — clean up

```bash
kubectl delete -f probe-demo-pod.yaml
kubectl delete -f probe-broken-pod.yaml
kubectl delete svc probe-demo-svc
```

## What you should be able to explain after this

- The difference between "container running" and "container ready"
- Why startupProbe exists separately from livenessProbe
- What happens on a readiness failure (removed from Endpoints) vs a
  liveness failure (container restarted)
- Why a missing startupProbe can cause a restart loop on slow-booting apps

## Status

- [ ] Watched a Pod stay `0/1` ready during its startup sleep, then flip to `1/1`
- [ ] Watched a broken liveness probe trigger an actual container restart
- [ ] Confirmed a not-yet-ready Pod is absent from a Service's Endpoints
- [ ] Notes written in own words
