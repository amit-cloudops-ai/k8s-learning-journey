# Project 05 — StatefulSet + PVC

Demonstrates how StatefulSet gives each Pod a stable name and its own dedicated,
durable storage (via PVC/PV), unlike a Deployment where Pods are interchangeable
and storage (emptyDir) dies with the Pod.

## Architecture
![StatefulSet + PVC architecture](architecture-diagram.png)

## Files
- `headless-service.yaml` — headless Service (clusterIP: None) required for StatefulSet DNS
- `web-statefulset.yaml` — 2-replica StatefulSet, each Pod gets its own PVC via volumeClaimTemplates

## What was proven
Deleted `web-0` entirely. Kubernetes recreated it with the same name and reattached
the same PVC — the pre-deletion log line was still present in `/data/identity.txt`,
proving the data survived Pod deletion (unlike emptyDir in Project 04).
