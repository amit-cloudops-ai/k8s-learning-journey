# 04 — Multi-container Pod (Sidecar Pattern)

**Goal:** feel, hands-on, what "containers in the same Pod share" actually
means — both network and storage — and why the sidecar pattern exists.

## The setup

One Pod, two containers:

- **main-app** — runs a tiny web server (`busybox httpd`) on port 8080, and
  separately writes a timestamped log line to a shared file every 3 seconds.
- **log-sidecar** — does nothing except watch that same log file and print
  new lines as they appear (`tail -f`), simulating a real log-shipping
  sidecar that would forward these lines to a logging system.

Two volumes are shared between them: one for logs, one for the web content.

## Step 1 — deploy and watch it start

```bash
kubectl apply -f sidecar-demo-pod.yaml
kubectl get pod sidecar-demo-pod
```

**Notice the READY column shows `2/2`** — this is new. Every Pod you've made
so far showed `1/1`. This number is literally "how many containers in this
Pod, and how many are ready" — you're seeing 2 containers being tracked as
one unit for the first time.

## Step 2 — prove the shared volume works

Check each container's logs separately — `-c` picks which container inside
the Pod you mean, since there's more than one now:

```bash
kubectl logs sidecar-demo-pod -c main-app
kubectl logs sidecar-demo-pod -c log-sidecar
```

**Notice:** `log-sidecar`'s own output shows the *exact same lines*
`main-app` is writing to its log file — even though `log-sidecar` never
wrote a single one of them itself. It's just reading a file that
`main-app` is writing to, live, because they share the same volume. Neither
container needed to know about the other's existence to make this work —
they just both mounted the same storage at the same path.

## Step 3 — prove the shared network works

Exec into the sidecar container specifically, and try to reach the web
server that `main-app` is running — on `localhost`, not the Pod's IP:

```bash
kubectl exec sidecar-demo-pod -c log-sidecar -- wget -qO- http://localhost:8080
```

**Notice:** this works, and returns the HTML `main-app` wrote. Two
completely separate container processes, one reachable from the other via
`localhost` — this is only possible because they're in the same Pod, sharing
one network identity. Try this same command against any Pod from an
*earlier* project and it would fail — different Pods never share
`localhost`, only same-Pod containers do.

## Step 4 — kill one container, watch the other survive

```bash
kubectl exec sidecar-demo-pod -c main-app -- kill 1
kubectl get pod sidecar-demo-pod
```

**Notice what actually happens here** — this is worth watching closely.
Since `main-app`'s main process (PID 1 inside that container) gets killed,
Kubernetes restarts *that specific container*, not the whole Pod, and not
`log-sidecar`. Check:

```bash
kubectl get pod sidecar-demo-pod
kubectl logs sidecar-demo-pod -c log-sidecar
```

You'll see `RESTARTS` go up for the Pod overall, but the sidecar's own log
history is untouched — it kept running the whole time. Each container in a
Pod has its own independent lifecycle, even while sharing the Pod's network
and any shared volumes.

## Step 5 — clean up

```bash
kubectl delete -f sidecar-demo-pod.yaml
```

## What you should be able to explain after this

- What "containers in a Pod share" concretely means: same network
  (`localhost` reachable between them), and any volumes you explicitly
  mount into more than one container
- Why sidecars exist: separating a concern (like log shipping) from the
  main app, without deploying a whole separate Pod and losing the shared
  network/storage convenience
- That containers in the same Pod still have independent lifecycles —
  killing one doesn't kill the other

## Status

- [ ] Confirmed `2/2` READY on a running Pod
- [ ] Proved shared volume: sidecar saw lines it never wrote
- [ ] Proved shared network: sidecar reached main-app's server via `localhost`
- [ ] Killed one container and confirmed the other kept running independently
- [ ] Notes written in own words
