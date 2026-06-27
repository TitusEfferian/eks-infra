# argocd — Argo CD + app-of-apps bootstrap

Umbrella chart around `argo/argo-cd` (chart **10.0.0** / appVersion **v3.4.4**)
for the `tituseff-playground` EKS Auto Mode cluster, plus the CRs that bootstrap
GitOps for this repo.

- Argo CD is **non-HA** (single replicas, autoscaling off), `server.insecure=true`,
  exposed via an **internet-facing HTTP:80 ALB** (no ACM/TLS). See `values.yaml`
  under `argo-cd:` and the security caveat in the top-level README.
- The chart's **own server Ingress is disabled** (`argo-cd.server.ingress.enabled:
  false`): the 10.0.0 template hard-codes a `host:` (falling back to
  `global.domain`) and cannot render a truly host-less rule. We ship our own
  host-less Ingress in `templates/argocd-server-ingress.yaml` (class `alb`,
  HTTP:80, target-type ip, health check `/healthz`).
- `templates/appproject.yaml` renders the `infra` `AppProject`.
- `templates/app-root.yaml` renders the `infra-root` **root** Application — but
  only once `repo.url` is set (it is guarded). It points at the `apps/` chart,
  which renders the child Applications (`alb`, `future`, optionally `argocd`).

All CRs are created in the Argo CD install namespace (`.Release.Namespace`,
normally `argocd`) because the controller only watches its own namespace by default.

## Install

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm dependency build .          # vendors argo-cd-10.0.0.tgz into charts/
helm upgrade --install argocd . -n argocd --create-namespace
```

With `repo.url` empty this installs Argo CD + the `infra` AppProject + the
host-less Ingress only (no root app). Then activate GitOps:

```yaml
# values.yaml
repo:
  url: "https://github.com/<you>/eks-infra.git"
  targetRevision: main
```

```bash
helm upgrade argocd . -n argocd
```

Now `infra-root` is rendered and syncs `apps/` from git → child apps `alb` (wave 0)
and `future` (wave 1). Because the children are synced by the **parent** root app,
their sync-wave annotations are honoured.

## Registering the git repo (Argo CD v3)

Argo CD v3 registers repositories via **Secrets**, not `argocd-cm`.

- **Public** `eks-infra` repo: nothing to do — `repo.url` is enough.
- **Private** repo: create a Secret in the `argocd` namespace labelled
  `argocd.argoproj.io/secret-type: repository`, e.g.:
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: eks-infra-repo
    namespace: argocd
    labels:
      argocd.argoproj.io/secret-type: repository
  stringData:
    type: git
    url: https://github.com/<you>/eks-infra.git
    username: <user-or-token-name>
    password: <token>
  ```
  (or `argocd repo add ... --username ... --password ...`).

## Adoption / ownership (helm install first, then GitOps)

Bootstrap order is: `helm install alb` and `helm install argocd` first, then Argo
CD **adopts** those releases via the child Applications.

- **Clean adoption.** The `alb` child app renders the **same** chart at the same
  path, so the live (Helm-created) objects already match desired state and Argo
  adopts rather than recreates them. `ServerSideApply=true` (set on each child)
  makes adoption cleaner and avoids client-side apply size limits on big CRDs.
- **`prune` is safe for the Helm release secret.** Pruning only removes tracked
  resources absent from desired state; the `sh.helm.release.v1.*` Secret is not
  part of the rendered output, so Argo never tracks or prunes it.
- **Single owner.** After Argo CD adopts a chart, **stop** running `helm upgrade`
  on it — drive changes through git so Helm and Argo don't fight.

## Self-management (`apps.selfManage`) — OFF by default

Set `selfManage: true` in `apps/values.yaml` to render the `argocd` child
Application (Argo CD managing its own release). It is deliberately conservative:
`automated.prune: false` (never prune its own components) and `ignoreDifferences`
on `Secret/argocd-secret` `/data` (so Argo doesn't fight the server-generated
admin/session/signing keys). Enable only once `argocd/values.yaml` in git
**exactly** matches the installed release; otherwise Argo CD will "correct" its own
running components mid-reconcile.

## CRD upgrade caveat

- **Argo CD CRDs upgrade normally.** The argo-cd chart **templates** its CRDs
  (`crds.install: true`, `crds.keep: true`), so `helm upgrade` updates them in
  place and they survive uninstall.
- **LBC CRDs do not** — the `alb` chart's subchart ships its CRDs under `crds/`,
  which Helm installs only once and never upgrades. See `../alb/README.md` for how
  to re-apply them on a controller version bump.
