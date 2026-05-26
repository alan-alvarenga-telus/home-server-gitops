# home-server-gitops

GitOps repo for a single-node k3s cluster the owner uses for **local development** but wants run as a **stable, production-shaped environment** — no "it's only dev" shortcuts. ArgoCD pulls from this repo and reconciles cluster state.

## How to work in this repo (read before editing)

The owner is **learning ArgoCD and kubectl** and is using this repo as a hands-on teaching exercise. When making changes:

- **Teach, don't just edit.** Before changing YAML, explain what you're doing, why it's the community best practice, and what the alternative would have been. Use accurate terminology (Application, AppProject, sync wave, kustomize overlay, CR vs CRD) so the owner can grep docs for more.
- **One lesson per step.** Don't bundle a typo fix with a refactor. Land each change atomically so the lesson is clear in `git log`.
- **Production-shaped, not production-scale.** Apply *stability* conventions immediately (proper layout, secrets tooling, sync waves, `ServerSideApply`, resource limits). Defer *scale* conventions until needed (HA replicas, ApplicationSets, multi-cluster generators) — on a single node they're YAGNI.
- **Don't propose "we'll fix it later when it matters."** Doing it right the first time is the point.

## Architecture

App-of-apps pattern, split into two layers governed by two `AppProject` boundaries:

- `bootstrap/` holds the manifests you `kubectl apply` after ArgoCD itself is installed.
  - `bootstrap/projects/infrastructure.yaml` — `AppProject` "infrastructure". Allows cluster-scoped resources, broader source repos.
  - `bootstrap/projects/apps.yaml` — `AppProject` "apps". **No** cluster-scoped resources allowed; this repo only as source.
  - `bootstrap/infrastructure.yaml` — root Application, `project: infrastructure`, watches `infrastructure/` recursively.
  - `bootstrap/apps.yaml` — root Application, `project: apps`, watches `apps/` recursively.
- `infrastructure/` — platform layer. Operators, controllers, CNI, storage, ingress, cert-manager, secrets controller. Provides *capabilities and CRDs*.
- `apps/` — workload layer. User-facing services and resources that *consume* infrastructure's capabilities.

**Bootstrap order:** AppProjects MUST be applied before any Application that references them. Apply `bootstrap/projects/` first, then the two root manifests.

Each subdirectory under `infrastructure/<name>/` or `apps/<name>/` holds one `application.yaml` (an ArgoCD `Application` CR). If the Application deploys in-repo manifests, those live under `<name>/manifests/`.

Current state:
- `infrastructure/cloudnative-pg/` — CNPG operator, Helm chart from `cloudnative-pg.github.io/charts`, namespace `cnpg-system`.
- `infrastructure/sealed-secrets/` — Bitnami sealed-secrets controller in `kube-system` (renamed to `sealed-secrets-controller` so `kubeseal` defaults work).
- `apps/postgres/` — a `postgresql.cnpg.io/v1` `Cluster` in namespace `postgres`, backing Authentik. Kustomize layout: `base/` + `overlays/home-server/`.
- `apps/authentik/` — SSO/identity provider, Helm chart from `charts.goauthentik.io`, namespace `authentik`. Multi-source Application (chart + values ref + supplementary kustomize overlay containing the SealedSecret, a bundled redis Deployment/Service, and an explicit Namespace). Exposed via Traefik at `authentik.home-server.lan`.

## Hostname & Ingress

k3s ships Traefik in `kube-system` as the default ingress controller (LoadBalancer service `kube-system/traefik`). HTTP-only for now — no TLS, no cert-manager.

**Cluster ingress IP:** `192.168.31.218` — the external IP of the Traefik LoadBalancer service (`kube-system/traefik`). Verify with:
```bash
kubectl get svc -n kube-system traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
All `*.home-server.lan` traffic terminates here and Traefik routes by Host header.

**Hostname convention:** `<app>.home-server.lan` for any app that needs browser access. The cluster expects clients to resolve these names to `192.168.31.218` — set up however you prefer (LAN DNS server with a wildcard, per-workstation `/etc/hosts` entry, etc.). DNS provisioning is out of scope for this repo; the cluster only owns the Ingress configuration.

**Why `.lan` and not `.local`:** `.local` is reserved by mDNS, which breaks unicast DNS resolution on Apple devices (macOS/iOS) unless Avahi is in the loop. We use `.lan` (common convention; not IANA-reserved but stable in practice) so any device on the LAN resolves names via the Pi-hole wildcard. `.internal` (RFC 8375 / ICANN-reserved 2024) is a stricter alternative if `.lan` ever causes friction.

**To expose a new HTTP app:**
1. In the app's chart values (or `Ingress` manifest), set `ingressClassName: traefik` and `hosts: [<app>.home-server.lan]`.
2. Browse to `http://<app>.home-server.lan` — Pi-hole resolves it, Traefik routes by Host header.

No need to touch the AppProject — `Ingress` is namespaced, allowed by `apps`' `namespaceResourceWhitelist: '*'`.

**To expose a TCP service (e.g. Postgres):** Traefik doesn't handle non-HTTP by default. Use a `Service` of `type: LoadBalancer` — k3s's klipper-lb assigns it the same node IP as Traefik, but on whatever port you declare. Example: `apps/postgres/overlays/home-server/postgres-rw-external.yaml` exposes the CNPG primary on `5432`. Pi-hole's wildcard means `postgres.home-server.lan:5432` resolves to the right IP automatically — no separate DNS record needed.

Don't modify operator-managed services. Always create a *new* Service that selects the same pods. CNPG owns `postgres-rw`/`postgres-ro`/`postgres-r` (ClusterIP); we add `postgres-rw-external` (LoadBalancer) alongside.

## Sync ordering

ArgoCD provides three ordering mechanisms; pick the right one for the right problem:

1. **Sync phases** (`PreSync` / `Sync` / `PostSync`) — for hooks like one-shot migration Jobs. Rarely used here.
2. **Sync waves** (`argocd.argoproj.io/sync-wave: "N"` annotation) — orders resources within a single parent's sync. Lower = earlier. **Only works inside one parent's tree.** Wave annotations on resources in `apps/` cannot order them against resources in `infrastructure/` — those are two independent parents.
3. **Sync options** — per-resource behavior toggles. Most important for cross-tree CRD races:
   - `SkipDryRunOnMissingResource=true` — workloads tolerate "CRD not found" during dry-run and retry the apply after the CRD arrives. Required on any Application that consumes a CRD installed by a different root (e.g. `apps/postgres` consuming the CNPG operator's CRDs from `infrastructure/cloudnative-pg`).

**Within the `apps/` tree:** sync-wave orders Applications relative to each other (`postgres` is wave 1; an Authentik app that depends on postgres would be wave 2; etc.).
**Across `apps/` and `infrastructure/`:** waves don't help. Use `SkipDryRunOnMissingResource=true` and let ArgoCD retries handle eventual consistency.

## Conventions

- One Application per directory under `apps/` or `infrastructure/`. Name the file `application.yaml`.
- Decide infra vs. app by *lifecycle*, not by domain: the CNPG **operator** is infra, but the **`Cluster` CR** that uses CNPG is an app. CRDs and controllers live in `infrastructure/`; consumers of those CRDs live in `apps/`.
- Every Application's `spec.project` matches the directory it lives in: `infrastructure` for `bootstrap/infrastructure.yaml` and everything under `infrastructure/`; `apps` for `bootstrap/apps.yaml` and everything under `apps/`. **Never use `project: default`** — it's a wide-open free-for-all.
- The `apps` AppProject allows exactly one cluster-scoped resource: `Namespace` (no group). Apps that ship namespaced RBAC (Role/RoleBinding) must declare their own `Namespace` with `argocd.argoproj.io/sync-wave: "-1"` to dodge ArgoCD's `rbacReconcile`-before-`CreateNamespace` race. **ClusterRole/ClusterRoleBinding/CRDs/etc. remain forbidden** in this project; those belong in `infrastructure/`.
- When adding a new app that pulls a remote Helm chart, add the chart repo URL to the appropriate AppProject's `sourceRepos` list, or the sync will fail with a sourceRepos validation error.
- **Manifest-based apps** use kustomize layering (`base/` + `overlays/<cluster>/`):
  ```
  apps/<name>/
    application.yaml              # spec.source.path: apps/<name>/overlays/<cluster>
    base/
      kustomization.yaml          # resources: [the universal manifests]
      <resource>.yaml
    overlays/
      <cluster>/                  # named after the target cluster, not "dev"/"prod"
        kustomization.yaml        # resources: [../../base, plus cluster-specific]
        <cluster-specific>.yaml   # SealedSecrets, patches, etc.
  ```
  The Application's `spec.source.path` points at the overlay, never at the base. ArgoCD auto-detects `kustomization.yaml` and runs kustomize.

- **Chart-based apps** use a multi-source Application — no `base/` because the Helm chart IS the universal layer:
  ```
  apps/<name>/
    application.yaml              # spec.sources: chart + repo-ref + repo-path
    overlays/
      <cluster>/
        kustomization.yaml        # resources: [supplementary resources only]
        values.yaml               # Helm values, referenced via $values ref;
                                  # NOT a k8s resource, NOT listed under resources:
        *.sealedsecret.yaml       # supplementary SealedSecrets, etc.
  ```
  The Application has three sources: the chart, a `ref: values` source pointing at this repo (so `$values` resolves), and a path source pointing at the overlay. ArgoCD renders chart + kustomize together as one Application.
- SealedSecrets live in the overlay, not the base. They're encrypted to a specific cluster's master key, so they're inherently cluster-specific.
- Before committing kustomize changes, sanity-check the build: `kubectl kustomize apps/<name>/overlays/<cluster>/`. Catches malformed references before ArgoCD does.
- All Applications use `automated.prune: true` + `automated.selfHeal: true`. Use `ServerSideApply=true` for anything with CRDs or large objects.
- Use `sync-wave` annotations when one Application depends on CRDs or services from another (e.g. `postgres` waits for the CNPG operator).
- **Secrets use Sealed-Secrets.** Never commit a raw `Secret` manifest. Always go through `kubeseal` to produce a `SealedSecret` CR — those ARE committable. Naming convention: `<secret-name>.sealedsecret.yaml`, colocated in the consuming app's kustomize **overlay** (e.g. `apps/postgres/overlays/home-server/postgres-authentik.sealedsecret.yaml`) — not the base, since SealedSecrets are cluster-specific.
- Sealed-Secrets are **strict-scoped by default** — encrypted for the exact `namespace/name` pair. Renaming or moving the `SealedSecret` to a different namespace breaks decryption. This is the security property we want; don't override it without a reason.
- The sealed-secrets controller's master key lives in `kube-system` (selector: `sealedsecrets.bitnami.com/sealed-secrets-key`). It's the only thing that can decrypt SealedSecrets in this repo — backed up to the owner's password manager. Restoration procedure in `README.md`.
- The sealed-secrets Helm chart sets `fullnameOverride: sealed-secrets-controller` so the deployment/service names match `kubeseal`'s compiled-in defaults (`sealed-secrets-controller` in `kube-system`). Without this, every `kubeseal` invocation needs `--controller-name=sealed-secrets --controller-namespace=kube-system` flags. Don't remove the override.
- Storage class is `local-path` (k3s built-in). Single-node — no replication, no HA yet.
- The Postgres cluster has `enableSuperuserAccess: true` so the `postgres` superuser can connect remotely via the LoadBalancer for admin tasks (CREATE DATABASE, CREATE EXTENSION). CNPG defaults this to `false` for safety; we override because the cluster is LAN-only and the password is strong. Application workloads must still use per-app accounts (`authentik`, etc.), not the superuser.

## Private Docker registry

`infrastructure/registry/` runs `registry:2` exposed at `https://registry.home-server.lan` via Traefik. htpasswd auth, sealed creds in `infrastructure/registry/overlays/home-server/registry-auth.sealedsecret.yaml`. TLS terminated by a self-signed cert (10-year validity, CN matches hostname) stored in `registry-tls.sealedsecret.yaml`; private key never leaves the cluster.

**Two pieces of host-level state live outside this repo** (one-time setup, documented in `README.md`):
- Workstation: the registry's CA cert at `/etc/docker/certs.d/registry.home-server.lan/ca.crt`. Extract from cluster: `kubectl get secret registry-tls -n registry -o jsonpath='{.data.tls\.crt}' | base64 -d`. Docker auto-discovers it; no daemon restart needed.
- k3s host: `/etc/rancher/k3s/registries.yaml` with `auth.username` / `auth.password` and `tls.ca_file` pointing to the same CA cert (also copied to the host). Containerd pulls authenticated automatically — **no per-namespace imagePullSecrets needed.**

**Build → push → deploy workflow:** `docker build -t registry.home-server.lan/<app>:<tag>` → `docker push registry.home-server.lan/<app>:<tag>` → reference `image: registry.home-server.lan/<app>:<tag>` in any Deployment manifest under `apps/`. The cluster pulls via the containerd config.

**Storage**: 20 GiB PVC on `local-path`. Inspect usage with `kubectl exec -n registry deploy/registry -- du -sh /var/lib/registry`. When it fills up, bump the PVC size and let CSI resize.

**Image deletion**: `REGISTRY_STORAGE_DELETE_ENABLED=true` is set, so `curl -X DELETE` against the v2 API works. The actual disk space is reclaimed only after running `registry garbage-collect` — schedule a CronJob if this becomes a problem.

## Adding a database for a new app

Use CNPG's declarative primitives — never manual `psql` against the cluster for things git should own.

1. Generate a password (`openssl rand -base64 24 | tr -d '/+=' | head -c 32`) and save to your password manager.
2. Seal it into the **postgres namespace** as `postgres-<appname>.sealedsecret.yaml` under `apps/postgres/overlays/home-server/`. Type `kubernetes.io/basic-auth`; keys `username` + `password`.
3. Add a `managed.roles[]` entry to `apps/postgres/base/cluster.yaml` referencing the SealedSecret name. Set `login: true` and `ensure: present`. Don't grant `superuser`/`createdb`/`createrole` unless the app actually needs them.
4. Add a `Database` CR under `apps/postgres/base/databases/<appname>.yaml`, `owner: <appname>`, `cluster.name: postgres`, `ensure: present`. Add it to `apps/postgres/base/kustomization.yaml` resources list.
5. Seal a *second* SealedSecret in the consuming app's namespace with the same password formatted for that app's env (`DATABASE_URL`, `<APP>_DB_PASSWORD`, etc.) Two SealedSecrets, one password — the price of namespace isolation. If this duplication starts to hurt, install Reflector to mirror Secrets across namespaces; until then, manual is fine.
6. Commit. CNPG reconciles: creates/alters the role with the right password, then creates the database. The new app picks up its credential from its own namespace Secret.

**Never** manually `CREATE USER` or `CREATE DATABASE` against the cluster from psql — git becomes the lying source of truth, and the next cluster rebuild loses everything.

## Repo URL

`https://github.com/alan-alvarenga-telus/home-server-gitops` — referenced from Application `repoURL` fields. Update both if the repo moves.

## Validation

There is no CI yet. Before committing:

- `kubectl apply --dry-run=client -f <file>` for raw manifests.
- For Application files, use `kubectl explain application.spec.source.directory` (and similar) to confirm field names. **ArgoCD silently ignores unknown keys** — there is no schema validation error for typos. Known traps we've hit in this repo:
  - `heml:` instead of `helm:` (the values block is skipped, chart defaults apply)
  - `recursive: true` instead of `recurse: true` (recursion never happens; root scans only the top level)
- For Helm chart values, **the chart's deprecation messages tell you a field is gone but not what to use instead.** Always sanity-check values structure against the *current* chart schema before pushing:
  ```bash
  helm repo add <name> <url>
  helm repo update
  helm show values <name>/<chart> | less
  ```
  Known traps we've hit:
  - Authentik chart 2026.x: top-level `envFrom:`, `env:`, `envValueFrom:`, `ingress:` are all deprecated. Use `authentik.existingSecret.secretName` for credentials (single Secret with ALL `AUTHENTIK_*` env vars including non-secret ones like DB host); use `server.ingress.*` for ingress. The chart also dropped its bundled redis subchart, so the workload must provide its own — we ship a minimal `redis.yaml` in the overlay.
  - When a chart provisions cluster-scoped RBAC (ClusterRole/ClusterRoleBinding) that the `apps` AppProject refuses, look for a chart toggle to disable it. The Authentik chart's outpost RBAC is toggled off via `serviceAccount.clusterRole.enabled: false`. Don't loosen the AppProject guardrail to accommodate a chart — find the toggle.
- After applying any Application change to the cluster, diff the live spec against git: `kubectl get application <name> -n argocd -o yaml | yq '.spec.source' | diff - <(yq '.spec.source' <file>)`. The cluster and the file should be byte-identical; if not, something patched the live resource without a commit.

## Root Application discovery rule

Root Applications (those in `bootstrap/`) MUST set `directory.include` to filter for only Application files. Otherwise `recurse: true` pulls in every YAML under the tree — including the workload manifests that child Applications are *supposed* to deploy — and the root tries to apply them directly. Use:

```yaml
directory:
  recurse: true
  include: '{application.yaml,**/application.yaml}'
```

Child Applications then deploy their own manifest trees via their own `spec.source.path`, which is not scanned by the root.

## Boundaries

- Don't commit raw `Secret` kinds with populated `data`/`stringData`. Use `SealedSecret` instead — see the secrets convention above.
- Don't commit kubeconfigs, kubeconfig fragments, or the sealed-secrets master key file. The master key file is never long-lived on disk; it goes straight to the password manager after `kubectl get`.
- Don't change `spec.destination.server` from `https://kubernetes.default.svc` unless you're intentionally targeting a remote cluster.
- Don't disable `prune` or `selfHeal` without a reason noted in the commit message — the whole point of this repo is that git is the source of truth.
