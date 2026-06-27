# eks-infra — GitOps infra for `tituseff-playground` (EKS Auto Mode)

A Helm + Argo CD GitOps repo for the `tituseff-playground` EKS **Auto Mode**
cluster (region `ap-northeast-1`, Kubernetes 1.36). You bootstrap it with `helm
install`, then Argo CD adopts and continuously reconciles the charts in this repo
from your git remote (app-of-apps pattern with a root Application).

## Layout

```
eks-infra/
├── alb/        # standalone AWS Load Balancer Controller (chart 3.4.0)
│               #   owns a non-default 'alb' IngressClass; isolated from Auto Mode's LB
├── argocd/     # Argo CD (chart 10.0.0) + AppProject + host-less server Ingress + root app
├── apps/       # app-of-apps payload: the child Application CRs (alb, future, [argocd])
│               #   that the 'infra-root' root Application syncs from git
├── future/     # empty placeholder for the next core component
└── README.md   # this runbook
```

Why a separate `apps/` chart? Argo CD sync **waves are only honoured when a parent
Application syncs the children**. If the child `Application` CRs were rendered
straight into the argocd chart's `templates/`, Helm would apply them all at once
and the waves would be ignored. So the argocd chart renders a single **root**
Application (`infra-root`) that points at `apps/`, and `apps/` renders the
children — giving real wave ordering (alb wave 0 → future wave 1).

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

### 3. Install Argo CD into `argocd`

```bash
cd ../argocd
helm dependency build .                 # vendors argo-cd-10.0.0.tgz
helm upgrade --install argocd . -n argocd --create-namespace
```

With `repo.url` empty (the default) this installs Argo CD, the `infra`
`AppProject`, and our host-less `argocd-server` Ingress — but **not** the root
app yet. Argo CD is now up and reachable via the ALB.

### 4. Activate GitOps — point Argo CD at your git remote

Push this `eks-infra` repo to a git remote, then set `repo.url` in
`argocd/values.yaml` and upgrade:

```yaml
# argocd/values.yaml
repo:
  url: "https://github.com/<you>/eks-infra.git"
  targetRevision: main
```

```bash
helm upgrade argocd . -n argocd
```

That renders the `infra-root` root Application, which syncs `apps/` from git and
creates the child Applications: **alb** (wave 0, adopts the release from step 2)
then **future** (wave 1). If the repo is **private**, register it first (see
`argocd/README.md` — Argo CD v3 uses a repository **Secret**, not `argocd-cm`).

> **Single owner per chart.** Once Argo CD adopts a chart (e.g. `alb`), stop
> running `helm upgrade` on it yourself — let Argo CD be the sole owner and make
> changes via git. Running both fights over the resources.

> **Bootstrap with committed values only.** In steps 2–3, install using just the
> chart's committed `values.yaml` — do **not** pass `--set` or extra `-f`
> overrides. After Argo CD adopts a chart (step 4) it reconciles live resources to
> match git with `selfHeal: true`, so any install-time override that isn't
> committed to the chart will show as OutOfSync and be reverted on the next sync.
> Put all configuration in `values.yaml` instead.

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
