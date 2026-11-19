# shorebird-ota

The **self-hosted Shorebird control plane** ("patch shop") + its K8s deploy kit, in one
clean repo. Jenkins builds the image **from `code_push_server/`** (in this repo) and deploys
it to Kubernetes.

## Layout
```
code_push_server/   # the control-plane source (Dart). Self-contained (no sibling deps).
  └─ console/        # the web console, baked into the image at /console
deploy/
  ├─ Jenkinsfile     # build code_push_server/ → push GHCR → apply manifests → smoke
  ├─ manifests/       # K8s: StatefulSet(SQLite+PVC) + Service + Ingress(TLS/cert-manager)
  ├─ values.env.example
  └─ OPERATIONS.md    # ← READ THIS: setup for infra (§1) + mobile (§2)
```

## How it works
```
git push ─▶ Jenkins ─▶ (1) docker build code_push_server/ (:git-sha) ─▶ push GHCR
              ─▶ (2) kubectl apply deploy/manifests/* + set image ─▶ (3) curl /healthz
```
Devices download OTA patches from the resulting `https://<DOMAIN>` control plane.

## Start here
- **Infra engineer** → `deploy/OPERATIONS.md §1` (cluster prereqs, Jenkins, GHCR, ops).
- **Mobile dev** → `deploy/OPERATIONS.md §2` (app config, release/patch, rebase, iOS).

## Push this repo
```bash
git remote add origin git@github.com:aldiinugroho/<nama-repo>.git
git add -A && git commit -m "shorebird-ota: control plane + K8s deploy kit" && git push -u origin main
```
(The control-plane source here was pulled from the `aldiinugroho/shorebird` fork; this repo
is the minimal, self-contained version for CI/CD.)
