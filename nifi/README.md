# nifi — Apache NiFi 2.7.2 standalone (raw manifests)

Runs a **single, non-clustered** Apache NiFi 2.7.2 node on the
`tituseff-playground` EKS **Auto Mode** cluster from the official
[`apache/nifi`](https://hub.docker.com/r/apache/nifi) image:

- one **StatefulSet** pod (`nifi-0`) with single-user auth and NiFi's own
  self-signed HTTPS cert on `:8443`,
- conf/, state/ and **all repositories** on **one encrypted gp3 EBS volume**
  (`reclaimPolicy: Retain`),
- reachable via `kubectl port-forward` always, and via an **internet-facing ALB**
  (ACM cert on the ALB, HTTPS re-encrypt to the pod) once the Ingress is enabled.

> **WARNING — plaintext credentials in git (playground only).** The NiFi login
> and the sensitive-properties key live in clear text in `values.yaml` and are
> rendered into the `nifi-credentials` Secret. This was a deliberate choice for
> this playground. Do not reuse this pattern anywhere real.

## What it renders

| Kind | Name | Scope | Notes |
| --- | --- | --- | --- |
| `Namespace` | `nifi` (`.Values.namespace`) | cluster | Pod Security `restricted` (enforce/warn/audit) |
| `StorageClass` | `nifi-gp3` (`.Values.storage.storageClass.name`) | cluster | `ebs.csi.eks.amazonaws.com`, gp3, encrypted, Retain, WaitForFirstConsumer, expandable, Auto Mode nodes only |
| `Secret` | `nifi-credentials` | namespaced | `username`, `password`, `sensitivePropsKey` (plaintext from values) |
| `Service` | `nifi-headless` | namespaced | headless (`clusterIP: None`), StatefulSet governing service, 8443 |
| `Service` | `nifi` | namespaced | ClusterIP, `https` 8443 → pod `https` |
| `StatefulSet` | `nifi` | namespaced | 1 replica, `seed-conf` initContainer, PVC `data` (100Gi) via `volumeClaimTemplates` |
| `PersistentVolumeClaim` | `data-nifi-0` | namespaced | created by the StatefulSet, not rendered directly; retained on delete/scale |
| `Ingress` | `nifi` | namespaced | **only if `ingress.enabled`**; `alb` IngressClass, HTTPS:443 + HTTP→HTTPS redirect |

No `PodDisruptionBudget`: with a single replica, a PDB that protects the pod
(`minAvailable: 1`) can never be satisfied during a drain, so it would block Auto
Mode node drains/consolidation (and node expiry) indefinitely. A brief NiFi
outage during a node replacement is accepted instead.

## Configure

Edit `values.yaml` (every key is documented inline). The ones you'll usually set:

| Value | Purpose | Default |
| --- | --- | --- |
| `credentials.username` | single-user login (≥ 4 chars) | `admin` (**placeholder**) |
| `credentials.password` | single-user password (≥ 12 chars) | `ChangeMe-NiFi-Pass1` (**placeholder**) |
| `credentials.sensitivePropsKey` | encrypts sensitive flow properties (≥ 12 chars, **alphanumeric only**, never change after first start) | `ChangeMeSensitiveKey123` (**placeholder**) |
| `host` | public FQDN; drives `NIFI_WEB_PROXY_HOST` + cert SANs + Ingress host — **set before first start** | `nifi.titusefferian.dev` |
| `image.tag` | NiFi version (never `latest`) | `2.7.2` |
| `jvm.heapInit` / `jvm.heapMax` | JVM heap | `2g` / `2g` |
| `resources` | container requests/limits (no CPU limit) | req `cpu: 1`, `memory: 4Gi`; limit `memory: 4Gi` |
| `storage.size` | data volume size | `100Gi` |
| `storage.storageClass.name` | StorageClass rendered + used | `nifi-gp3` |
| `storage.storageClass.kmsKeyId` | optional CMK ARN for EBS encryption | `""` (aws/ebs key) |
| `ingress.enabled` | render the ALB Ingress | `false` |
| `ingress.certificateArn` | ACM cert ARN — **required** when the Ingress is enabled (render fails otherwise) | `""` |
| `ingress.loadBalancerName` | fixed ALB name (≤ 32 chars); must match Terraform `nifi_alb_name` in the EKS repo. Only applied when the Ingress is created — changing it later requires deleting and recreating the Ingress (which creates a new ALB) | `tituseff-playground-nifi` |
| `ingress.inboundCidrs` | source CIDRs allowed to reach the ALB — narrow this (e.g. your IP `/32`) | `0.0.0.0/0` |
| `ingress.sslPolicy` | ALB TLS policy | `ELBSecurityPolicy-TLS13-1-2-2021-06` |

**Credential gotchas.**
- Too-short credentials fail **silently**: the pod starts, but NiFi does not
  apply them. Always check `kubectl -n nifi logs nifi-0 -c nifi | grep ERROR:`.
- `sensitivePropsKey` must be alphanumeric: start.sh writes it into
  `nifi.properties` with `sed` using `|` as delimiter, so `|`, `&` or `\`
  corrupt the file.
- **Never change `sensitivePropsKey` after the first start** — sensitive values
  already stored in the flow become undecryptable.
- Changing a credential in `values.yaml` rolls the pod automatically (the pod
  template carries a `checksum/secret` annotation; env vars are read only at start).

## Prerequisites

1. **Change the placeholder credentials** in `values.yaml` before enabling the
   chart (see the rules above).
2. **ACM certificate** (only for the ALB) — managed by the EKS Terraform repo
   ([TitusEfferian/EKS#14](https://github.com/TitusEfferian/EKS/pull/14)). Stage 1:
   apply with `enable_nifi_ingress = true` to create the DNS-validated cert for
   `nifi.titusefferian.dev`, then copy `terraform output nifi_acm_certificate_arn`
   into `ingress.certificateArn` and set `ingress.enabled: true`.
3. **Route53 alias record** (only for the ALB). There is **no ExternalDNS** in the
   cluster; once the ALB exists, Stage 2 in the same Terraform repo
   (`enable_nifi_dns = true`, `nifi_alb_name = "tituseff-playground-nifi"`) looks
   up the ALB by `ingress.loadBalancerName` and creates the alias **A** record
   `nifi.titusefferian.dev` → ALB.

## Install

GitOps (the normal path): `nifi: true` is the default in `../apps/values.yaml`,
so once this is on the tracked branch the `infra-root` app renders the `nifi`
child Application (sync wave 2, after the ALB controller) and Argo CD syncs this
chart.

First boot takes a few minutes (keystore generation, NAR unpacking); the
startupProbe allows up to 10 minutes.

Render locally (no cluster needed):

```bash
helm lint nifi
helm template nifi nifi
helm template nifi nifi --set ingress.enabled=true \
  --set ingress.certificateArn=arn:aws:acm:ap-northeast-1:<acct>:certificate/<id>
```

## Access

**Port-forward** (always works, no ALB/DNS needed):

```bash
kubectl -n nifi port-forward svc/nifi 8443:8443
# then open https://localhost:8443/nifi  (accept the self-signed cert warning)
```

The browser sends `Host: localhost:8443`, which NiFi accepts: `localhost` is a SAN
on the generated cert and `8443` is its HTTPS port.

**ALB** (after the prerequisites): `https://nifi.titusefferian.dev/nifi`. HTTP:80
redirects to HTTPS:443. The ALB re-encrypts to the pod on 8443.

## Verify

```bash
kubectl -n nifi get sts,pod,pvc,svc,ingress
kubectl -n nifi logs nifi-0 -c nifi | grep -E "ERROR|NIFI_WEB_PROXY_HOST|Started Application"
curl -k https://nifi.titusefferian.dev/nifi-api/authentication/configuration
```

ALB target health: the health check hits `/nifi-api/authentication/configuration`
with `Host=<target IP>`, which is not a SAN on NiFi's cert, so NiFi answers `400`;
`success-codes: "200,400"` treats that as healthy (a 400 from Jetty still proves
NiFi is up).

## Caveats

- **Plaintext credentials in git** — playground only (see the warning above).
- **Self-signed backend cert: 60 days, never auto-renews.** The ALB does not
  validate target certs, so the ALB path keeps working after expiry (port-forward
  users just see an expired-cert warning). To regenerate: delete
  `conf/keystore.p12` and `conf/truststore.p12` on the volume, blank
  `nifi.security.keystorePasswd`, `nifi.security.keyPasswd` and
  `nifi.security.truststorePasswd` in `conf/nifi.properties`, then restart the pod.
- **Changing `host` after first start** requires the same cert regeneration — the
  SANs are fixed when the cert is first generated.
- **EBS is AZ-pinned.** The volume lives in the AZ where `nifi-0` first scheduled;
  the pod can only ever run in that AZ. If that AZ is unavailable, recover by
  snapshotting the volume and restoring it in another AZ.
- **`reclaimPolicy: Retain`.** Deleting the PVC (or the app) leaves the EBS volume
  and its PV behind — delete them manually when you really mean it
  (`kubectl delete pv <pv>` then delete the volume in EC2).
- **Upgrading NiFi.** The seeded `conf/` stays at the version that first started;
  after bumping `image.tag`, diff the new image's default `conf/` against the
  volume and merge changes manually.
- **One shared volume for every repository.** Content and provenance compete for
  the same gp3 IOPS/throughput and space; split them onto their own PVCs later if
  load requires it.
