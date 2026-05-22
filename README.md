# home-server-gitops

GitOps repo for a single-node k3s cluster (`home-server`). ArgoCD reconciles cluster state from `main`.

See [`CLAUDE.md`](./CLAUDE.md) for repository conventions and architecture detail. **This README is the runbook** — how to rebuild the cluster from zero, and how to fix it when it breaks.

> **About this repo:** this is a learning-and-stability environment. The owner is intentionally taking the production-shaped path (proper AppProjects, sealed-secrets, sync waves) even though it's a homelab. If you find a step that doesn't work, that's normal — see [Troubleshooting](#troubleshooting) at the bottom and update the README with what you learned.

## Architecture at a glance

```
bootstrap/                 # kubectl apply, manually, after ArgoCD is installed
  projects/
    infrastructure.yaml    # AppProject "infrastructure" (chart repos allowed)
    apps.yaml              # AppProject "apps" (only Namespace allowed cluster-wide)
  infrastructure.yaml      # Root Application -> watches infrastructure/
  apps.yaml                # Root Application -> watches apps/

infrastructure/            # Platform layer (operators, controllers, CRDs)
  cloudnative-pg/          # Postgres operator (Helm)
  sealed-secrets/          # Bitnami sealed-secrets controller (Helm)

apps/                      # Workload layer
  postgres/                # CNPG Cluster CR. Kustomize base/overlay.
    application.yaml
    base/{kustomization.yaml,cluster.yaml}
    overlays/home-server/{kustomization.yaml,*.sealedsecret.yaml}
  authentik/               # SSO IdP. Multi-source: chart + values ref + overlay.
    application.yaml
    overlays/home-server/
      kustomization.yaml
      namespace.yaml       # explicit, sync-wave -1
      values.yaml          # Helm values (NOT a k8s resource)
      authentik-env.sealedsecret.yaml
      redis.yaml           # bundled redis (chart 2026.5 dropped its subchart)
```

Two `AppProject`s scope what each layer can do. The `apps` project allows **only `Namespace`** as a cluster-scoped resource — a deliberate guardrail against workload charts silently elevating to ClusterRoles/CRDs/etc.

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

Nine tiers, manual through Tier 7, GitOps from Tier 8 forward. Each tier ends with a verification you can run before moving on.

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
# Expected: infrastructure, apps, cloudnative-pg, sealed-secrets, postgres, authentik
```

The `postgres` and `authentik` Applications will show `OutOfSync` until you finish Tiers 4–5 (sealing all credentials). That's expected — keep going.

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

### Tier 5: Seal the Authentik credentials

Authentik needs four secrets in *its own* namespace (Kubernetes Secrets don't cross namespaces). One reads the Postgres password back from the cluster; the others are fresh.

```bash
# Reuse the existing postgres password from the cluster (no re-typing)
PG_AUTHENTIK_PASS=$(kubectl get secret postgres-authentik -n postgres \
  -o jsonpath='{.data.password}' | base64 -d)

# Fresh Authentik secrets — SAVE THESE to your password manager
AK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n/+=' | head -c 64)
AK_BOOTSTRAP_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
AK_BOOTSTRAP_TOKEN=$(openssl rand -hex 32)
echo "authentik akadmin password: $AK_BOOTSTRAP_PASSWORD"
echo "authentik bootstrap API token: $AK_BOOTSTRAP_TOKEN"

# Seal all 8 keys at once (chart's existingSecret path bypasses values.yaml,
# so non-secret fields like DB host live here too)
kubectl create secret generic authentik-env \
  --namespace=authentik \
  --from-literal=AUTHENTIK_SECRET_KEY="$AK_SECRET_KEY" \
  --from-literal=AUTHENTIK_POSTGRESQL__HOST="postgres-rw.postgres.svc.cluster.local" \
  --from-literal=AUTHENTIK_POSTGRESQL__NAME="authentik" \
  --from-literal=AUTHENTIK_POSTGRESQL__USER="authentik" \
  --from-literal=AUTHENTIK_POSTGRESQL__PASSWORD="$PG_AUTHENTIK_PASS" \
  --from-literal=AUTHENTIK_REDIS__HOST="authentik-redis-master.authentik.svc.cluster.local" \
  --from-literal=AUTHENTIK_BOOTSTRAP_PASSWORD="$AK_BOOTSTRAP_PASSWORD" \
  --from-literal=AUTHENTIK_BOOTSTRAP_TOKEN="$AK_BOOTSTRAP_TOKEN" \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > apps/authentik/overlays/home-server/authentik-env.sealedsecret.yaml

# Sanity check + commit
kubectl kustomize apps/authentik/overlays/home-server/ >/dev/null && echo "ok"

git add apps/authentik/overlays/home-server/authentik-env.sealedsecret.yaml
git commit -m "seal authentik environment"
git push
```

### Tier 6: Wait for full reconciliation

```bash
# All Applications should reach Synced + Healthy
kubectl get application -n argocd -w
# Ctrl+C when stable

# Postgres
kubectl get secret -n postgres
# Expected: postgres-authentik, postgres-superuser, plus the CNPG-managed TLS secrets
kubectl get cluster,pods -n postgres
# Expected: postgres cluster Healthy, 1 Running pod

# Authentik (takes ~2 min for first install — image pulls + DB migrations)
kubectl get pods -n authentik
# Expected: authentik-redis-master, authentik-server, authentik-worker — all 1/1 Running
```

### Tier 7: Hostname & Ingress

Authentik is exposed at `http://authentik.home-server.local` via Traefik (k3s's default ingress controller). DNS resolution is provided by the LAN's **Pi-hole** via a dnsmasq wildcard — `*.home-server.local` automatically resolves to Traefik's external IP for every device that uses Pi-hole as DNS.

```bash
# Get Traefik's external IP from the cluster
TRAEFIK_IP=$(kubectl get svc -n kube-system traefik \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Traefik IP: $TRAEFIK_IP"
```

**On the Pi-hole box (one-time setup):** SSH in as your normal user, then:
```bash
# From workstation:
ssh pi@192.168.31.57    # adjust user/IP if different

# Then on the Pi-hole:
sudo tee /etc/dnsmasq.d/02-home-server.conf >/dev/null <<'EOF'
# Wildcard DNS for the home-server k3s cluster.
# Any *.home-server.local resolves to Traefik's LoadBalancer IP.
address=/home-server.local/192.168.31.218
EOF
sudo pihole restartdns

# Verify on the Pi-hole itself
dig @127.0.0.1 authentik.home-server.local +short   # → 192.168.31.218
exit
```

If you ever change the Traefik IP (rare, but happens when reinstalling k3s), update the `address=` line and rerun `sudo pihole restartdns`. The IPs are also documented in `CLAUDE.md` → Network topology.

**Fallback** (no Pi-hole, or testing in isolation) — add `/etc/hosts` on the workstation:
```bash
echo "$TRAEFIK_IP authentik.home-server.local" | sudo tee -a /etc/hosts
```

Open `http://authentik.home-server.local` in your browser. Log in with:
- Username: `akadmin`
- Password: the `AK_BOOTSTRAP_PASSWORD` you saved in Tier 5

Note: HTTP only, no TLS yet. Cert-manager + Let's Encrypt is a future improvement.

### Tier 8: ArgoCD reconciles everything else

You're done with manual steps. Future changes flow through git:

```bash
vim apps/postgres/base/cluster.yaml
git add -p && git commit -m "increase shared_buffers"
git push
# ArgoCD picks it up within ~3 minutes (or click Refresh in the UI for instant).
```

## End-to-end smoke test

Two checks prove the full stack is healthy.

**Postgres** — connect from inside the cluster using the credentials in the sealed Secret:

```bash
PG_PASS=$(kubectl get secret postgres-authentik -n postgres \
  -o jsonpath='{.data.password}' | base64 -d)
kubectl run -it --rm psql --image=postgres:16 --restart=Never -n postgres -- \
  bash -c "PGPASSWORD='$PG_PASS' psql -h postgres-rw.postgres.svc -U authentik -d authentik -c 'SELECT version();'"
```

A PostgreSQL version banner means: k3s → ArgoCD → CNPG operator → Cluster CR → primary pod → DB authentication via sealed Secret all working.

**Authentik via Traefik** — exercises the ingress path:

```bash
curl -sI http://authentik.home-server.local | head -5
# Expected: HTTP/1.1 302 Found, Location: /if/flow/initial-setup/ (or /if/flow/default-authentication-flow/)
```

A 302 to an Authentik flow means: Traefik → Authentik server → Postgres → Redis all reachable. If you instead get `curl: (6) Could not resolve host`, your `/etc/hosts` entry is missing. If you get a Traefik 404, the Ingress hasn't reconciled yet — wait a moment and retry.

## Day-2 operations

- **Add a new app:** create `apps/<name>/application.yaml`, commit, push. The `apps` root picks it up via recursive scan.
- **Add a new operator/controller:** same shape under `infrastructure/<name>/`. If it pulls from a new Helm repo, add the repo URL to `bootstrap/projects/infrastructure.yaml` and **re-apply that file manually** — AppProjects aren't reconciled by ArgoCD yet.
- **Add a new credential:** generate the value, seal it with `kubeseal`, commit the `.sealedsecret.yaml` next to the app that consumes it.
- **Expose a new app via Traefik:** in the app's chart values (or its own Ingress manifest), set `ingressClassName: traefik` and a hostname matching `<app>.home-server.local`. Then on your workstation:
  ```bash
  TRAEFIK_IP=$(kubectl get svc -n kube-system traefik \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo "$TRAEFIK_IP <app>.home-server.local" | sudo tee -a /etc/hosts
  ```
  Then browse to `http://<app>.home-server.local`. No AppProject change needed — `Ingress` is a namespaced resource and is allowed by default.
- **Pause GitOps temporarily:** disable auto-sync on a specific Application in the UI. Re-enable when done. If you make changes by hand during the pause, commit them — `selfHeal: true` reverts manual edits on the next reconciliation loop.

## Troubleshooting

> **Before you panic:** ArgoCD has three independent status fields per Application. They can disagree. Trust them in this order:
>
> 1. **`status.health.status`** — what the actual workloads are doing right now. `Healthy` means it's running.
> 2. **`status.sync.status`** — does the cluster match git? `Synced` means yes.
> 3. **`status.operationState.message`** — the *last sync attempt's* result. **This is sticky** and can show old errors after the system recovered.
>
> If health and sync are both green, the cluster is fine — even if `operationState.message` shows a scary error from earlier. See "Stale ComparisonError after a path/rename change" below.

### `repo <url> is not permitted in project '<project>'`
Reason: the chart repo is in `bootstrap/projects/<project>.yaml` (git) but you haven't re-applied the project to the cluster yet. AppProjects don't reconcile from git automatically.
Fix: `kubectl apply -f bootstrap/projects/`

### `services "sealed-secrets-controller" not found`
Reason: either the sealed-secrets controller isn't running yet, or the Helm chart's `fullnameOverride` was removed. The chart should produce a deployment AND service named `sealed-secrets-controller` — check `infrastructure/sealed-secrets/application.yaml` for `fullnameOverride: sealed-secrets-controller` in the helm values.
Fix: ensure the override is set; commit; let ArgoCD reconcile; verify with `kubectl get svc -n kube-system | grep sealed-secrets-controller`.

### `Cluster CRD not found` (postgres app fails initial sync)
Reason: the CNPG operator hasn't installed the `postgresql.cnpg.io` CRDs yet. Race between the `infrastructure` and `apps` roots syncing at bootstrap.
Mitigation in place: the postgres Application carries `SkipDryRunOnMissingResource=true` in its `syncOptions`, so the dry-run step tolerates missing CRDs. ArgoCD retries the actual apply on the next reconciliation loop, by which point the CRDs are present. You may still see a transient error on first bootstrap; it self-clears within ~60s and does not require intervention.
If it persists past a few minutes: the CNPG operator install itself is stuck — check `kubectl get application cloudnative-pg -n argocd` and `kubectl get pods -n cnpg-system` for the real problem.

### `AppProject "apps" not found` (or similar)
Reason: you applied a root Application before its AppProject existed.
Fix: `kubectl apply -f bootstrap/projects/` first, then re-trigger sync via `kubectl patch app <name> -n argocd --type merge -p '{"operation":{"sync":{}}}'`.

### Application stuck `OutOfSync / Missing` for a long time
Reason: usually a sync error you haven't seen. Look at events.
Diagnostic: `kubectl describe application <name> -n argocd | tail -40`. The `Message:` fields under operationState are the real error.

### Stale `ComparisonError: app path does not exist` after a path/rename change
Symptom: you renamed/moved a directory (e.g. `manifests/` → `base/` + `overlays/`), pushed the commit, and the Application now shows `Sync: Synced, Health: Healthy` but the UI still has a red error about a path that no longer exists.
Reason: in the brief moment between commits, ArgoCD started a sync against the new revision but with the old `spec.source.path` still cached. That sync **operation** errored. The next **comparison** succeeded because git and cluster actually match — but `operationState.message` is sticky and retains the failed sync's text. Auto-sync doesn't re-run because there's no diff to reconcile.
Fix: force a fresh sync to overwrite the stale message.
```bash
kubectl patch app <name> -n argocd --type merge -p '{"operation":{"sync":{}}}'
# or click Sync in the UI
```
This is not a real error. If `Sync: Synced` and `Health: Healthy`, you're fine.

### Helm chart render fails with `<field> is deprecated` (Authentik chart)
Reason: the chart maintainers ship deprecations between releases. The error message tells you the field is gone but not what to use instead.
Diagnostic: `helm repo update && helm show values <repo>/<chart>` to see current schema. For Authentik specifically, current shape is documented in `CLAUDE.md` → Validation section.
Common Authentik 2026.x replacements: top-level `envFrom` / `env` / `envValueFrom` → `authentik.existingSecret.secretName`; top-level `ingress` → `server.ingress`; bundled redis subchart → not bundled, provide your own.

### `namespaces "authentik" not found` during rbacReconcile
Reason: ArgoCD's `kubectl auth reconcile` step for namespaced RBAC (Role/RoleBinding) runs before `CreateNamespace=true` can create the namespace. Charts that ship their own RBAC trip this race.
Fix: declare the `Namespace` as an explicit resource in the kustomize overlay with `argocd.argoproj.io/sync-wave: "-1"`. Requires the `apps` AppProject to allow the `Namespace` kind in its `clusterResourceWhitelist` — which is already configured for this repo. See `apps/authentik/overlays/home-server/namespace.yaml` for the pattern.

### A `kubeseal` command failed but left an empty file
Reason: shell redirection `>` creates the file before the command runs. If kubeseal fails, the file exists at 0 bytes and would silently deploy nothing if committed.
Fix: `rm` the empty file, re-run with correct flags, verify `wc -l <file>` is non-zero before committing.

### You committed something to git that you shouldn't have
- Secrets in plaintext: rotate them in the actual systems immediately. Git history is forever.
- Master key file: same — the key is compromised. Rotate by generating a new sealed-secrets keypair, re-encrypting every SealedSecret. See the [sealed-secrets key renewal docs](https://github.com/bitnami-labs/sealed-secrets#sealed-secrets-key-renewal-re-encryption-and-related-issues).

## What this README does NOT cover

- **TLS.** Traefik serves HTTP only. No cert-manager, no Let's Encrypt. Browser shows a "not secure" warning. A future lesson will add cert-manager + a local CA (or Let's Encrypt staging) for `*.home-server.local`.
- **Real DNS.** Hostnames resolve only via per-workstation `/etc/hosts` entries. A homelab DNS server (Pi-hole, AdGuard Home, dnsmasq) would let every device on the LAN reach apps by name without manual `/etc/hosts` edits.
- **Backups.** CNPG can back up to S3-compatible storage; not configured here.
- **Monitoring.** No Prometheus, Grafana, or alerting. `monitoring.enablePodMonitor: false` in `apps/postgres/base/cluster.yaml`.
- **Storage HA.** `local-path` is single-node only. Node loss = data loss.
- **k3s upgrades.** Out-of-band: `curl -sfL https://get.k3s.io | sh -` with a newer release.
- **AppProject GitOps reconciliation.** Project changes still require manual `kubectl apply`. Self-managed projects are a pending lesson.
- **Pinned chart versions.** All chart Applications use `targetRevision: "*"`. Convenient now, will eventually break when a maintainer ships a breaking change (we already hit this with Authentik 2026.5 deprecations). Pin specific versions when stability matters more than freshness.

## Recovery: "I broke it, start over"

The whole point of GitOps is that a clean rebuild is cheap.

```bash
# On the host
sudo /usr/local/bin/k3s-uninstall.sh

# Then run this README from Tier 0.
```

The only things you cannot recover from git:
1. The sealed-secrets master key
2. The Postgres `authentik` user password
3. The Postgres `postgres` superuser password
4. The Authentik admin (`akadmin`) password
5. The Authentik bootstrap API token

Items 2–5 are *encoded* in the SealedSecrets in git, but you can't decrypt them without the master key, which is item 1. Keep all five in your password manager.

**Restoration order matters:** restore the master key (Tier 3 recovery procedure) *before* the sealed-secrets controller starts processing the SealedSecrets in git. Otherwise the controller generates a new keypair, can't decrypt the existing SealedSecrets (which were sealed with the *old* key), and everything stays OutOfSync until you either restore the original key or re-seal every secret against the new key.
