# Day 81 — Introduction to Amazon EKS with Terraform

## EKS Architecture

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes offering. AWS manages the **control plane** (API server, etcd, scheduler, controller manager) while you manage the **data plane** (worker nodes where pods run).

```
YOU (kubectl)
     │
     ▼ API calls
┌─────────────────────────────────────────┐
│         EKS Control Plane (AWS-managed) │
│   API Server │ etcd │ Scheduler │ CM    │
└─────────────────────────────────────────┘
     │ schedules pods
     ▼
┌──────────────── Your VPC (10.0.0.0/16) ─────────────────┐
│                                                           │
│  AZ-a                AZ-b                AZ-c            │
│  Public  10.0.1/24   Public  10.0.2/24   Public 10.0.3/24│
│  Private 10.0.4/24   Private 10.0.5/24   Private10.0.6/24│
│  Intra   10.0.7/24   Intra   10.0.8/24   Intra  10.0.9/24│
│                                                           │
│  [EC2 t3.medium]     [EC2 t3.medium]     [EC2 t3.medium] │
│  Pods + EBS          Pods + EBS          Pods + EBS       │
└───────────────────────────────────────────────────────────┘
```

**Subnet purposes:**
- **Public subnets** — Internet-facing Load Balancers (tagged `kubernetes.io/role/elb`)
- **Private subnets** — Worker nodes (tagged `kubernetes.io/role/internal-elb`)
- **Intra subnets** — Control plane ENIs only, fully isolated

---

## Terraform Configuration Explained

### `variables.tf`
Declares the input variables (knobs) the configuration accepts — region, cluster name, K8s version, node instance type, and node counts. Does not provision anything itself.

### `terraform.tfvars`
Provides the actual values for those variables. Notably overrides `node_instance_type` to `m7i-flex.large` (8GB RAM) to accommodate Ollama's memory requirements.

### `provider.tf`
Three responsibilities:
1. Declares AWS (~6.0) and Helm (~2.17) providers
2. Defines `locals` — all computed values (VPC CIDR, subnet CIDRs, AZ list)
3. Fetches live AZ data from AWS using a `data` source, making the config region-agnostic

### `vpc.tf`
Uses the `terraform-aws-modules/vpc/aws` module to create a full VPC with 9 subnets (3 public, 3 private, 3 intra) across 3 AZs, 3 NAT Gateways, and an Internet Gateway. Subnet tags are critical for the Load Balancer Controller to know where to place AWS load balancers.

### `eks.tf`
The core file. Creates:
- EKS cluster (Kubernetes 1.35) with public + private API endpoint
- 6 managed add-ons (coredns, kube-proxy, vpc-cni, eks-pod-identity-agent, aws-ebs-csi-driver, metrics-server)
- Managed node group (3x t3.medium, min 3, max 5) in private subnets
- IRSA (IAM Roles for Service Accounts) for the EBS CSI driver — gives only the CSI controller permission to manage EBS volumes

### `argocd.tf`
Installs ArgoCD via Helm after the EKS cluster is ready (`depends_on = [module.eks]`). Exposes the server as a LoadBalancer service for easy browser access.

### `outputs.tf`
Prints useful post-deployment information: cluster endpoint, VPC/subnet IDs, and helper commands for `kubectl` and ArgoCD password retrieval.

---

## EKS Add-ons

| Add-on | Pods | Purpose |
|--------|------|---------|
| `vpc-cni` | 1 per node (DaemonSet) | Assigns real VPC IPs to pods |
| `coredns` | 2 (Deployment) | DNS resolution inside the cluster |
| `kube-proxy` | 1 per node (DaemonSet) | Service network routing |
| `eks-pod-identity-agent` | 1 per node (DaemonSet) | Enables pod-level IAM roles |
| `aws-ebs-csi-driver` | 2 controllers + 1 per node | Creates/attaches EBS volumes for PVCs |
| `metrics-server` | 2 (Deployment) | CPU/RAM metrics for HPA and kubectl top |

---

## Cluster Verification

### kubectl get nodes -o wide
```
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-0-4-201.us-west-2.compute.internal   Ready    <none>   9m    v1.35.5-eks-3385e9b
ip-10-0-5-174.us-west-2.compute.internal   Ready    <none>   9m    v1.35.5-eks-3385e9b
ip-10-0-6-224.us-west-2.compute.internal   Ready    <none>   9m    v1.35.5-eks-3385e9b
```
3 nodes across us-west-2a, us-west-2b, us-west-2c — all Ready.

### kubectl get pods -n kube-system
All 6 add-ons running: aws-node (vpc-cni), coredns, ebs-csi-controller, ebs-csi-node, eks-pod-identity-agent, kube-proxy, metrics-server.

### kubectl top nodes
```
NAME                                       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
ip-10-0-4-201.us-west-2.compute.internal   52m          2%       674Mi           20%
ip-10-0-5-174.us-west-2.compute.internal   49m          2%       562Mi           17%
ip-10-0-6-224.us-west-2.compute.internal   39m          2%       506Mi           15%
```

### ArgoCD
- URL: `http://a175211758257420e81fcd41f7b75493-1405390457.us-west-2.elb.amazonaws.com`
- Version: v3.4.2
- Status: Accessible, logged in successfully
- Applications: None yet (GitOps setup comes on Day 84-86)

---

## EKS Cost Breakdown

| Component | Cost |
|-----------|------|
| EKS Control Plane | $0.10/hr (~$73/month) |
| t3.medium nodes (3x) | ~$0.042/hr each (~$91/month total) |
| NAT Gateways (3x) | ~$0.045/hr each + $0.045/GB data (~$99/month) |
| EBS volumes (15Gi total) | ~$1.50/month |
| LoadBalancer (ArgoCD) | ~$0.025/hr (~$18/month) |
| **Total** | **~$282/month (~$9.40/day)** |

**Why is NAT Gateway surprisingly expensive?**
Each private subnet node must route all outbound traffic (pulling container images, AWS API calls, CloudWatch logs) through a NAT Gateway. With 3 NAT Gateways (one per AZ for HA) each charging $0.045/hr plus $0.045/GB of processed data, the costs compound quickly — especially when nodes are constantly pulling multi-hundred-MB container images.

> ⚠️ Always run `terraform destroy` when not actively using the cluster to avoid unexpected charges (~$9/day).

---

## Key Concepts Learned

**Managed vs Self-Managed:** EKS manages the control plane — you never SSH into an API server or worry about etcd backups. You only manage worker nodes.

**IRSA (IAM Roles for Service Accounts):** Instead of giving broad IAM permissions to entire EC2 nodes, IRSA lets individual pods assume specific IAM roles. Only the EBS CSI controller can create EBS volumes; the BankApp pod gets zero AWS permissions.

**Three subnet types:** Public (LBs), Private (nodes, secure), Intra (control plane ENIs, most isolated).

**Terraform idempotency:** Running `terraform apply` multiple times is safe — it only creates what doesn't exist yet, making it easy to resume after failures.

**DaemonSets vs Deployments:** Add-ons like vpc-cni, kube-proxy, and eks-pod-identity-agent run as DaemonSets (one pod per node automatically). CoreDNS and metrics-server run as Deployments (fixed replica count).
