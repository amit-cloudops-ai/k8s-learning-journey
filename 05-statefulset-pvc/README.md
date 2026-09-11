# Project 05 — StatefulSet + Persistent Volume Claims (PVC)

This project teaches how Kubernetes gives certain Pods a **stable identity** and
**permanent storage that survives Pod deletion** — something a plain Deployment
cannot do.

If you're new to Kubernetes: read this top to bottom before running anything.
Every command below is explained, not just listed.

---

## 1. The problem this solves

In earlier projects, a **Deployment** was used. Deployment Pods are treated as
disposable and identical:

- If a Pod dies, it's replaced by a **new Pod with a new random name**
  (e.g. `nginx-7c977bd7bb-4z4s5` becomes `nginx-7c977bd7bb-9kx2p`).
- Any data that Pod had (using `emptyDir`, see Project 04) is **gone forever**
  the moment the Pod dies.

This is perfectly fine for stateless apps (a web server that doesn't remember
anything between requests). It is **not fine** for something like a database,
where losing the Pod must NOT mean losing the data, and where "which specific
instance am I talking to" can matter (e.g. only one instance may be the
primary that accepts writes).

**StatefulSet** exists to solve exactly this problem. It gives every Pod:

1. A **stable, predictable name** (`web-0`, `web-1`, ...) — if `web-0` dies,
   its replacement is also named `web-0`, never a random new name.
2. Its **own dedicated, durable storage** that reattaches to the same Pod
   name every time, even after deletion.
3. **Ordered startup** — `web-0` fully starts before `web-1` starts, and so on.

---

## 2. New concepts, explained from scratch

### PV, PVC, and StorageClass — three different things, not one renamed

- **PV (Persistent Volume)** — an actual piece of real storage that exists in
  the cluster (a real disk), independent of any Pod.
- **PVC (Persistent Volume Claim)** — a Pod's *request* for storage
  ("I need 100Mi"), which Kubernetes matches (binds) to a PV.
- **StorageClass** — a template that tells Kubernetes *how* to automatically
  create a new PV whenever a PVC asks for one, with no human manually writing
  a PV file. This is called **dynamic provisioning**, and it's the default on
  almost every real cluster (including the `kind` cluster used in this repo,
  which ships a StorageClass named `standard`).

The full chain, in order, every time a StatefulSet Pod is created:

```
Pod's volumeMount  ->  PVC (a request)  ->  StorageClass creates a PV on demand  ->  PV is bound to the PVC
```

A `volume`/`volumeMount` (from Project 04) is a generic *slot* on a Pod for
attaching storage. What fills that slot can be different things:
`emptyDir` (temporary, dies with the Pod), a ConfigMap (Project 02), or —
new here — a PVC (durable, survives Pod deletion).

### `volumeClaimTemplates` — why not just write a `volume:` block like Project 04?

In Project 04, we hand-wrote one Pod with one fixed name and one specific
volume. That's only possible because there was exactly one Pod, with a name
we chose ourselves.

A StatefulSet doesn't let you hand-write individual Pods — you write one
`template:`, and Kubernetes stamps out multiple copies from it. Because
StatefulSet Pods get predictable numbered names (`web-0`, `web-1`, ...),
Kubernetes *can* auto-generate one matching PVC per Pod, using
`volumeClaimTemplates` as the cookie-cutter. Each numbered Pod gets its own
private PVC — `web-0` never shares storage with `web-1`.

### Headless Service — why StatefulSet needs a different kind of Service

Every Service in Project 01 gave you **one shared IP** that load-balances
across all matching Pods — you don't know or care which specific Pod answers.
That's the correct behavior for stateless apps.

A StatefulSet sometimes needs the opposite: the ability to reach **one
specific numbered Pod directly** (e.g., "talk to database replica #0
specifically, because only it can accept writes"). Setting `clusterIP: None`
makes a Service **headless** — instead of one shared IP, Kubernetes creates
an individual DNS name for each Pod: `web-0.web-headless`,
`web-1.web-headless`, etc.

**Important:** a regular ClusterIP Service and a headless Service are not
"one is better than the other" — they solve different problems and both
exist side by side in real production clusters. Stateless apps (web servers,
APIs) use a normal ClusterIP Service for automatic load balancing. Stateful
apps (databases) use headless Services when a caller must target one specific
instance by name. A single real-world StatefulSet often has both: a headless
one for direct addressing, and a normal one on top for general
"give me any read replica" traffic.

---

## 3. Architecture diagram

![StatefulSet + PVC architecture](architecture-diagram.png)

Reading it top to bottom:

- The **headless service** at the top doesn't load-balance — it hands out a
  DNS name per Pod so each one is individually addressable. It points at
  both Pods below it.
- Each **Pod** (`web-0`, `web-1`) has one container that writes into `/data`
  — that's the `volumeMount`, the "door" connecting the container to shared
  storage.
- That door connects down to a **PVC**, and each Pod has its own separate
  one — this is what `volumeClaimTemplates` creates automatically.
- Each PVC is bound to a **PV** that was never written by hand — it appeared
  automatically because of the `standard` StorageClass.

The two columns never touch each other — that is the entire point of
per-Pod storage. `web-0`'s data has zero connection to `web-1`'s data, all
the way down to the disk.

---

## 4. Files in this project

| File | Purpose |
|---|---|
| `headless-service.yaml` | Headless Service (`clusterIP: None`) required for StatefulSet Pod DNS |
| `web-statefulset.yaml` | 2-replica StatefulSet; each Pod gets its own PVC via `volumeClaimTemplates` |
| `architecture-diagram.png` | Visual diagram of the full Pod -> PVC -> PV chain |

---

## 5. Step-by-step: build it yourself

```bash
kubectl apply -f headless-service.yaml
kubectl apply -f web-statefulset.yaml

# watch the ORDERED startup -- web-0 must finish becoming Running
# before web-1 even starts being created
kubectl get pods -w
```

Confirm two separate PVCs were auto-created, one per Pod:

```bash
kubectl get pvc
# expect: data-web-0   Bound   ...
#         data-web-1   Bound   ...
```

See the PVs that were dynamically created behind the scenes (you never wrote
these yourself):

```bash
kubectl get pv
```

Confirm each Pod wrote to its own separate file:

```bash
kubectl exec web-0 -- cat /data/identity.txt
kubectl exec web-1 -- cat /data/identity.txt
```

---

## 6. The real test: does the data survive Pod deletion?

This is the entire point of the project. Delete a Pod completely:

```bash
kubectl delete pod web-0
kubectl get pods -w
```

Watch it come back -- **same name**, `web-0`, not a new random name. Once
it's `Running` again, check the file:

```bash
kubectl exec web-0 -- cat /data/identity.txt
```

**Expected result -- TWO lines, not one:**

```
I am web-0, started at <original timestamp>
I am web-0, started at <new, later timestamp>
```

Why two lines: deleting the Pod destroys the container completely.
Kubernetes rebuilds a brand new `web-0` container from scratch, which runs
its startup command again and **appends** a new line. But the **old line is
still there**, because the underlying storage (the PVC/PV) was never
deleted along with the Pod -- only the Pod died. The replacement Pod
reattaches to the *same* PVC it had before.

Compare this to Project 04's `emptyDir`: deleting that Pod would have wiped
its data completely, because `emptyDir` storage is tied to the Pod's own
lifetime. This is the exact difference StatefulSet + PVC exists to fix.

---

## 7. Common beginner questions (answered here so you don't have to ask)

**Q: Does the headless Service give the Pods their stable names?**
No. The **StatefulSet** alone is responsible for the ordinal naming
(`web-0`, `web-1`, ...), driven by its `replicas` count. The headless
Service's only job is adding DNS records on top of Pods that already have
those names. Delete the headless Service and the Pods keep their names --
you'd just lose the ability to reach them by that DNS name.

**Q: Does a normal ClusterIP Service give a fixed IP to one specific Pod?**
No -- this is a common misconception. ClusterIP gives a fixed IP to the
**Service**, which load-balances across *whichever* Pods currently match its
label selector. Individual Pods underneath it still have their own
changing names/IPs, exactly like in Project 01. ClusterIP was never
"Pod-specific" addressing.

**Q: If headless lets you reach a Pod by name, why not always do that
instead of ClusterIP, even in production?**
Because most apps *want* load balancing. A stateless web server behind a
Deployment runs many identical replicas specifically so callers don't have
to care which one answers. If every caller had to pick one exact named Pod,
you'd lose load balancing and have to build that routing logic yourself.
ClusterIP and headless Services solve different problems -- production
clusters use both, for different kinds of workloads.

**Q: Are `volume`, PV, and PVC just three names for the same thing?**
No. `volumes:`/`volumeMounts:` (on a Pod) is a generic *slot* for attaching
storage -- the thing filling that slot could be `emptyDir`, a ConfigMap, or a
PVC. A **PVC** is a *request* for storage. A **PV** is the *actual disk*
that request gets matched to. They are three distinct layers, not
synonyms.

**Q: Why can't we just manually attach a separate `emptyDir` volume to each
Pod, like Project 04, instead of using `volumeClaimTemplates`?**
Two reasons. First, `emptyDir` isn't durable -- it dies with the Pod, which
defeats the entire purpose of this project. Second, a Deployment or
StatefulSet doesn't let you hand-write individual Pods one at a time -- you
write one `template:`, and Kubernetes stamps out copies from it. Because
StatefulSet Pods get predictable numbered names, Kubernetes can
automatically create one PVC per Pod ordinal using `volumeClaimTemplates` --
this automatic, per-replica PVC creation is not possible with a Deployment,
since Deployment Pods get random names and are meant to be interchangeable.
