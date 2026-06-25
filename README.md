# SRE Builds of Kubernetes environments via IaC (Terraform/OpenTofu)

## Overview

The infrastructure is split into four independent Terraform projects. Each project manages its own state and communicates with downstream projects exclusively through **AWS SSM Parameter Store** — no shared state files, no remote state data sources.

```
01-foundation          →  SSM  →  02-infra-EKS-Cluster
      ↓                                     (independent)
90-godaddy-ns-update             03-infra-Manual-K8s-Cluster
```

---

## Execution Order

| Step | Project | Must run after | Notes |
|------|---------|----------------|-------|
| 1 | `01-foundation` | Nothing — this is the base layer | Creates AWS pre-requisite resources and services |
| 2 | `90-godaddy-ns-update` | `01-foundation` (needs the Route 53 nameservers) | Uses terraform for scripted updates or it can be done on Go-Daddy DNS settings page |
| 3 | `02-infra-EKS-Cluster` | `01-foundation` + NS delegation confirmed | Builds the AWS EKS Cluster |
| 4 | `03-infra-Manual-K8s-Cluster` | `01-foundation` + NS delegation confirmed | Builds the infra for manual EKS build |

Steps 2 and 3 are independent of each other — `02` and `03` can run in either order or in parallel after `01` and `90` complete. `90-godaddy-ns-update` must run before either infra project because ACM certificate DNS validation will fail if Route 53 is not yet authoritative for the domain.

---

## Dependency Map

```
         ┌──────────────────────────┐
         │      01-foundation       │
         │  Route 53 zones + ACM    │
         │  Publishes to SSM:       │
         │  • public_zone_id        │
         │  • private_zone_id       │
         │  • wildcard_cert_arn     │
         │  • cloudfront_cert_arn   │
         │  • nameservers           │
         └──────────┬───────────────┘
                    │ nameservers output
                    ▼
         ┌──────────────────────────┐
         │   90-godaddy-ns-update   │
         │  Pushes NS records to    │
         │  GoDaddy via REST API    │
         │  Polls until propagated  │
         └──────────┬───────────────┘
                    │ DNS now authoritative → ACM validation passes
          ┌─────────┴──────────┐
          ▼                    ▼
┌──────────────────┐  ┌──────────────────────────┐
│ 02-infra-EKS-    │  │ 03-infra-Manual-K8s-     │
│    Cluster       │  │      Cluster             │
│  Reads from SSM: │  │  Reads from SSM:         │
│  • public_zone   │  │  • private_zone_id       │
│  • private_zone  │  │                          │
│  • wildcard_cert │  │  EC2 control plane +     │
│  • cf_cert       │  │  workers + NLB + DNS     │
│                  │  └──────────────────────────┘
│  EKS cluster +   │
│  Helm releases + │
│  CloudFront      │
└──────────────────┘
```

---

## Project Details

### `01-foundation`

**What it does**

Creates the shared DNS and TLS layer that every other project depends on. Specifically:

- **Route 53 public hosted zone** — `bpmbi.us`. After apply, the four NS records output by this project must be pointed to from GoDaddy.
- **Route 53 private hosted zone** — `dev.internal.bpmbi.us`. VPC-scoped; used for cluster-internal DNS (control plane nodes, etcd peers, NLB endpoints).
- **ACM wildcard certificate** (primary) — covers `*.bpmbi.us`, `bpmbi.us`, `*.dev.bpmbi.us`, `*.dev.apps.bpmbi.us`. Attached to ALBs and EKS ingress.
- **ACM wildcard certificate** (CloudFront) — same SANs but issued via the `aws.us_east_1` provider alias (CloudFront requires certs in us-east-1).
- **SSM Parameters** — publishes all IDs and ARNs to `/bpmbi/dev/dns/*` and `/bpmbi/dev/acm/*` so downstream projects can consume them without touching this state file.

**Providers**

| Provider | Version | Why |
|----------|---------|-----|
| `hashicorp/aws` | `~> 5.40` | Primary region resources |
| `hashicorp/aws` (alias `us_east_1`) | `~> 5.40` | ACM cert for CloudFront must be in us-east-1 |

**Prerequisites**

- AWS credentials with IAM permissions for Route 53, ACM, SSM.
- An existing VPC (its ID is required to associate the private hosted zone).
- The domain `bpmbi.us` registered at GoDaddy (or another registrar — the 90-project handles the NS update).

**`terraform.tfvars` inputs**

| Variable | Description | Example |
|----------|-------------|---------|
| `domain_name` | Root domain | `bpmbi.us` |
| `environment` | `dev` or `prod` | `dev` |
| `project_name` | Tag/SSM path prefix | `bpmbi` |
| `aws_region` | Deployment region | `us-east-1` |
| `vpc_id` | VPC for private hosted zone | `vpc-0abc...` |
| `eks_subdomain` | Subdomain label for EKS | `eks` |
| `apps_subdomain` | Subdomain for ExternalDNS app records | `apps` |
| `k8s_subdomain` | Subdomain for manual K8s cluster | `k8s` |
| `api_subdomain` | Subdomain for API gateway | `api` |
| `monitor_subdomain` | Subdomain for monitoring (Grafana) | `monitor` |
| `internal_subdomain` | Label for private zone | `internal` |

**Apply command**

```bash
cd terraform/01-foundation
terraform init
terraform apply
```

ACM certificate DNS validation waits up to 90 minutes. In practice it completes in 2–10 minutes once NS delegation is in place. If running for the first time, ACM cannot validate until GoDaddy's nameservers are updated — run `90-godaddy-ns-update` immediately after this apply and let propagation complete before certs validate.

**Key outputs**

| Output | Description |
|--------|-------------|
| `godaddy_nameservers` | The four NS records to paste into GoDaddy |
| `public_zone_id` | Route 53 public zone ID |
| `private_zone_id` | Route 53 private zone ID |
| `wildcard_cert_arn` | ACM cert ARN for ALBs |
| `cloudfront_cert_arn` | ACM cert ARN for CloudFront |
| `ssm_paths` | All SSM paths written by this project |

**SSM parameters written**

```
/bpmbi/dev/dns/public_zone_id
/bpmbi/dev/dns/private_zone_id
/bpmbi/dev/dns/nameservers
/bpmbi/dev/dns/fqdn_internal
/bpmbi/dev/dns/fqdn_eks
/bpmbi/dev/dns/fqdn_monitor
/bpmbi/dev/acm/wildcard_cert_arn
/bpmbi/dev/acm/cloudfront_cert_arn
```

**Important notes**

- The Route 53 public zone has `prevent_destroy = true` commented in the code (re-enable after first successful apply). Destroying the public zone assigns new nameservers, requiring another GoDaddy update and up to 48 hours of propagation.
- This project is intended to be applied once and left running. Destroy only for a full environment teardown.

---

### `90-godaddy-ns-update`

**What it does**

Automates the manual step of updating GoDaddy's nameserver records to point at Route 53. Without this, the domain remains authoritative through GoDaddy's default nameservers and ACM certificate DNS validation will fail.

- Calls `PATCH /v1/domains/bpmbi.us` on the GoDaddy REST API with the four Route 53 nameservers.
- Polls `dig NS bpmbi.us @8.8.8.8` every 60 seconds for up to 60 minutes, exiting as soon as `awsdns` appears in the response.
- Only re-runs when nameservers or the domain name change (via `null_resource` triggers).

**Providers**

| Provider | Version | Why |
|----------|---------|-----|
| `hashicorp/null` | `~> 3.2` | `null_resource` + `local-exec` for curl |

**Prerequisites**

- `01-foundation` applied — you need the four nameserver values from its output.
- GoDaddy API key + secret from [developer.godaddy.com/keys](https://developer.godaddy.com/keys) — use **Production** keys, not OTE/Test.
- `curl` and `dig` available on the machine running Terraform.
- The GoDaddy account must own the domain (`bpmbi.us`).

**`terraform.tfvars` inputs**

| Variable | Description | How to get it |
|----------|-------------|---------------|
| `domain_name` | Domain to update | `bpmbi.us` |
| `nameservers` | List of exactly 4 NS values | `terraform -chdir=../01-foundation output godaddy_nameservers` |
| `godaddy_api_key` | GoDaddy API key (sensitive) | developer.godaddy.com/keys |
| `godaddy_api_secret` | GoDaddy API secret (sensitive) | developer.godaddy.com/keys |

**Apply command**

```bash
# Get nameservers from 01-foundation first
terraform -chdir=../01-foundation output godaddy_nameservers

cd terraform/90-godaddy-ns-update
terraform init
terraform apply
```

**Key outputs**

| Output | Description |
|--------|-------------|
| `nameservers_set` | The NS values pushed to GoDaddy |
| `verify_commands` | `dig` and dnschecker.org commands to confirm propagation |

**Helper script**

```bash
# Check propagation status (no API key needed for live DNS check)
./check-nameservers.sh

# With GoDaddy API to also see what GoDaddy has stored
GODADDY_API_KEY=xxx GODADDY_API_SECRET=yyy ./check-nameservers.sh
```

**Important notes**

- Keep `terraform.tfvars` out of git — it contains API credentials. The `.gitignore` should exclude it.
- The `null_resource` only re-runs when the `triggers` map changes. If you need to force a re-run: `terraform destroy && terraform apply` or `terraform taint null_resource.godaddy_ns_update`.
- GoDaddy propagation typically takes 5–30 minutes but can take up to 48 hours in rare cases.

---

### `02-infra-EKS-Cluster`

**What it does**

Provisions the AWS-managed Kubernetes (EKS) cluster and all supporting infrastructure:

- **EKS control plane** — managed by AWS; version configurable, defaults to 1.32.
- **Managed node group** — EC2 worker nodes with configurable instance type and count (min/desired/max for cluster autoscaler).
- **IRSA roles** — IAM Roles for Service Accounts for AWS Load Balancer Controller and ExternalDNS.
- **Helm releases** (Phase 2):
  - `aws-load-balancer-controller` — creates ALBs from `Ingress` resources.
  - `external-dns` — creates Route 53 records from `Service` and `Ingress` annotations.
- **CloudFront distribution** (optional, `enable_cloudfront = true`) — puts WAF + CloudFront in front of `monitor.dev.bpmbi.us`.

DNS zone IDs and cert ARNs are **not** declared as variables — they are read at runtime from SSM parameters written by `01-foundation`.

**Providers**

| Provider | Version | Why |
|----------|---------|-----|
| `hashicorp/aws` | `~> 5.40` | EKS, IAM, CloudFront, WAF |
| `hashicorp/helm` | `~> 2.12` | Helm chart deployments (Phase 2) |
| `hashicorp/kubernetes` | `~> 2.25` | K8s resource management |
| `hashicorp/tls` | `~> 4.0` | OIDC thumbprint for IRSA |

**Prerequisites**

- `01-foundation` applied and SSM parameters present.
- `90-godaddy-ns-update` applied and DNS propagation confirmed — ACM certs must be `ISSUED` before Ingress/ExternalDNS can function.
- An existing VPC with public and private subnets (at least one per AZ recommended for HA).
- AWS credentials with permissions for EKS, EC2, IAM, Route 53, SSM, WAFv2, CloudFront.

**`terraform.tfvars` inputs**

| Variable | Description | Example |
|----------|-------------|---------|
| `domain_name` | Must match `01-foundation` | `bpmbi.us` |
| `environment` | Must match `01-foundation` | `dev` |
| `project_name` | Must match `01-foundation` | `bpmbi` |
| `aws_region` | Deployment region | `us-east-1` |
| `vpc_id` | VPC for node group | `vpc-0abc...` |
| `vpc_cidr` | VPC CIDR | `10.0.0.0/16` |
| `public_subnet_ids` | Subnets for internet-facing ALBs | `["subnet-aaa", ...]` |
| `private_subnet_ids` | Subnets for EKS node group | `["subnet-bbb", ...]` |
| `eks_cluster_version` | Kubernetes version | `1.32` |
| `eks_node_instance_type` | Worker node EC2 type | `t3.large` |
| `eks_node_min_size` | Min nodes | `2` |
| `eks_node_desired_size` | Initial node count | `3` |
| `eks_node_max_size` | Max nodes (autoscaler) | `10` |
| `eks_node_disk_size_gb` | EBS root volume per node | `50` |
| `enable_cloudfront` | Enable CloudFront + WAF | `false` |
| `cloudfront_price_class` | Edge location coverage | `PriceClass_100` |

Subdomain variables (`eks_subdomain`, `apps_subdomain`, `monitor_subdomain`, `internal_subdomain`) must match the values used in `01-foundation`.

**Two-phase apply (required)**

The Helm and Kubernetes providers need a live cluster endpoint to initialize. The cluster doesn't exist until Phase 1 completes, so a single `terraform apply` fails at Phase 2.

```bash
cd terraform/02-infra-EKS-Cluster

# Phase 1 — create the EKS cluster and node group
terraform apply -target=module.eks

# Between phases — update local kubeconfig
aws eks update-kubeconfig --region us-east-1 --name bpmbi-eks-dev

# Verify nodes are ready
kubectl get nodes

# Phase 2 — deploy Helm releases and CloudFront
terraform apply
```

**Key outputs**

| Output | Description |
|--------|-------------|
| `eks_cluster_name` | Cluster name |
| `eks_cluster_endpoint` | API server URL |
| `kubeconfig_command` | Exact command to update `~/.kube/config` |
| `eks_external_dns_role_arn` | IRSA role ARN for ExternalDNS |
| `eks_lb_controller_role_arn` | IRSA role ARN for LB controller |
| `dns_endpoints` | Public HTTPS endpoints provisioned |
| `cloudfront_monitor_domain` | CloudFront domain (if enabled) |

**DNS endpoints created**

| Endpoint | Covers |
|----------|--------|
| `eks.dev.bpmbi.us` | EKS ingress (covered by `*.dev.bpmbi.us` cert SAN) |
| `monitor.dev.bpmbi.us` | Grafana / monitoring (CloudFront alias) |
| `*.dev.apps.bpmbi.us` | ExternalDNS-managed app records |

**Important notes**

- Subdomain labels must exactly match `01-foundation`. The SSM path `/bpmbi/dev/...` is constructed from `project_name` and `environment`.
- `enable_cloudfront = false` by default to avoid WAF costs during development. Enable for production or when a Grafana ALB exists.
- To scale the node group: change `eks_node_desired_size` and `terraform apply`. No cluster rebuild required.

---

### `03-infra-Manual-K8s-Cluster`

**What it does**

Provisions a self-managed Kubernetes cluster on EC2 using `kubeadm` with stacked etcd topology. This cluster runs alongside or independently of the EKS cluster — useful for learning kubeadm, testing bare-metal K8s scenarios, or workloads that need full control-plane access.

- **Control plane EC2 instances** — 1, 3, or 5 nodes (must be odd for etcd quorum). Runs API server, controller-manager, scheduler, and etcd.
- **Worker EC2 instances** — 1 to 20 nodes, configurable instance type and disk size.
- **Internal NLB** — load balances the Kubernetes API server across all control plane nodes. Used as `--control-plane-endpoint` in kubeadm.
- **Route 53 private DNS records** — per-node A records in `dev.internal.bpmbi.us` for stable cluster-internal addressing (`cp-1.dev.internal.bpmbi.us`, `etcd-1.dev.internal.bpmbi.us`, `worker-1.nodes.dev.internal.bpmbi.us`).
- **Security groups** — intra-cluster rules (etcd peer ports, API server, kubelet, NodePort range) plus optional SSH from allowed CIDRs.
- **IAM instance profile** — allows nodes to call SSM Session Manager (no bastion needed) and EC2 metadata.

Terraform only provisions the infrastructure. The Kubernetes software itself (kubeadm init/join) is a manual step run after apply.

**Providers**

| Provider | Version | Why |
|----------|---------|-----|
| `hashicorp/aws` | `~> 5.40` | EC2, NLB, Route 53, IAM, SSM |

**Prerequisites**

- `01-foundation` applied — private zone ID is read from SSM.
- An existing VPC with private subnets for the EC2 nodes.
- An EC2 key pair in the target region (for SSH access; SSM Session Manager is also available without a key).
- AWS credentials with permissions for EC2, IAM, Route 53, SSM, ELB.
- `session-manager-plugin` installed locally for SSM-based shell access:
  ```bash
  brew install --cask session-manager-plugin
  ```

**`terraform.tfvars` inputs**

| Variable | Description | Example |
|----------|-------------|---------|
| `domain_name` | Must match `01-foundation` | `bpmbi.us` |
| `environment` | Must match `01-foundation` | `dev` |
| `project_name` | Must match `01-foundation` | `bpmbi` |
| `aws_region` | Deployment region | `us-east-1` |
| `vpc_id` | VPC for instances | `vpc-0abc...` |
| `vpc_cidr` | VPC CIDR for SG rules | `10.0.0.0/16` |
| `public_subnet_ids` | Reserved for future public NLB | `["subnet-aaa", ...]` |
| `private_subnet_ids` | Subnets for CP and worker nodes | `["subnet-bbb", ...]` |
| `k8s_cp_instance_type` | Control plane EC2 type | `t3.medium` |
| `k8s_cp_node_count` | Number of CP nodes (1, 3, or 5) | `3` |
| `k8s_cp_volume_size_gb` | EBS root volume per CP node | `50` |
| `k8s_worker_instance_type` | Worker EC2 type | `t3.large` |
| `k8s_worker_node_count` | Number of workers (1–20) | `3` |
| `k8s_worker_volume_size_gb` | EBS root volume per worker | `100` |
| `k8s_ssh_key_name` | EC2 key pair name | `my-key` |
| `k8s_ami_id` | Override AMI (leave empty for latest Ubuntu 22.04 LTS) | `""` |
| `k8s_allowed_ssh_cidrs` | CIDRs allowed to SSH | `["10.0.0.0/8"]` |

**Apply command**

```bash
cd terraform/03-infra-Manual-K8s-Cluster
terraform init
terraform apply
```

**Post-apply: kubeadm bootstrap**

After Terraform apply, SSH or SSM into `cp-1` and run the `kubeadm init` command shown in the outputs:

```bash
# Connect via SSM (no key pair needed)
aws ssm start-session --target <cp-1-instance-id> --region us-east-1

# On cp-1: initialise the cluster
sudo kubeadm init \
  --control-plane-endpoint "<NLB_FQDN>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16

# Copy kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Install CNI (Calico example)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Join additional CP nodes (use --certificate-key from kubeadm init output)
# Join workers (use standard join command from kubeadm init output)
```

**Key outputs**

| Output | Description |
|--------|-------------|
| `kubeadm_init_command` | Full `kubeadm init` command with correct endpoint |
| `cp_endpoint_fqdn` | Private Route 53 FQDN for the NLB (use as `--control-plane-endpoint`) |
| `cp_nlb_dns` | Raw NLB DNS name |
| `cp_private_ips` | Map of CP node name → private IP |
| `worker_private_ips` | Map of worker name → private IP |
| `cp_instance_ids` | Map of CP node name → instance ID |
| `ssm_connect_commands` | Ready-to-run `aws ssm start-session` commands per CP node |

**DNS records created (private zone)**

```
cp-1.dev.internal.bpmbi.us      → CP node 1 private IP
cp-2.dev.internal.bpmbi.us      → CP node 2 private IP
cp-3.dev.internal.bpmbi.us      → CP node 3 private IP
etcd-1.dev.internal.bpmbi.us    → etcd member 1 (same IPs as CP)
worker-1.nodes.dev.internal.bpmbi.us  → worker 1 private IP
api.k8s.dev.internal.bpmbi.us   → NLB CNAME (control plane endpoint)
```

**Important notes**

- `k8s_cp_node_count` must be 1, 3, or 5 — etcd requires an odd number for leader election. Changing this value after initial bootstrap requires careful etcd member management.
- `k8s_worker_node_count` can be changed freely with `terraform apply` — existing workers are not replaced (Terraform will add or remove instances).
- SSH is restricted to `k8s_allowed_ssh_cidrs` (default: VPC CIDR only). SSM Session Manager works without any key pair — install `session-manager-plugin` via `brew install --cask session-manager-plugin`.
- This cluster has no automatic node replacement. If a node fails, Terraform can recreate it but kubeadm join must be re-run manually.

---

## Common Variables Across All Projects

These variables must be consistent across all projects — they are used to construct SSM paths and subdomain names:

| Variable | Default | Used in |
|----------|---------|---------|
| `domain_name` | `bpmbi.us` | All projects |
| `environment` | `dev` | All projects |
| `project_name` | `bpmbi` | All projects |
| `aws_region` | `us-east-1` | All projects |
| `eks_subdomain` | `eks` | 01, 02 |
| `apps_subdomain` | `apps` | 01, 02 |
| `k8s_subdomain` | `k8s` | 01, 03 |
| `monitor_subdomain` | `monitor` | 01, 02 |
| `internal_subdomain` | `internal` | 01, 02, 03 |

If any of these differ between projects, SSM lookups will fail or DNS records will not resolve correctly.

---

## Destroy Order

To tear down the full environment, destroy in reverse order:

```bash
# 1. Destroy infra layers first (order between 02 and 03 doesn't matter)
terraform -chdir=terraform/02-infra-EKS-Cluster destroy
terraform -chdir=terraform/03-infra-Manual-K8s-Cluster destroy

# 2. Destroy GoDaddy config (optional — nameservers stay until 01 is destroyed)
terraform -chdir=terraform/90-godaddy-ns-update destroy

# 3. Destroy foundation last
# NOTE: If prevent_destroy is enabled on Route 53 zones, comment it out first
terraform -chdir=terraform/01-foundation destroy
```

After destroying `01-foundation`, Route 53 will assign new nameservers if the zone is recreated. Re-run `90-godaddy-ns-update` with the new values.

---

## ACM Certificate SAN Coverage

Understanding which FQDNs are covered by the ACM certs avoids `InvalidViewerCertificate` errors:

| SAN | Covers | Used by |
|-----|--------|---------|
| `*.bpmbi.us` | `X.bpmbi.us` (one level only) | Top-level subdomains |
| `bpmbi.us` | Bare domain | Redirect |
| `*.dev.bpmbi.us` | `X.dev.bpmbi.us` — e.g. `monitor.dev.bpmbi.us`, `eks.dev.bpmbi.us` | Service endpoints |
| `*.dev.apps.bpmbi.us` | `X.dev.apps.bpmbi.us` | ExternalDNS app records |

Service FQDNs follow the pattern `service.env.domain` (e.g. `monitor.dev.bpmbi.us`) not `env.service.domain`. Two-level subdomains like `dev.monitor.bpmbi.us` are not covered by any wildcard in the cert.

