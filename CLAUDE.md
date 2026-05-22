# home-server-gitops

GitOps repo for a single-node k3s cluster the owner uses for **local development** but wants run as a **stable, production-shaped environment** — no "it's only dev" shortcuts. ArgoCD pulls from this repo and reconciles cluster state.

## How to work in this repo (read before editing)

The owner is **learning ArgoCD and kubectl** and is using this repo as a hands-on teaching exercise. When making changes:

- **Teach, don't just edit.** Before changing YAML, explain what you're doing, why it's the community best practice, and what the alternative would have been. Use accurate terminology (Application, AppProject, sync wave, kustomize overlay, CR vs CRD) so the owner can grep docs for more.
- **One lesson per step.** Don't bundle a typo fix with a refactor. Land each change atomically so the lesson is clear in `git log`.
- **Production-shaped, not production-scale.** Apply *stability* conventions immediately (proper layout, secrets tooling, sync waves, `ServerSideApply`, resource limits). Defer *scale* conventions until needed (HA replicas, ApplicationSets, multi-cluster generators) — on a single node they're YAGNI.
- **Don't propose "we'll fix it later when it matters."** Doing it right the first time is the point.

## Architecture

App-of-apps pattern, split into two layers:

- `bootstrap/` holds the two root `Application` CRs that you `kubectl apply` after ArgoCD itself is installed.
  - `bootstrap/infrastructure.yaml` watches `infrastructure/` recursively.
  - `bootstrap/apps.yaml` watches `apps/` recursively.
- `infrastructure/` — platform layer. Operators, controllers, CNI, storage, ingress, cert-manager, secrets controller. Provides *capabilities and CRDs*.
- `apps/` — workload layer. User-facing services and resources that *consume* infrastructure's capabilities.

Each subdirectory under `infrastructure/<name>/` or `apps/<name>/` holds one `application.yaml` (an ArgoCD `Application` CR). If the Application deploys in-repo manifests, those live under `<name>/manifests/`.

Current state:
- `infrastructure/cloudnative-pg/` — CNPG operator, Helm chart from `cloudnative-pg.github.io/charts`, namespace `cnpg-system`.
- `apps/postgres/` — a `postgresql.cnpg.io/v1` `Cluster` in namespace `postgres`, backing Authentik.

Ordering between Applications is controlled by `argocd.argoproj.io/sync-wave` annotations (lower = earlier). Wave ordering applies *within a single parent's sync* — to enforce "infrastructure before apps" across the two root Applications, rely on the `ServerSideApply` syncOption and tolerate-missing-CRD options on workloads. (See Lesson 7 — sync-wave discipline.)

## Conventions

- One Application per directory under `apps/` or `infrastructure/`. Name the file `application.yaml`.
- Decide infra vs. app by *lifecycle*, not by domain: the CNPG **operator** is infra, but the **`Cluster` CR** that uses CNPG is an app. CRDs and controllers live in `infrastructure/`; consumers of those CRDs live in `apps/`.
- In-repo manifests go in `<name>/manifests/` and the Application's `spec.source.path` points there.
- All Applications use `automated.prune: true` + `automated.selfHeal: true`. Use `ServerSideApply=true` for anything with CRDs or large objects.
- Use `sync-wave` annotations when one Application depends on CRDs or services from another (e.g. `postgres` waits for the CNPG operator).
- Secrets are referenced by name (e.g. `postgres-authentik`, `postgres-superuser`) — they are NOT in this repo. They're applied to the cluster out-of-band. Do not commit secret material.
- Storage class is `local-path` (k3s built-in). Single-node — no replication, no HA yet.

## Repo URL

`https://github.com/alan-alvarenga-telus/home-server-gitops` — referenced from Application `repoURL` fields. Update both if the repo moves.

## Validation

There is no CI yet. Before committing:

- `kubectl apply --dry-run=client -f <file>` for raw manifests.
- For Helm-based Applications, double-check the `helm:` key spelling — ArgoCD silently ignores unknown keys, so a typo like `heml:` will skip your values without error.

## Boundaries

- Don't commit secrets, kubeconfigs, or anything under a `Secret` kind with populated `data`/`stringData`.
- Don't change `spec.destination.server` from `https://kubernetes.default.svc` unless you're intentionally targeting a remote cluster.
- Don't disable `prune` or `selfHeal` without a reason noted in the commit message — the whole point of this repo is that git is the source of truth.
