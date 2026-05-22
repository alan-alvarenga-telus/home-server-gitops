# home-server-gitops

GitOps repo for a single-node k3s cluster (`home-server`). ArgoCD reconciles cluster state from `main`.

See [`CLAUDE.md`](./CLAUDE.md) for repository conventions and architecture detail. **This README is the runbook** — how to rebuild the cluster from zero, and how to fix it when it breaks.

> **About this repo:** this is a learning-and-stability environment. The owner is intentionally taking the production-shaped path (proper AppProjects, sealed-secrets, sync waves) even though it's a homelab. If you find a step that doesn't work, that's normal — see [Troubleshooting](#troubleshooting) at the bottom and update the README with what you learned.

## Architecture at a glance

```
bootstrap/                 # kubectl apply, manually, after ArgoCD is installed
  projects/
    infrastructure.yaml    # AppProject "infrastructure" (chart repos allowed)
    apps.yaml              # AppProject "apps" (no cluster-scoped resources)
  infrastructure.yaml      # Root Application -> watches infrastructure/
  apps.yaml                # Root Application -> watches apps/

infrastructure/            # Platform layer (operators, controllers, CRDs)
  cloudnative-pg/
  sealed-secrets/

apps/                      # Workload layer
  postgres/
    application.yaml
    base/                  # Universal manifests
      kustomization.yaml
      cluster.yaml
    overlays/
      home-server/         # Cluster-specific kustomize overlay
        kustomization.yaml
        *.sealedsecret.yaml
```

Two `AppProject`s scope what each layer can do. The `apps` project explicitly forbids cluster-scoped resources — a guardrail against a workload chart silently elevating to ClusterRoles.

## Prerequisites

**On the cluster host:**
- Linux (tested: Ubuntu 7.0.0, k3s v1.35.5+k3s1)
- 4 GB RAM minimum, 8 GB recommended
- A user with `sudo`
- `curl`, `openssl` available

**On your workstation:**
- `kubectl` 1.32+
- `git`
- A password manager — you'll save two database passwords and one sealed-secrets master key

## Bootstrap from zero

Six tiers, manual through tier 5, GitOps from tier 6 forward. Each tier ends with a verification you can run before moving on.

### Tier 0: Install k3s

```bash
# On the cluster host
curl -sfL https://get.k3s.io | sh -

# Copy kubeconfig to your workstation
sudo cat /etc/rancher/k3s/k3s.yaml > ~/kubeconfig-home-server
# Edit ~/kubeconfig-home-server: replace 127.0.0.1 with the host's LAN IP
export KUBECONFIG=~/kubeconfig-home-server
```

**Verify:** `kubectl get nodes` shows `home-server   Ready`.

### Tier 1: Install ArgoCD

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=available --timeout=300s deployment/argocd-server
```

Retrieve the initial admin password and forward the UI:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Open https://localhost:8080 (user: admin)
```

**Verify:** `kubectl get pods -n argocd` shows 7 pods Running.

### Tier 2: Apply AppProjects, then root Applications

Order matters: AppProjects must exist before any Application that references them.

```bash
git clone https://github.com/alan-alvarenga-telus/home-server-gitops.git
cd home-server-gitops

# Projects FIRST
kubectl apply -f bootstrap/projects/

# Then the root Applications
kubectl apply -f bootstrap/infrastructure.yaml
kubectl apply -f bootstrap/apps.yaml
```

**Verify:**
```bash
kubectl get appproject -n argocd
# Expected: default, infrastructure, apps

kubectl get application -n argocd
# Expected: infrastructure, apps, cloudnative-pg, sealed-secrets, postgres
```

The `postgres` Application will show `OutOfSync` / `Healthy` until you finish Tier 4 (sealing the database credentials). That's expected — keep going.

### Tier 3: Back up the sealed-secrets master key

After the `sealed-secrets` Application syncs, the controller in `kube-system` generates a master keypair. **This key is the only thing that can decrypt the SealedSecrets stored in this repo. Lose it and every credential in git is unrecoverable.**

Wait for the controller, then export and back up the key:

```bash
kubectl wait -n kube-system --for=condition=available --timeout=180s \
  deployment/sealed-secrets-controller

kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master.key
```

**NOW, before doing anything else:**
1. Open `sealed-secrets-master.key` and paste the entire contents into a secure note in your password manager. Title it `home-server sealed-secrets master key`.
2. Shred the local copy. The file is in `.gitignore` but you still don't want it sitting around.

```bash
shred -u sealed-secrets-master.key   # or `rm -P` on macOS, `rm` if neither is available
```

**Recovery procedure** (to put on the same password manager entry): on a fresh cluster, restore with
```bash
kubectl apply -f <restored-key-file>
kubectl -n kube-system rollout restart deployment sealed-secrets-controller
```

### Tier 4: Install kubeseal and seal the Postgres credentials

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | jq -r .tag_name | sed 's/^v//')
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf "kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

Seal each Postgres credential. Save the generated passwords to your password manager **before you redirect the output** — once sealed, you can't read them back.

```bash
# Pick a strong password and save it to your password manager NOW
PG_AUTHENTIK_PASS=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
echo "authentik db password: $PG_AUTHENTIK_PASS"

kubectl create secret generic postgres-authentik \
  --namespace=postgres \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=authentik \
  --from-literal=password="$PG_AUTHENTIK_PASS" \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > apps/postgres/overlays/home-server/postgres-authentik.sealedsecret.yaml

# Repeat for the superuser
PG_SUPERUSER_PASS=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
echo "postgres superuser password: $PG_SUPERUSER_PASS"

kubectl create secret generic postgres-superuser \
  --namespace=postgres \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=postgres \
  --from-literal=password="$PG_SUPERUSER_PASS" \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > apps/postgres/overlays/home-server/postgres-superuser.sealedsecret.yaml
```

If any raw `Secret` with these names already exists in the cluster (from a partial bootstrap), delete them so the sealed-secrets controller can claim ownership:

```bash
kubectl delete secret postgres-authentik postgres-superuser postgres-uthentik \
  -n postgres --ignore-not-found
```

Commit and push:

```bash
git add apps/postgres/overlays/home-server/*.sealedsecret.yaml
git commit -m "seal postgres credentials"
git push
```

### Tier 5: Wait for full reconciliation

```bash
# All Applications should reach Synced + Healthy
kubectl get application -n argocd -w
# Ctrl+C when stable

# Sealed-secrets controller should have produced the real Secrets
kubectl get secret -n postgres
# Expected: postgres-authentik, postgres-superuser, plus the CNPG-managed TLS secrets

# Postgres cluster should be Healthy with one Running pod
kubectl get cluster,pods -n postgres
```

### Tier 6: ArgoCD reconciles everything else

You're done with manual steps. Future changes flow through git:

```bash
vim apps/postgres/base/cluster.yaml
git add -p && git commit -m "increase shared_buffers"
git push
# ArgoCD picks it up within ~3 minutes (or click Refresh in the UI for instant).
```

## End-to-end smoke test

After bootstrap, connect to Postgres from inside the cluster:

```bash
PG_PASS=<the authentik password from your password manager>
kubectl run -it --rm psql --image=postgres:16 --restart=Never -n postgres -- \
  bash -c "PGPASSWORD='$PG_PASS' psql -h postgres-rw.postgres.svc -U authentik -d authentik -c 'SELECT version();'"
```

A PostgreSQL version banner means the full stack is working: k3s → ArgoCD → CNPG operator → Cluster CR → primary pod → readable database with the credentials in your sealed Secret.

## Day-2 operations

- **Add a new app:** create `apps/<name>/application.yaml`, commit, push. The `apps` root picks it up via recursive scan.
- **Add a new operator/controller:** same shape under `infrastructure/<name>/`. If it pulls from a new Helm repo, add the repo URL to `bootstrap/projects/infrastructure.yaml` and **re-apply that file manually** — AppProjects aren't reconciled by ArgoCD yet (see `Lesson 5.5` in our backlog).
- **Add a new credential:** generate the value, seal it with `kubeseal`, commit the `.sealedsecret.yaml` next to the app that consumes it.
- **Pause GitOps temporarily:** disable auto-sync on a specific Application in the UI. Re-enable when done. If you make changes by hand during the pause, commit them — `selfHeal: true` reverts manual edits on the next reconciliation loop.

## Troubleshooting

### `repo <url> is not permitted in project '<project>'`
Reason: the chart repo is in `bootstrap/projects/<project>.yaml` (git) but you haven't re-applied the project to the cluster yet. AppProjects don't reconcile from git automatically.
Fix: `kubectl apply -f bootstrap/projects/`

### `services "sealed-secrets-controller" not found`
Reason: either the sealed-secrets controller isn't running yet, or the Helm chart's `fullnameOverride` was removed. The chart should produce a deployment AND service named `sealed-secrets-controller` — check `infrastructure/sealed-secrets/application.yaml` for `fullnameOverride: sealed-secrets-controller` in the helm values.
Fix: ensure the override is set; commit; let ArgoCD reconcile; verify with `kubectl get svc -n kube-system | grep sealed-secrets-controller`.

### `Cluster CRD not found` (postgres app fails initial sync)
Reason: the CNPG operator hasn't installed the `postgresql.cnpg.io` CRDs yet. Race between the infrastructure and apps roots syncing.
Fix: ArgoCD retries on backoff; usually resolves within 60s. Lesson 7 (sync-wave discipline) makes this deterministic via `SkipDryRunOnMissingResource=true`.

### `AppProject "apps" not found` (or similar)
Reason: you applied a root Application before its AppProject existed.
Fix: `kubectl apply -f bootstrap/projects/` first, then re-trigger sync via `kubectl patch app <name> -n argocd --type merge -p '{"operation":{"sync":{}}}'`.

### Application stuck `OutOfSync / Missing` for a long time
Reason: usually a sync error you haven't seen. Look at events.
Diagnostic: `kubectl describe application <name> -n argocd | tail -40`. The `Message:` fields under operationState are the real error.

### A `kubeseal` command failed but left an empty file
Reason: shell redirection `>` creates the file before the command runs. If kubeseal fails, the file exists at 0 bytes and would silently deploy nothing if committed.
Fix: `rm` the empty file, re-run with correct flags, verify `wc -l <file>` is non-zero before committing.

### You committed something to git that you shouldn't have
- Secrets in plaintext: rotate them in the actual systems immediately. Git history is forever.
- Master key file: same — the key is compromised. Rotate by generating a new sealed-secrets keypair, re-encrypting every SealedSecret. See the [sealed-secrets key renewal docs](https://github.com/bitnami-labs/sealed-secrets#sealed-secrets-key-renewal-re-encryption-and-related-issues).

## What this README does NOT cover

- **Ingress / TLS.** No ingress controller installed; reach things via `kubectl port-forward`.
- **Backups.** CNPG can back up to S3-compatible storage; not configured here.
- **Monitoring.** No Prometheus, Grafana, or alerting. `monitoring.enablePodMonitor: false` in `apps/postgres/base/cluster.yaml`.
- **Storage HA.** `local-path` is single-node only. Node loss = data loss.
- **k3s upgrades.** Out-of-band: `curl -sfL https://get.k3s.io | sh -` with a newer release.
- **AppProject GitOps reconciliation.** Project changes still require manual `kubectl apply`. Self-managed projects are a pending lesson.

## Recovery: "I broke it, start over"

The whole point of GitOps is that a clean rebuild is cheap.

```bash
# On the host
sudo /usr/local/bin/k3s-uninstall.sh

# Then run this README from Tier 0.
```

The only things you cannot recover from git:
1. The two database passwords (in your password manager)
2. The sealed-secrets master key (also in your password manager)

Restore the key (Tier 3 recovery procedure) before applying the postgres SealedSecrets, otherwise the controller will refuse to decrypt them (it was sealed with the *old* key, and a fresh install generates a new one by default).
