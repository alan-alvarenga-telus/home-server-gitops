# home-server-gitops

GitOps repo for a single-node k3s cluster (`home-server`). ArgoCD reconciles cluster state from `main`.

See [`CLAUDE.md`](./CLAUDE.md) for repository conventions and the architecture deep-dive. This README is the **runbook** — how to rebuild the cluster from zero.

## Architecture at a glance

```
bootstrap/                 # kubectl apply once, after ArgoCD is installed
  projects/
    infrastructure.yaml    # AppProject "infrastructure"
    apps.yaml              # AppProject "apps"
  infrastructure.yaml      # Root Application → watches infrastructure/
  apps.yaml                # Root Application → watches apps/

infrastructure/            # Platform layer (operators, controllers, CRDs)
  cloudnative-pg/

apps/                      # Workload layer (services consuming infra capabilities)
  postgres/
```

Two `AppProject`s scope what each layer can do. The `apps` project explicitly forbids cluster-scoped resources — a guardrail against a workload chart silently elevating to ClusterRoles.

## Prerequisites

On the host that will run k3s:

- Linux (tested on Ubuntu 7.0.0 / k3s v1.35.5+k3s1)
- 4 GB RAM minimum, 8 GB recommended
- A user with `sudo` access
- `curl`, `openssl` available

On your workstation (where you run `kubectl`):

- `kubectl` 1.32+
- `git`
- A password manager — you'll generate two database passwords during bootstrap

## Bootstrap procedure

Tier 0–2 are manual. After Tier 2, everything else reconciles from this repo.

### Tier 0: Install k3s

```bash
# On the cluster host
curl -sfL https://get.k3s.io | sh -

# Copy kubeconfig to your workstation
sudo cat /etc/rancher/k3s/k3s.yaml > ~/kubeconfig-home-server
# Edit ~/kubeconfig-home-server: replace 127.0.0.1 with the host's LAN IP
export KUBECONFIG=~/kubeconfig-home-server
kubectl get nodes  # should show "home-server   Ready"
```

### Tier 1: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for the controller to be ready
kubectl wait -n argocd --for=condition=available --timeout=300s deployment/argocd-server

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Forward the UI to your workstation (Ctrl+C to stop)
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Then open https://localhost:8080  (user: admin, password: from the previous step)
```

**Verification:** All seven ArgoCD pods Running:
```bash
kubectl get pods -n argocd
```

### Tier 2: Hand the keys to GitOps

The order matters: AppProjects must exist before any Application that references them.

```bash
# Clone the repo on your workstation
git clone https://github.com/alan-alvarenga-telus/home-server-gitops.git
cd home-server-gitops

# Apply AppProjects FIRST
kubectl apply -f bootstrap/projects/

# Then the two root Applications
kubectl apply -f bootstrap/infrastructure.yaml
kubectl apply -f bootstrap/apps.yaml
```

**Verification:**
```bash
kubectl get appproject -n argocd
# Expected: default, infrastructure, apps

kubectl get application -n argocd
# Expected: infrastructure, apps, cloudnative-pg, postgres
```

### Tier 2.5: Back up the sealed-secrets master key (one-time, irreversible)

After the `infrastructure` Application syncs, the **sealed-secrets controller** runs in `kube-system` and generates a master keypair. This key is the only thing that can decrypt the `SealedSecret` resources stored in this repo. **Lose it and every credential in git becomes unrecoverable.**

Wait for the controller to be Ready, then export the key to your password manager and shred the local copy:

```bash
kubectl wait -n kube-system --for=condition=available --timeout=120s \
  deployment/sealed-secrets

kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master.key

# Upload sealed-secrets-master.key to your password manager NOW, then:
shred -u sealed-secrets-master.key   # or `rm -P` on macOS, or `rm` if you don't have shred
```

To restore on a fresh cluster: `kubectl apply -f sealed-secrets-master.key && kubectl -n kube-system rollout restart deployment sealed-secrets`.

### Tier 2.6: Install kubeseal and seal the Postgres credentials

The `kubeseal` CLI is how you encrypt `Secret` manifests with the master key before committing them to git.

```bash
# Install kubeseal on your workstation (Linux x86_64)
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | jq -r .tag_name | sed 's/^v//')
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf "kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

Then seal each credential. Save the generated passwords to your password manager before you forget — sealed-secrets stores them encrypted but doesn't show them back to you.

```bash
# Postgres owner credential (consumed by CNPG bootstrap.initdb.secret)
PG_AUTHENTIK_PASS=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
echo "authentik db password: $PG_AUTHENTIK_PASS"   # save in password manager

kubectl create secret generic postgres-authentik \
  --namespace=postgres \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=authentik \
  --from-literal=password="$PG_AUTHENTIK_PASS" \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > apps/postgres/manifests/postgres-authentik.sealedsecret.yaml

# Postgres superuser
PG_SUPERUSER_PASS=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
echo "postgres superuser password: $PG_SUPERUSER_PASS"   # save in password manager

kubectl create secret generic postgres-superuser \
  --namespace=postgres \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=postgres \
  --from-literal=password="$PG_SUPERUSER_PASS" \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > apps/postgres/manifests/postgres-superuser.sealedsecret.yaml
```

If you previously created the raw Secrets manually (during a pre-sealed-secrets bootstrap), delete them now so the controller can take ownership of fresh ones:

```bash
kubectl delete secret postgres-authentik postgres-superuser -n postgres --ignore-not-found
```

Commit and push the sealed manifests:

```bash
git add apps/postgres/manifests/*.sealedsecret.yaml
git commit -m "seal postgres credentials"
git push
```

ArgoCD reconciles the `postgres` Application → the SealedSecret manifests land in the cluster → the sealed-secrets controller decrypts each into a real Secret → CNPG picks them up and bootstraps the database.

### Tier 3: Wait for reconciliation

```bash
# Watch Applications reach Synced/Healthy.
kubectl get application -n argocd -w
# Ctrl+C once all four show Synced + Healthy.

# CNPG operator should be running
kubectl get pods -n cnpg-system

# Postgres Cluster should have one Running pod
kubectl get cluster,pods -n postgres
```

The first sync of `postgres` may fail once with `Cluster CRD not found` while the CNPG operator is still installing. ArgoCD will retry; it self-heals within ~60 seconds. Lesson 7 (sync-wave discipline) will make this race deterministic.

## Verification: end-to-end smoke test

After bootstrap, you should be able to connect to Postgres from inside the cluster:

```bash
# Run an ephemeral psql pod
kubectl run -it --rm psql --image=postgres:16 --restart=Never -n postgres -- \
  bash -c 'PGPASSWORD="<the-authentik-password>" psql -h postgres-rw.postgres.svc -U authentik -d authentik -c "SELECT version();"'
```

If you get the Postgres version banner back, the full stack — k3s → ArgoCD → CNPG operator → Cluster CR → primary pod → readable database — is healthy.

## What this README does NOT cover

- **Ingress / TLS.** No ingress controller is installed; everything is reached via `kubectl port-forward`.
- **Backups.** CNPG can back up to S3-compatible storage; not configured.
- **Monitoring.** No Prometheus, Grafana, or alerting. `monitoring.enablePodMonitor: false` in the cluster spec.
- **Storage HA.** `local-path` is single-node only. A node failure loses data. Acceptable for a homelab; not for anything that matters.
- **Cluster upgrades.** k3s upgrades are out-of-band (`curl … sh -` again with a newer release).

## Recovery: "I broke it, start over"

The whole point of GitOps is that a clean rebuild is cheap. To wipe and rebuild:

```bash
# On the host
sudo /usr/local/bin/k3s-uninstall.sh

# Then run the full bootstrap above from Tier 0.
```

The only thing you cannot recover from git is the two database passwords. Keep them in your password manager. (Lesson 5 will fix this — the encrypted form will live in git, and the decryption key will be the only thing you need outside.)

## Day-2 operations

Once bootstrapped, all changes flow through git:

```bash
# Make a change
vim apps/postgres/manifests/cluster.yaml
git add -p && git commit -m "increase postgres shared_buffers"
git push

# ArgoCD picks it up within ~3 minutes (or click "Refresh" in the UI for instant)
```

To pause GitOps for emergency manual debugging, disable auto-sync on a specific Application — but always commit the fix afterward, or `selfHeal: true` will revert your manual change on the next reconciliation loop.
