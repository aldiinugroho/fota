# Shorebird OTA Control Plane — Setup & Ops (handoff)

Deployed from **this clean repo** (e.g. `aldiinugroho/shorebird-ota`) to Kubernetes via
Jenkins: Jenkins builds the control-plane image **from `code_push_server/`** (your source,
in this repo), pushes to GHCR, and applies `deploy/manifests/`. Two teams use it. Read
your section.

**TL;DR**
- **INFRA** owns the control plane: cluster, Jenkins, registry, TLS, secrets, backups.
  Deploys by pushing to this repo.
- **MOBILE** owns the app: builds the Flutter app + pushes `release`/`patch` to the
  control-plane URL. Never touches the cluster.
- They meet at one URL: `https://<DOMAIN>` (e.g. `https://ota.<lo>.com`).

## What gets deployed
`code_push_server/` (the "patch shop") → Docker image → **1 StatefulSet**
(SQLite in a PVC) + Service + TLS Ingress on the cluster. That's it.
This repo is deliberately minimal: **just the control plane + the deploy kit**. The rest of
the Shorebird toolchain (`shorebird_cli`, `code_push_client`, `elf`, …) stays in the toolchain
repo and runs on the mobile team's Mac/CI — it is NOT deployed here.

---

## 1) INFRA — setup & ops

### 1.1 Cluster prerequisites (once)
| Need | Why |
|---|---|
| Ingress controller (e.g. `nginx`) | serves `https://<DOMAIN>`; sets `INGRESS_CLASS` |
| **cert-manager** | auto-provisions the Let's Encrypt cert |
| A dynamic **StorageClass** | the `10Gi` PVC (SQLite + artifacts); sets `STORAGE_CLASS` |
| registry + k8s pull-secret | store/pull the control-plane image (GHCR) |

> No external DB service: the control plane uses **embedded SQLite** (a file on the PVC).
> Postgres+S3 only if you later need multi-replica.

### 1.2 GHCR
- Image: `ghcr.io/aldiinugroho/shorebird-controlplane:<git-sha>`.
- A PAT with `packages: write` → Jenkins credential `ghcr-push`; a k8s Secret `ghcr-pull`.

### 1.3 Jenkins
- Agent labeled `docker`: `docker`, `kubectl` (kubeconfig + `K8S_CONTEXT`), `git`, `envsubst`.
- Credentials: `ghcr-push`, `cps-api-key`, `cps-url-signing-secret` (the two `cps-*` =
  `openssl rand -hex 32` each, generated once).
- Job: **SCM = `aldiinugroho/shorebird` (main), Jenkinsfile location = `deploy/Jenkinsfile`.**
  Fill `environment {}` (`DOMAIN`, `NS`, `INGRESS_CLASS`, `STORAGE_CLASS`, `K8S_CONTEXT`).

### 1.4 Deploy flow (every push)
```
git push ─▶ Jenkins:
   1. docker build code_push_server/  → :git-sha  (your source, bakes the console)
   2. docker push → ghcr.io/aldiinugroho/shorebird-controlplane
   3. envsubst deploy/manifests/*.yaml → kubectl apply + set image
   4. curl https://<DOMAIN>/healthz   (smoke)
```

### 1.5 Ops
- **Back up** the `data` PVC (SQLite + artifacts = all state).
- **Secrets**: rotate via Jenkins `cps-*` creds → next deploy updates the Secret.
  **Don't rotate `URL_SIGNING_SECRET` after shipping** (signed URLs/managed keys break).
- **Rollback**: point `statefulset/cps` at an older image tag.
- **Monitor**: `kubectl -n $NS logs statefulset/cps-cps-0`, `/healthz` + `/readyz`.

---

## 2) MOBILE — using the control plane

You depend on one URL infra keeps up: `https://<DOMAIN>`.

### 2.1 App config (in the app repo, e.g. `moofi-flutter`)
```yaml
# shorebird.yaml
base_url: https://<DOMAIN>          # where the DEVICE checks/downloads updates
```
```bash
# build env — where the BUILD registers releases/patches
export SHOREBIRD_HOSTED_URL=https://<DOMAIN>
export SHOREBIRD_TOKEN="<cps-api-key>"      # infra gives you the key
```

### 2.2 release vs patch
- **release** = a new app binary (AOT base) → install from store / dev build.
- **patch** = a small diff to a release → OTA to installed apps. No store.
```bash
shorebird release android     # new base (after bumping the version)
shorebird patch android       # OTA a diff to a released version
```

### 2.3 One-time "rebase" (when infra changes the URL)
`base_url` is baked into a release (not a patch). When the URL changes: update
`base_url` → **re-release + reinstall**; update `SHOREBIRD_HOSTED_URL` on the build side.

### 2.4 iOS
Built on a **Mac** (Xcode) — can't build on Linux. But pushed to the **same** control
plane (`SHOREBIRD_HOSTED_URL` + token identical). Same `base_url` as Android.

### 2.5 In-app force-update
The app shows a full-screen **update gate** (download + progress) when a patch is available
(`UpdateGate`, foreground + resume + every 60s). Infra just keeps the URL up.

---

## Ownership boundary
| INFRA owns | MOBILE owns |
|---|---|
| cluster, K8s, Jenkins, registry, TLS, certs | the app, releases/patches, `shorebird.yaml`, build env |
| `cps-api-key` / `URL_SIGNING_SECRET` | uses the API key; never rotates it |
| the control-plane URL + keeping it up | points the app + build at that URL |
| backups | the in-app update gate UX |

## Files in this repo
- `deploy/` — the deploy kit: `manifests/{00-config,10-controlplane,20-ingress}.yaml`,
  `Jenkinsfile`, `values.env.example`, this doc.
- `code_push_server/` — the control-plane **source** (what Jenkins builds; its
  `console/` is baked into the image at `/console`).
