# egress-test — per-client static-egress verification (netshoot)

Deploys a single [`nicolaka/netshoot`](https://github.com/nicolaka/netshoot) pod
**pinned to one subnet** so you can `exec` in and confirm its outbound public IP
equals a client's **dedicated NAT Elastic IP**. This verifies the per-client
static-egress solution on the `tituseff-playground` EKS **Auto Mode** cluster.

## How the pinning works

Per-client egress = a dedicated private **subnet** + dedicated **NAT gateway** +
dedicated **Elastic IP**. The client's pods must run on nodes **in that subnet**,
so their egress follows *that* subnet's route table (`0.0.0.0/0` → that NAT → that
EIP). On Auto Mode you can't edit the built-in NodePools' subnets, so this chart
ships its **own** NodeClass + NodePool:

- **`NodeClass`** (`eks.amazonaws.com/v1`) — `subnetSelectorTerms` resolves to
  **exactly one** subnet; `role` reuses the Auto Mode node role; `snatPolicy` is
  left at its default (`Random`) so pod egress SNATs to the node ENI and takes the
  subnet's NAT path. *(This is the Auto Mode API — not upstream `karpenter.k8s.aws`.)*
- **`NodePool`** (`karpenter.sh/v1`) — labels nodes `client=<name>` and taints
  them `client=<name>:NoSchedule`; `nodeClassRef` points at the NodeClass above.
- **`Deployment`** — the netshoot pod pins with **both** a `nodeSelector`
  (`client=<name>`) **and** a matching toleration, kept alive with `sleep infinity`.

## What it renders

| Kind | Name | Scope |
| --- | --- | --- |
| `Namespace` | `egress-test` (`.Values.namespace`) | cluster |
| `NodeClass` (`eks.amazonaws.com/v1`) | `<client>` | cluster |
| `NodePool` (`karpenter.sh/v1`) | `<client>` | cluster |
| `Deployment` | `netshoot` | namespaced |

## Configure

Edit `values.yaml` (all keys are documented inline). The ones you'll usually set:

| Value | Purpose | Default |
| --- | --- | --- |
| `client` | node label/taint + NodeClass/NodePool name + nodeSelector | `sample-client` |
| `subnet.id` | pin by subnet **id** (wins when set) | `""` |
| `subnet.tagKey` / `subnet.tagValue` | pin by **tag** when `subnet.id` is empty | `egress-client` / `sample-client` |
| `nodeRole` | Auto Mode node role **NAME** (not ARN) — **set before use** | _placeholder_ |
| `image.tag` | netshoot version | `v0.16` |

> **`nodeRole` — get the real value.** In the EKS Terraform stack
> (`my_terraform_collections/EKS`):
> ```bash
> terraform output -raw node_iam_role_arn   # arn:aws:iam::<acct>:role/<NAME>
> ```
> use the part after `role/`. The committed default is a **placeholder** — set the
> real name before enabling the chart, and re-read it whenever the cluster is rebuilt.

## Prerequisite: the dedicated subnet

**Production per-client path.** The client's dedicated private subnet (with its own
NAT + EIP) must exist and be tagged so `subnet.tag*` resolves to it and **only**
it, e.g. `egress-client=<client>`. That subnet + NAT + EIP + tag are created in
Terraform (separate repo) — this chart only consumes them.

**Quick pinning-mechanics test (no dedicated subnet yet).** Point the NodeClass at
an existing private subnet id and expect the cluster's **shared** NAT EIP. Fetch the
real values first — they change on every cluster/VPC rebuild, so never hardcode them:

```bash
# private subnet ids the cluster currently uses:
aws eks describe-cluster --name tituseff-playground \
  --query 'cluster.resourcesVpcConfig.subnetIds' --output text
# the shared NAT EIP curl should print:
VPC=$(aws eks describe-cluster --name tituseff-playground --query cluster.resourcesVpcConfig.vpcId --output text)
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=$VPC \
  --query 'NatGateways[].NatGatewayAddresses[].PublicIp' --output text
```

Pin the NodeClass to one of those subnet ids; `curl ifconfig.me` from the pod should
then print that shared NAT EIP. This proves the NodeClass/NodePool/nodeSelector
mechanics before the per-client dedicated subnet exists.

## Run it

Prereq: `kubectl` pointed at the cluster and you are on the API allowlisted IP
(see the repo root README).

### Option A — direct render + apply (standalone, best for the quick test)

```bash
cd egress-test

# Quick mechanics test against an existing private subnet (expect shared EIP):
helm template egress-test . \
  --set subnet.id=subnet-XXXXXXXX \
  | kubectl apply -f -

# Production per-client test (dedicated subnet tagged egress-client=<client>):
helm template egress-test . \
  --set client=<client> \
  | kubectl apply -f -
```

### Option B — GitOps (Argo CD)

The `egress-test` child Application is **gated off** by default. Set
`egressTest: true` in `../apps/values.yaml`, set the client/subnet in this chart's
`values.yaml`, commit, and let the root app sync (or `argocd app sync egress-test`).

## Test procedure

```bash
# 1. Auto Mode provisions a node in the pinned subnet (~1-2 min). Confirm it is in
#    the RIGHT subnet/AZ (the -L column shows the client label):
kubectl get nodes -l client=<client> -o wide -L topology.kubernetes.io/zone

# 2. Wait for the netshoot pod to be Ready:
kubectl -n egress-test rollout status deploy/netshoot

# 3. THE TEST — outbound public IP as seen from the pod:
kubectl -n egress-test exec deploy/netshoot -- curl -s ifconfig.me; echo
#   -> expect the client's dedicated NAT Elastic IP
#      (or the shared NAT EIP 54.65.125.80 for the quick mechanics test).
```

Extra checks (netshoot has the tools):

```bash
# Which subnet did the node actually land in? (map the node to its ENI/subnet)
kubectl get nodes -l client=<client> \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}'

# A couple more egress opinions (should all be the same EIP):
kubectl -n egress-test exec deploy/netshoot -- curl -s https://checkip.amazonaws.com
kubectl -n egress-test exec deploy/netshoot -- dig +short myip.opendns.com @resolver1.opendns.com
```

## Teardown

```bash
# Option A (delete exactly what you rendered — also removes the cluster-scoped
# NodeClass/NodePool, which a namespace delete would NOT):
helm template egress-test . --set subnet.id=subnet-XXXXXXXX | kubectl delete -f -

# Option B: set egressTest: false in ../apps/values.yaml and let Argo prune.
```

The node is consolidated away by Auto Mode shortly after the pod is gone.

## Gotchas

- **Exactly one subnet.** `subnetSelectorTerms` MUST resolve to a single subnet or
  provisioning is ambiguous/fails. Prefer `subnet.id`, or a tag unique to one
  subnet. (Security groups may match more than one — that's fine.)
- **Reuse the Auto Mode node role.** Its EKS access entry (type EC2 +
  `AmazonEKSAutoNodePolicy`) already exists, so custom-NodeClass nodes join with no
  extra setup. A **new** role would need its own EC2-type access entry + that
  policy, or the nodes never register.
- **`general-purpose` is untainted.** The built-in NodePool has no taint, so the
  pod's **`nodeSelector: client=<name>`** is what keeps it off general-purpose
  nodes; the NodePool's **taint** keeps unrelated pods off yours. You need both.
- **Don't disable SNAT.** Leave `snatPolicy` at the default (`Random`). Setting it
  to `Disabled` stops pod egress from using the node's subnet/NAT path and breaks
  the test.
- **netshoot must stay alive.** A bare pod's shell exits → `CrashLoopBackOff`; the
  Deployment uses `command: ["sleep","infinity"]`. Outbound `curl` needs no extra
  capabilities, so the pod runs non-root with all capabilities dropped.
</content>
