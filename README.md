# eks-infra — GitOps infra for `tituseff-playground` (EKS Auto Mode)

A Helm + Argo CD GitOps repo for the `tituseff-playground` EKS **Auto Mode**
cluster (region `ap-northeast-1`, Kubernetes 1.36). You bootstrap it with `helm
install`, then Argo CD adopts and continuously reconciles the charts in this repo
from your git remote (app-of-apps pattern with a root Application).

## Layout

```
eks-infra/
├── alb/           # standalone AWS Load Balancer Controller (chart 3.4.0)
│                  #   owns a non-default 'alb' IngressClass; isolated from Auto Mode's LB
├── argocd/        # Argo CD (chart 10.0.0) + host-less server Ingress (no Argo CRs here)
├── apps/          # app-of-apps payload: child Application CRs (alb, future, [argocd])
│                  #   that the 'infra-root' root Application syncs from git
├── bootstrap.yaml # AppProject + root Application — kubectl apply AFTER Argo CD is up
├── future/        # empty placeholder for the next core component
└── README.md      # this runbook
```

Why the split into `apps/` + `bootstrap.yaml`?
- **Wave ordering** needs a parent: Argo CD honours `sync-wave` only when a parent
  Application syncs the children. So `bootstrap.yaml`'s root Application
  (`infra-root`) points at `apps/`, and `apps/` renders the children — giving real
  ordering (alb wave 0 → future wave 1).
- **CRD ordering**: Argo CD CRs (`AppProject`, `Application`) cannot live in the
  `argocd` Helm chart — Helm can't create a CR whose CRD is installed by the *same*
  release ("no matches for kind AppProject"). So they live in `bootstrap.yaml`,
  applied with `kubectl apply` only **after** Argo CD (and its CRDs) is installed.

Each of `alb` / `argocd` wraps an upstream chart as a vendored subchart (`helm
dependency build` puts the `.tgz` in `charts/` — keep it in git). `apps` and
`future` have no dependencies.

## Pinned versions

| Component | Chart | App / image version |
| --- | --- | --- |
| Argo CD | `argo/argo-cd` 10.0.0 | v3.4.4 |
| AWS Load Balancer Controller | `eks/aws-load-balancer-controller` 3.4.0 | `public.ecr.aws/eks/aws-load-balancer-controller:v3.4.0` |
| Pod Identity IAM module (EKS stack) | `terraform-aws-modules/eks-pod-identity/aws` | `~> 2.8` |
| EKS module (EKS stack) | `terraform-aws-modules/eks/aws` | 21.24.0 |

## Operational constraints (read before you start)

- **Source-IP allowlist.** The cluster's public Kubernetes API endpoint is locked
  to a single allow-listed CIDR (`public_access_cidrs` in the EKS stack's
  `terraform.tfvars`). You must be on that allow-listed IP for `helm`/`kubectl` to
  reach the API server. If your IP changes, update `public_access_cidrs` in the EKS
  stack and re-apply.
- **Cluster RBAC.** Currently only the `AdministratorAccess` SSO principal has
  cluster RBAC (via the EKS access entry created at cluster creation). Authenticate
  as that principal before running `kubectl`/`helm`.
- **CLIs:** `terraform`, `helm` v3+ (developed against v4.1.4), `aws`, `kubectl`,
  and `argocd`. Point kubectl at the cluster:
  ```bash
  aws eks update-kubeconfig --name tituseff-playground --region ap-northeast-1
  ```

## Bootstrap order

### 1. Terraform — IAM + subnet tags (in the EKS stack)

The AWS-side prerequisites live in your existing EKS stack at
`my_terraform_collections/EKS` (file `aws-load-balancer-controller.tf`). It is
**additive** (a plan shows only the new Pod Identity module + `aws_ec2_tag`
resources). It creates:

- an IAM role + AWS LBC policy and an **EKS Pod Identity** association binding it
  to `kube-system / aws-load-balancer-controller` (no OIDC/IRSA, no SA annotation), and
- `kubernetes.io/role/elb=1` + `kubernetes.io/cluster/tituseff-playground=shared`
  tags on the default-VPC public subnets (so the controller can discover them).

```bash
cd /path/to/my_terraform_collections/EKS
terraform init
terraform apply
terraform output -raw lbc_pod_identity_role_arn   # informational (Pod Identity needs no annotation)
```

### 2. Install the ALB controller into `kube-system`

```bash
cd alb
helm dependency build .                 # vendors aws-load-balancer-controller-3.4.0.tgz
helm upgrade --install alb . -n kube-system
kubectl -n kube-system rollout status deploy/alb-aws-load-balancer-controller
```

On a fresh Auto Mode cluster the controller pod is `Pending` until Auto Mode
provisions the first node (~1-2 min) — the `rollout status` blocks until it is
`Ready`. **Wait for `Ready` before step 3** (the Argo CD Ingress is reconciled by
this controller).

### 3. Install Argo CD into `argocd`

```bash
cd ../argocd
helm dependency build .                 # vendors argo-cd-10.0.0.tgz
helm upgrade --install argocd . -n argocd --create-namespace
kubectl -n argocd rollout status deploy/argocd-server
```

This installs Argo CD (controllers + CRDs) and our host-less `argocd-server`
Ingress. **No** Argo CD `Application`/`AppProject` resources are created here —
those are bootstrapped next, once the CRDs exist.

### 4. Bootstrap GitOps (app-of-apps) — after Argo CD is up

```bash
cd ..
kubectl apply -f bootstrap.yaml          # creates the infra AppProject + infra-root root app
```

`infra-root` syncs `apps/` from git and creates the child Applications: **alb**
(wave 0, adopts the release from step 2) then **future** (wave 1). The repo URL is
set in `bootstrap.yaml` (default `https://github.com/TitusEfferian/eks-infra.git`)
— edit it if you forked/renamed. If the repo is **private**, register it first
(see `argocd/README.md` — Argo CD v3 uses a repository **Secret**, not `argocd-cm`).

> **Single owner per chart.** Once Argo CD adopts a chart (e.g. `alb`), stop
> running `helm upgrade` on it yourself — let Argo CD be the sole owner and make
> changes via git. Running both fights over the resources.

> **Bootstrap with committed values only.** In steps 2–3, install using just the
> chart's committed `values.yaml` — do **not** pass `--set` or extra `-f`
> overrides. After Argo CD adopts a chart it reconciles live resources to match git
> with `selfHeal: true`, so any install-time override that isn't committed to the
> chart will show as OutOfSync and be reverted on the next sync. Put all
> configuration in `values.yaml` instead.

## Access Argo CD

```bash
# ALB DNS name (wait ~2-3 min for ADDRESS to populate):
kubectl -n argocd get ingress argocd-server -w

# Initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Log in over plain HTTP (no TLS):
argocd login <alb-dns> --username admin --plaintext --grpc-web
# UI: http://<alb-dns>
```

## Security caveat (playground only)

This is an **insecure, internet-facing HTTP:80** exposure: `server.insecure=true`
serves the Argo CD API/UI without TLS and the ALB has a single HTTP listener, so
credentials and tokens cross the network in clear text (hence the `--plaintext`
flag on `argocd login`). Treat this strictly as a playground:

- **Rotate the admin password** immediately after first login and delete the
  `argocd-initial-admin-secret`.
- Do not store sensitive repos/credentials here.
- For real use, add ACM + an HTTPS:443 listener (see `argocd/values.yaml` /
  `argocd/templates/argocd-server-ingress.yaml`) and restrict the ALB.
