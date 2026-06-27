# alb — standalone AWS Load Balancer Controller

Wrapper chart around `eks/aws-load-balancer-controller` (chart **3.4.0** /
appVersion **v3.4.0**) for the `tituseff-playground` EKS Auto Mode cluster.

It runs the controller as a **self-managed** install (spec.controller
`ingress.k8s.aws/alb`) and owns a dedicated, **non-default** `alb` IngressClass —
intentionally separate from EKS Auto Mode's built-in `eks.amazonaws.com/alb`
class, which is left dormant.

## What this chart does

- Vendors the upstream controller as a subchart (alias `lbc`) — see `values.yaml`.
- Installs into `kube-system`, with `serviceAccount.name:
  aws-load-balancer-controller` and **no** IRSA annotation: IAM comes from an EKS
  **Pod Identity** association created in terraform (`aws-load-balancer-controller.tf`).
- Renders our own `templates/ingressclass.yaml` (IngressClass `alb` +
  cluster-scoped `IngressClassParams`); `createIngressClassResource: false` keeps
  the subchart from creating a competing one.
- Sets `enableServiceMutatorWebhook: false` so it does **not** hijack
  `type=LoadBalancer` Services (those stay with Auto Mode's NLB), and
  `defaultTargetType: ip` (required on Auto Mode).

## Install

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm dependency build .          # vendors aws-load-balancer-controller-3.4.0.tgz into charts/
helm upgrade --install alb . -n kube-system
```

Prereq: run `terraform apply` in the EKS stack first so the Pod Identity role +
association and the subnet `kubernetes.io/role/elb` tags exist.

## Verify

```bash
kubectl -n kube-system rollout status deploy/alb-aws-load-balancer-controller
kubectl get ingressclass alb -o yaml      # spec.controller: ingress.k8s.aws/alb
```

## CRD upgrade caveat

The `aws-load-balancer-controller` subchart ships its CRDs (the `elbv2.k8s.aws`
group, e.g. `IngressClassParams`, `TargetGroupBinding`) under `crds/`, which Helm
installs **only on first install** and never touches on `helm upgrade`. So a chart
version bump will **not** update the CRDs. On a controller version bump, re-apply
the CRDs manually from the new chart, e.g.:

```bash
helm pull eks/aws-load-balancer-controller --version <new> --untar
kubectl apply -f aws-load-balancer-controller/crds/
```
