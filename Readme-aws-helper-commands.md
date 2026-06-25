# AWS Debugging & Validation Commands — bpmbi.us

Quick reference for validating each Terraform project's resources after apply.
All commands assume `--region us-east-1` unless noted. Replace placeholder values (`vpc-xxx`, `i-xxx`, etc.) with your actual resource IDs.

---

## Table of Contents

- [Environment Setup](#environment-setup)
- [SSM Parameter Store](#ssm-parameter-store) — bridge between all projects
- [01-foundation — Route 53](#01-foundation--route-53)
- [01-foundation — ACM Certificates](#01-foundation--acm-certificates)
- [90-godaddy-ns-update — DNS Propagation](#90-godaddy-ns-update--dns-propagation)
- [02-infra-EKS-Cluster — EKS](#02-infra-eks-cluster--eks)
- [02-infra-EKS-Cluster — Helm / K8s](#02-infra-eks-cluster--helm--k8s)
- [02-infra-EKS-Cluster — CloudFront & WAF](#02-infra-eks-cluster--cloudfront--waf)
- [03-infra-Manual-K8s-Cluster — EC2 Nodes](#03-infra-manual-k8s-cluster--ec2-nodes)
- [03-infra-Manual-K8s-Cluster — NLB](#03-infra-manual-k8s-cluster--nlb)
- [03-infra-Manual-K8s-Cluster — SSM Sessions](#03-infra-manual-k8s-cluster--ssm-sessions)
- [VPC & Networking](#vpc--networking)
- [IAM Roles & Policies](#iam-roles--policies)
- [End-to-End Health Check](#end-to-end-health-check)

---

## Environment Setup

```bash
# Set common variables — paste once per terminal session
export AWS_REGION=us-east-1
export DOMAIN=bpmbi.us
export ENV=dev
export PROJECT=bpmbi
export SSM_PREFIX="/${PROJECT}/${ENV}"

# Confirm active identity
aws sts get-caller-identity

# Confirm region
aws configure get region
```

---

## SSM Parameter Store

SSM is the bridge between all projects. Validate it first — if parameters are missing, downstream projects will fail on `data "aws_ssm_parameter"` lookups.

```bash
# List ALL parameters written by 01-foundation
aws ssm get-parameters-by-path \
  --path "${SSM_PREFIX}" \
  --recursive \
  --query "Parameters[*].{Name:Name,Value:Value}" \
  --output table

# Get individual parameters
aws ssm get-parameter --name "${SSM_PREFIX}/dns/public_zone_id"  --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/dns/private_zone_id" --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/dns/nameservers"     --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/acm/wildcard_cert_arn"   --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/acm/cloudfront_cert_arn" --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/dns/fqdn_eks"     --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/dns/fqdn_monitor" --query "Parameter.Value" --output text
aws ssm get-parameter --name "${SSM_PREFIX}/dns/fqdn_internal"--query "Parameter.Value" --output text
```

Expected: 9 parameters, all with non-empty values. If any are missing, `01-foundation` did not complete successfully.

---

## 01-foundation — Route 53

```bash
# List all hosted zones (shows both public and private)
aws route53 list-hosted-zones \
  --query "HostedZones[*].{ID:Id,Name:Name,Private:Config.PrivateZone,Records:ResourceRecordSetCount}" \
  --output table

# Get the public zone ID
PUBLIC_ZONE_ID=$(aws ssm get-parameter \
  --name "${SSM_PREFIX}/dns/public_zone_id" \
  --query "Parameter.Value" --output text)
echo "Public zone: $PUBLIC_ZONE_ID"

# Get the private zone ID
PRIVATE_ZONE_ID=$(aws ssm get-parameter \
  --name "${SSM_PREFIX}/dns/private_zone_id" \
  --query "Parameter.Value" --output text)
echo "Private zone: $PRIVATE_ZONE_ID"

# List all records in the public zone (should include NS, SOA, and ACM validation CNAMEs)
aws route53 list-resource-record-sets \
  --hosted-zone-id "$PUBLIC_ZONE_ID" \
  --query "ResourceRecordSets[*].{Name:Name,Type:Type,TTL:TTL}" \
  --output table

# List all records in the private zone (node A records added by 03-project)
aws route53 list-resource-record-sets \
  --hosted-zone-id "$PRIVATE_ZONE_ID" \
  --query "ResourceRecordSets[*].{Name:Name,Type:Type,TTL:TTL}" \
  --output table

# Show the NS records for the public zone (these go into GoDaddy)
aws route53 list-resource-record-sets \
  --hosted-zone-id "$PUBLIC_ZONE_ID" \
  --query "ResourceRecordSets[?Type=='NS'].ResourceRecords[*].Value" \
  --output text

# Check zone VPC associations (private zone)
aws route53 get-hosted-zone \
  --id "$PRIVATE_ZONE_ID" \
  --query "VPCs"
```

---

## 01-foundation — ACM Certificates

```bash
# List all ACM certificates
aws acm list-certificates \
  --query "CertificateSummaryList[*].{ARN:CertificateArn,Domain:DomainName,Status:Status}" \
  --output table

# Get wildcard cert ARN from SSM
WILDCARD_CERT=$(aws ssm get-parameter \
  --name "${SSM_PREFIX}/acm/wildcard_cert_arn" \
  --query "Parameter.Value" --output text)

# Get CloudFront cert ARN from SSM
CF_CERT=$(aws ssm get-parameter \
  --name "${SSM_PREFIX}/acm/cloudfront_cert_arn" \
  --query "Parameter.Value" --output text)

# Check wildcard cert status and SANs
aws acm describe-certificate \
  --certificate-arn "$WILDCARD_CERT" \
  --query "Certificate.{Status:Status,Domain:DomainName,SANs:SubjectAlternativeNames,Validation:DomainValidationOptions[*].{Domain:DomainName,Status:ValidationStatus}}" \
  --output json

# Check CloudFront cert (must show Status: ISSUED)
aws acm describe-certificate \
  --certificate-arn "$CF_CERT" \
  --query "Certificate.{Status:Status,Domain:DomainName,SANs:SubjectAlternativeNames}" \
  --output json

# Quick status check — both should show ISSUED
aws acm describe-certificate --certificate-arn "$WILDCARD_CERT" \
  --query "Certificate.Status" --output text

aws acm describe-certificate --certificate-arn "$CF_CERT" \
  --query "Certificate.Status" --output text

# List ACM validation CNAME records in Route 53
aws route53 list-resource-record-sets \
  --hosted-zone-id "$PUBLIC_ZONE_ID" \
  --query "ResourceRecordSets[?Type=='CNAME'].{Name:Name,Value:ResourceRecords[0].Value}" \
  --output table
```

Expected: both certs show `ISSUED`. If `PENDING_VALIDATION`, DNS propagation is not complete — wait and re-check.

---

## 90-godaddy-ns-update — DNS Propagation

```bash
# Check NS records from multiple resolvers
# System resolver
dig NS $DOMAIN +short

# Google
dig NS $DOMAIN @8.8.8.8 +short

# Cloudflare
dig NS $DOMAIN @1.1.1.1 +short

# OpenDNS
dig NS $DOMAIN @208.67.222.222 +short

# All four should return awsdns nameservers. Example:
# ns-1134.awsdns-13.org.
# ns-1753.awsdns-27.co.uk.
# ns-572.awsdns-07.net.
# ns-94.awsdns-11.com.

# Confirm Route 53 is authoritative (SOA query hits AWS)
dig SOA $DOMAIN +short

# Check propagation globally
# Open in browser: https://dnschecker.org/#NS/bpmbi.us

# Validate a specific subdomain resolves through Route 53
dig eks.dev.$DOMAIN @8.8.8.8 +short
dig monitor.dev.$DOMAIN @8.8.8.8 +short

# Use the helper script (from 90-godaddy-ns-update folder)
# cd terraform/90-godaddy-ns-update
# ./check-nameservers.sh
```

---

## 02-infra-EKS-Cluster — EKS

```bash
# List EKS clusters
aws eks list-clusters --output table

# Describe the cluster (status, endpoint, version)
aws eks describe-cluster \
  --name "${PROJECT}-eks-${ENV}" \
  --query "cluster.{Name:name,Status:status,Version:version,Endpoint:endpoint,RoleArn:roleArn}" \
  --output json

# Quick status check — should be ACTIVE
aws eks describe-cluster \
  --name "${PROJECT}-eks-${ENV}" \
  --query "cluster.status" --output text

# List node groups
aws eks list-nodegroups \
  --cluster-name "${PROJECT}-eks-${ENV}" \
  --output table

# Describe node group (instance type, scaling config, status)
aws eks describe-nodegroup \
  --cluster-name "${PROJECT}-eks-${ENV}" \
  --nodegroup-name "${PROJECT}-eks-${ENV}-nodes" \
  --query "nodegroup.{Status:status,DesiredSize:scalingConfig.desiredSize,MinSize:scalingConfig.minSize,MaxSize:scalingConfig.maxSize,InstanceType:instanceTypes[0],AMIType:amiType}" \
  --output json

# Update kubeconfig (run this between Phase 1 and Phase 2)
aws eks update-kubeconfig \
  --region $AWS_REGION \
  --name "${PROJECT}-eks-${ENV}"

# Check EKS OIDC provider (needed for IRSA)
aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[*].Arn" \
  --output table

# List EKS add-ons
aws eks list-addons \
  --cluster-name "${PROJECT}-eks-${ENV}" \
  --output table

# Check EKS access entries (who can access the cluster)
aws eks list-access-entries \
  --cluster-name "${PROJECT}-eks-${ENV}" \
  --output table
```

---

## 02-infra-EKS-Cluster — Helm / K8s

Run after `aws eks update-kubeconfig`.

```bash
# Verify kubectl connectivity
kubectl cluster-info
kubectl get nodes -o wide

# Check node status (all should be Ready)
kubectl get nodes

# List Helm releases across all namespaces
helm list -A

# Check AWS Load Balancer Controller
helm status aws-load-balancer-controller -n kube-system
kubectl get deployment aws-load-balancer-controller -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check ExternalDNS
helm status external-dns -n kube-system
kubectl get deployment external-dns -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns

# Check ExternalDNS logs (look for "Desired change: CREATE" lines after deploying ingress)
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns --tail=50

# Check LB Controller logs (look for reconciliation of Ingress resources)
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=50

# Check IRSA annotations on service accounts
kubectl get sa aws-load-balancer-controller -n kube-system -o jsonpath='{.metadata.annotations}'
kubectl get sa external-dns -n kube-system -o jsonpath='{.metadata.annotations}'

# List all ingress resources (should show ALB addresses once created)
kubectl get ingress -A

# List all services of type LoadBalancer
kubectl get svc -A --field-selector spec.type=LoadBalancer

# Check cluster autoscaler (if deployed)
kubectl get deployment cluster-autoscaler -n kube-system 2>/dev/null || echo "Not deployed"

# System pod health
kubectl get pods -n kube-system

# Events (helpful for diagnosing stuck pods or failed reconciliation)
kubectl get events -n kube-system --sort-by='.lastTimestamp' | tail -20
```

---

## 02-infra-EKS-Cluster — CloudFront & WAF

```bash
# List CloudFront distributions
aws cloudfront list-distributions \
  --query "DistributionList.Items[*].{ID:Id,Domain:DomainName,Aliases:Aliases.Items[0],Status:Status,Enabled:Enabled}" \
  --output table

# Get the distribution by alias
CF_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Aliases.Items[?contains(@,'monitor.${ENV}.${DOMAIN}')]].Id" \
  --output text)
echo "CloudFront ID: $CF_ID"

# Describe distribution (status, origin, cert ARN)
aws cloudfront get-distribution \
  --id "$CF_ID" \
  --query "Distribution.{Status:Status,Domain:DomainName,Origins:DistributionConfig.Origins.Items[*].DomainName,CertARN:DistributionConfig.ViewerCertificate.ACMCertificateArn}" \
  --output json

# Check WAF WebACL (must be in us-east-1 for CloudFront)
aws wafv2 list-web-acls \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --query "WebACLs[*].{Name:Name,ID:Id,ARN:ARN}" \
  --output table

# Get WAF WebACL details (rules, default action)
WAF_ARN=$(aws wafv2 list-web-acls \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --query "WebACLs[?contains(Name,'bpmbi')].ARN" \
  --output text)

aws wafv2 get-web-acl \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --name "${ENV}-bpmbi-cf-waf" \
  --id "$(aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1 --query "WebACLs[?contains(Name,'bpmbi')].Id" --output text)" \
  --query "WebACL.{Name:Name,DefaultAction:DefaultAction,Rules:Rules[*].Name}" \
  --output json

# Test CloudFront endpoint
curl -sv "https://monitor.${ENV}.${DOMAIN}" 2>&1 | grep -E "< HTTP|Server:|x-cache:|x-amz"

# Check Route 53 alias record pointing to CloudFront
aws route53 list-resource-record-sets \
  --hosted-zone-id "$PUBLIC_ZONE_ID" \
  --query "ResourceRecordSets[?Name=='monitor.${ENV}.${DOMAIN}.']" \
  --output json
```

---

## 03-infra-Manual-K8s-Cluster — EC2 Nodes

```bash
# List all EC2 instances tagged for the manual K8s cluster
aws ec2 describe-instances \
  --filters \
    "Name=tag:Project,Values=${PROJECT}" \
    "Name=tag:Environment,Values=${ENV}" \
    "Name=instance-state-name,Values=running,stopped,pending" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name'].Value|[0],State:State.Name,Type:InstanceType,PrivateIP:PrivateIpAddress,AZ:Placement.AvailabilityZone}" \
  --output table

# List control plane nodes specifically
aws ec2 describe-instances \
  --filters \
    "Name=tag:Role,Values=control-plane" \
    "Name=tag:Environment,Values=${ENV}" \
    "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name'].Value|[0],PrivateIP:PrivateIpAddress,State:State.Name}" \
  --output table

# List worker nodes
aws ec2 describe-instances \
  --filters \
    "Name=tag:Role,Values=worker" \
    "Name=tag:Environment,Values=${ENV}" \
    "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name'].Value|[0],PrivateIP:PrivateIpAddress,State:State.Name}" \
  --output table

# Check instance system status (both status checks should be 'ok')
aws ec2 describe-instance-status \
  --filters "Name=tag:Project,Values=${PROJECT}" \
  --query "InstanceStatuses[*].{ID:InstanceId,System:SystemStatus.Status,Instance:InstanceStatus.Status}" \
  --output table

# Get IAM instance profile attached to nodes
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=${PROJECT}" "Name=tag:Environment,Values=${ENV}" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Profile:IamInstanceProfile.Arn}" \
  --output table

# Check security groups for K8s cluster
aws ec2 describe-security-groups \
  --filters \
    "Name=tag:Project,Values=${PROJECT}" \
    "Name=tag:Environment,Values=${ENV}" \
  --query "SecurityGroups[*].{ID:GroupId,Name:GroupName,Description:Description}" \
  --output table

# Inspect inbound rules for control plane SG
CP_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Role,Values=control-plane" "Name=tag:Environment,Values=${ENV}" \
  --query "SecurityGroups[0].GroupId" --output text)

aws ec2 describe-security-groups \
  --group-ids "$CP_SG_ID" \
  --query "SecurityGroups[0].IpPermissions[*].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,CIDR:IpRanges[0].CidrIp}" \
  --output table
```

---

## 03-infra-Manual-K8s-Cluster — NLB

```bash
# List all load balancers tagged for the project
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,State:State.Code,Type:Type,Scheme:Scheme}" \
  --output table

# Find the K8s API NLB
NLB_ARN=$(aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName,'k8s') && Type=='network'].LoadBalancerArn" \
  --output text)
echo "NLB ARN: $NLB_ARN"

# Get NLB DNS name
aws elbv2 describe-load-balancers \
  --load-balancer-arns "$NLB_ARN" \
  --query "LoadBalancers[0].DNSName" --output text

# Check NLB state (should be 'active')
aws elbv2 describe-load-balancers \
  --load-balancer-arns "$NLB_ARN" \
  --query "LoadBalancers[0].State.Code" --output text

# List target groups
aws elbv2 describe-target-groups \
  --load-balancer-arn "$NLB_ARN" \
  --query "TargetGroups[*].{Name:TargetGroupName,Port:Port,Protocol:Protocol,HealthyCount:HealthyThresholdCount}" \
  --output table

# Check target health (all targets should be 'healthy')
TG_ARN=$(aws elbv2 describe-target-groups \
  --load-balancer-arn "$NLB_ARN" \
  --query "TargetGroups[0].TargetGroupArn" --output text)

aws elbv2 describe-target-health \
  --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,Health:TargetHealth.State,Reason:TargetHealth.Reason}" \
  --output table

# Resolve the private Route 53 CNAME for the API server
# (must run from within the VPC — not from your laptop)
# dig api.k8s.${ENV}.internal.${DOMAIN} +short
```

---

## 03-infra-Manual-K8s-Cluster — SSM Sessions

```bash
# Install session-manager-plugin (macOS)
brew install --cask session-manager-plugin

# Verify plugin is installed
session-manager-plugin --version

# List SSM-managed instances (should show all cluster nodes)
aws ssm describe-instance-information \
  --query "InstanceInformationList[*].{ID:InstanceId,Ping:PingStatus,Platform:PlatformName,Version:PlatformVersion,AgentVersion:AgentVersion}" \
  --output table

# Quick connectivity check — instance should respond
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=i-XXXXXXXXXXXXXXXXX" \
  --query "InstanceInformationList[0].PingStatus" --output text
# Expected: "Online"

# Open a shell on a control plane node (no key pair needed)
aws ssm start-session \
  --target i-XXXXXXXXXXXXXXXXX \
  --region $AWS_REGION

# Run a remote command without opening a shell (example: check kubelet status)
aws ssm send-command \
  --instance-ids "i-XXXXXXXXXXXXXXXXX" \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["systemctl status kubelet --no-pager"]}' \
  --query "Command.CommandId" --output text

# Get the output of a sent command
aws ssm get-command-invocation \
  --command-id "COMMAND-ID-FROM-ABOVE" \
  --instance-id "i-XXXXXXXXXXXXXXXXX" \
  --query "StandardOutputContent" --output text

# Run a command on ALL cluster nodes at once
aws ssm send-command \
  --targets "Key=tag:Project,Values=${PROJECT}" "Key=tag:Environment,Values=${ENV}" \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["hostname && systemctl is-active kubelet"]}' \
  --region $AWS_REGION

# Check cloud-init completion on a node (useful right after launch)
aws ssm send-command \
  --instance-ids "i-XXXXXXXXXXXXXXXXX" \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["cloud-init status", "tail -20 /var/log/cloud-init-output.log"]}' \
  --query "Command.CommandId" --output text
```

---

## VPC & Networking

```bash
# List VPCs
aws ec2 describe-vpcs \
  --query "Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key=='Name'].Value|[0],Default:IsDefault}" \
  --output table

# List subnets with public/private distinction
aws ec2 describe-subnets \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch,Name:Tags[?Key=='Name'].Value|[0]}" \
  --output table

# Check route tables (private subnets should route through NAT, public through IGW)
aws ec2 describe-route-tables \
  --query "RouteTables[*].{ID:RouteTableId,VPC:VpcId,Routes:Routes[*].{Dest:DestinationCidrBlock,GW:GatewayId,NAT:NatGatewayId}}" \
  --output json

# Check NAT gateways
aws ec2 describe-nat-gateways \
  --query "NatGateways[*].{ID:NatGatewayId,State:State,SubnetId:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp}" \
  --output table

# Check internet gateway
aws ec2 describe-internet-gateways \
  --query "InternetGateways[*].{ID:InternetGatewayId,VPC:Attachments[0].VpcId,State:Attachments[0].State}" \
  --output table

# Verify subnet tags required for EKS ALB (kubernetes.io/role/elb = 1 for public)
aws ec2 describe-subnets \
  --filters "Name=mapPublicIpOnLaunch,Values=true" \
  --query "Subnets[*].{ID:SubnetId,Tags:Tags[?starts_with(Key,'kubernetes')]}" \
  --output json
```

---

## IAM Roles & Policies

```bash
# List IAM roles for this project
aws iam list-roles \
  --query "Roles[?contains(RoleName,'${PROJECT}')].{Name:RoleName,Created:CreateDate}" \
  --output table

# Check EKS cluster role
aws iam get-role \
  --role-name "${PROJECT}-eks-${ENV}-cluster-role" \
  --query "Role.{Name:RoleName,ARN:Arn,TrustPolicy:AssumeRolePolicyDocument}" \
  --output json 2>/dev/null || echo "Role not found — check actual name in console"

# List policies attached to the LB controller IRSA role
LB_ROLE=$(aws iam list-roles \
  --query "Roles[?contains(RoleName,'lb-controller')].RoleName" \
  --output text)

aws iam list-attached-role-policies \
  --role-name "$LB_ROLE" \
  --query "AttachedPolicies[*].{Name:PolicyName,ARN:PolicyArn}" \
  --output table

# Check ExternalDNS IRSA role trust policy (should trust OIDC provider)
DNS_ROLE=$(aws iam list-roles \
  --query "Roles[?contains(RoleName,'external-dns')].RoleName" \
  --output text)

aws iam get-role \
  --role-name "$DNS_ROLE" \
  --query "Role.AssumeRolePolicyDocument" \
  --output json

# List IAM instance profiles (used by 03-project EC2 nodes)
aws iam list-instance-profiles \
  --query "InstanceProfiles[?contains(InstanceProfileName,'${PROJECT}')].{Name:InstanceProfileName,Role:Roles[0].RoleName}" \
  --output table
```

---

## End-to-End Health Check

Run this sequence after all projects are applied to confirm the full stack is operational.

```bash
#!/bin/bash
# Full stack validation — run from your laptop
set -e
DOMAIN=bpmbi.us
ENV=dev
PROJECT=bpmbi
CLUSTER="${PROJECT}-eks-${ENV}"
SSM_PREFIX="/${PROJECT}/${ENV}"
REGION=us-east-1

echo ""
echo "════════════════════════════════════════════════"
echo "  bpmbi.us Full Stack Validation"
echo "════════════════════════════════════════════════"

echo ""
echo "── [1] AWS Identity ─────────────────────────────"
aws sts get-caller-identity --query "{Account:Account,User:Arn}" --output table

echo ""
echo "── [2] SSM Parameters ───────────────────────────"
aws ssm get-parameters-by-path \
  --path "$SSM_PREFIX" --recursive \
  --query "Parameters[*].Name" --output table

echo ""
echo "── [3] Route 53 Zones ───────────────────────────"
aws route53 list-hosted-zones \
  --query "HostedZones[*].{Name:Name,Private:Config.PrivateZone,Records:ResourceRecordSetCount}" \
  --output table

echo ""
echo "── [4] ACM Certificate Status ───────────────────"
for ARN in \
  $(aws ssm get-parameter --name "${SSM_PREFIX}/acm/wildcard_cert_arn" --query "Parameter.Value" --output text) \
  $(aws ssm get-parameter --name "${SSM_PREFIX}/acm/cloudfront_cert_arn" --query "Parameter.Value" --output text); do
  echo -n "  $ARN → "
  aws acm describe-certificate --certificate-arn "$ARN" --query "Certificate.Status" --output text
done

echo ""
echo "── [5] DNS — NS Records ─────────────────────────"
echo -n "  Google (8.8.8.8):    "; dig NS $DOMAIN @8.8.8.8 +short | head -1
echo -n "  Cloudflare (1.1.1.1):"; dig NS $DOMAIN @1.1.1.1 +short | head -1

echo ""
echo "── [6] EKS Cluster ──────────────────────────────"
aws eks describe-cluster --name "$CLUSTER" \
  --query "cluster.{Status:status,Version:version}" --output table 2>/dev/null \
  || echo "  EKS cluster not found or not yet applied"

echo ""
echo "── [7] EKS Nodes ────────────────────────────────"
kubectl get nodes --no-headers 2>/dev/null \
  || echo "  kubectl not configured — run: aws eks update-kubeconfig --region $REGION --name $CLUSTER"

echo ""
echo "── [8] Helm Releases ────────────────────────────"
helm list -A --no-headers 2>/dev/null \
  || echo "  kubectl not configured"

echo ""
echo "── [9] EC2 K8s Nodes ────────────────────────────"
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=${PROJECT}" "Name=tag:Environment,Values=${ENV}" \
            "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{Name:Tags[?Key=='Name'].Value|[0],State:State.Name,IP:PrivateIpAddress}" \
  --output table

echo ""
echo "── [10] SSM Reachability ────────────────────────"
aws ssm describe-instance-information \
  --query "InstanceInformationList[*].{ID:InstanceId,Ping:PingStatus}" \
  --output table

echo ""
echo "════════════════════════════════════════════════"
echo "  Validation complete"
echo "════════════════════════════════════════════════"
```

Save as `terraform/validate-stack.sh`, then:

```bash
chmod +x terraform/validate-stack.sh
./terraform/validate-stack.sh
```

---

## Quick Reference — Common Failure Modes

| Symptom | First command to run | Likely cause |
|---------|----------------------|--------------|
| `02` or `03` plan fails with "parameter not found" | `aws ssm get-parameters-by-path --path /bpmbi/dev --recursive` | `01-foundation` not applied |
| ACM cert stuck in `PENDING_VALIDATION` | `dig NS bpmbi.us @8.8.8.8 +short` | NS not delegated to Route 53 yet |
| CloudFront `InvalidViewerCertificate` | `aws acm describe-certificate --certificate-arn $CF_CERT --query "Certificate.{Status:Status,SANs:SubjectAlternativeNames}"` | FQDN not covered by cert SANs |
| Helm release fails "cluster unreachable" | `kubectl cluster-info` | `aws eks update-kubeconfig` not run after Phase 1 |
| ExternalDNS not creating records | `kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns` | IRSA role missing Route 53 permissions |
| NLB targets unhealthy | `aws elbv2 describe-target-health --target-group-arn $TG_ARN` | kubelet not running / SG rules blocking port 6443 |
| SSM session fails | `aws ssm describe-instance-information --filters Key=InstanceIds,Values=i-xxx` | SSM agent not running or IAM profile missing |
| EKS nodes not joining | `kubectl get nodes` + node CloudWatch logs | Node IAM role missing `AmazonEKSWorkerNodePolicy` |

