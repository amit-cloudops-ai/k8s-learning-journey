# 02 — ConfigMaps + Secrets

**Goal:** externalize config from the container image, and feel the real
difference between env-var injection and mounted-volume injection.

## Why this matters

Right now, changing anything about your app's config means editing the Pod
spec or rebuilding the image. A ConfigMap separates *config* from *code* —
same nginx image, different ConfigMap per environment. A Secret is identical
mechanically, but flagged for sensitive values — though as you'll see,
"flagged" doesn't mean "encrypted."

## Step 1 — apply the ConfigMap and Secret

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl get configmap nginx-html-config -o yaml
kubectl get secret db-secret -o yaml
```

**Notice:** the Secret's `data` field shows `DB_PASSWORD` as a base64 blob,
not plaintext. Decode it yourself to prove the point:

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

**This is the "why" that matters:** base64 is encoding, not encryption.
Anyone with read access to Secrets in your cluster can decode this in one
command. Real secret protection needs RBAC restricting who can even read
Secret objects (project 07) — Kubernetes Secrets alone aren't a vault.

## Step 2 — deploy the Pod using both

```bash
kubectl apply -f nginx-config-pod.yaml
kubectl get pods
```

## Step 3 — check the env vars landed

```bash
kubectl exec nginx-config-pod -- printenv | grep -E "GREETING|DB_PASSWORD"
```

Both values came from outside the image — nothing here was baked into
`nginx:1.27`.

## Step 4 — check the mounted volume

```bash
kubectl exec nginx-config-pod -- cat /usr/share/nginx/html/index.html
```

You should see the custom HTML from the ConfigMap, not nginx's default
welcome page. Confirm it over HTTP too:

```bash
kubectl port-forward pod/nginx-config-pod 8080:80
# visit http://localhost:8080, then Ctrl+C
```

## Step 5 — the real lesson: env vars are a snapshot, mounted volumes are live

Change the ConfigMap:

```bash
kubectl edit configmap nginx-html-config
# change the greeting value to something else, save and exit
```

Now check both places:

```bash
kubectl exec nginx-config-pod -- printenv | grep GREETING
kubectl exec nginx-config-pod -- cat /usr/share/nginx/html/index.html
```

**Notice:** the env var is still the OLD value — it was injected once, at Pod
startup, and never re-checked. But if you edited `index.html` in the
ConfigMap and wait ~60 seconds (kubelet's sync interval), the mounted file
updates live inside the running Pod, no restart needed.

This is the single most common ConfigMap gotcha in real deployments: people
expect env vars to update live like mounted files do. They don't. If an app
needs to pick up config changes without a restart, it must read from a
mounted file, not an env var — or you accept that env var changes require a
Pod restart to take effect.

## Step 6 — clean up

```bash
kubectl delete -f nginx-config-pod.yaml
kubectl delete -f configmap.yaml
kubectl delete -f secret.yaml
```

## What you should be able to explain after this

- Why ConfigMaps/Secrets exist — separating config from image
- Why Secrets are base64, not encrypted, and what that means for real security
- The concrete difference between env-var and mounted-volume injection
- Why that difference matters for apps that need live config reloads

## Status

- [ ] Applied ConfigMap and Secret
- [ ] Decoded the Secret's base64 value manually
- [ ] Confirmed env vars and mounted file both work
- [ ] Watched env var stay stale after a ConfigMap edit while the mounted file updated
- [ ] Notes written in own words
