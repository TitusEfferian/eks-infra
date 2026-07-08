# apps — app-of-apps payload

The child Argo CD `Application` CRs that the **root** Application (`infra-root`,
defined in `../argocd/templates/app-root.yaml`) syncs from git. This chart is **not
installed with `helm`** directly — the root app renders it, injecting `repo.url`
and `repo.targetRevision` as Helm parameters.

It lives in its own chart (not in the argocd chart's `templates/`) so that the
children are synced by a **parent** Application — which is what makes Argo CD
**sync waves** take effect (alb wave 0 → future wave 1). Children rendered straight
into a chart's `templates/` get applied all at once by Helm and the waves are
ignored.

## Children

| App | path | dest namespace | wave | prune | gate |
| --- | --- | --- | --- | --- | --- |
| `alb` | `alb` | `kube-system` | 0 | yes | always |
| `future` | `future` | `future` | 1 | yes | always (no-op until the chart is populated) |
| `egress-test` | `egress-test` | `egress-test` | 2 | yes | `egressTest: true` only |
| `argocd` | `argocd` | `argocd` | -1 | **no** | `selfManage: true` only |

## Values

- `repo.url`, `repo.targetRevision` — injected by the root app (defaults are only
  for direct `helm template apps .`).
- `project` — the AppProject the children belong to; must match the argocd chart's
  `project` (default `infra`).
- `selfManage` — render the `argocd` self-management app (advanced/opt-in). See
  `../argocd/README.md`.
- `egressTest` — render the `egress-test` netshoot verification app (opt-in; a
  deliberate test that provisions a real Auto Mode node). See
  `../egress-test/README.md`.

## Render locally

```bash
helm template apps . -n argocd \
  --set repo.url=https://github.com/<you>/eks-infra.git
# add --set selfManage=true to also render the argocd self-management app
# add --set egressTest=true to also render the egress-test verification app
```
