# Project 05 — StatefulSet + PVC

Demonstrates how StatefulSet gives each Pod a stable name and its own dedicated,
durable storage (via PVC/PV), unlike a Deployment where Pods are interchangeable
and storage (emptyDir) dies with the Pod.

## Architecture
StatefulSet + PVC architecture
<img width="2720" height="1720" alt="statefulset_pvc_architecture" src="https://github.com/user-attachments/assets/b6a9a8d3-7cfb-4f84-bb09-94df2807a350" />

## Files
- `headless-service.yaml` — headless Service (clusterIP: None) required for StatefulSet DNS
- `web-statefulset.yaml` — 2-replica StatefulSet, each Pod gets its own PVC via volumeClaimTemplates

## What was proven
Deleted `web-0` entirely. Kubernetes recreated it with the same name and reattached
the same PVC — the pre-deletion log line was still present in `/data/identity.txt`,
proving the data survived Pod deletion (unlike emptyDir in Project 04).
