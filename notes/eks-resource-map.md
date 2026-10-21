# EKS resource map: how the cluster boots, who joins what, and what each Terraform resource does

**Layer:** `terraform/eks/` | **Cluster:** `sre-observability` | **K8s:** 1.36 | **Written:** Day 9

## 1. The two worlds

EKS is two things bolted together:

- **Control plane** (API server, etcd, scheduler, controllers). AWS runs it in an AWS-owned account. You never see the instances. You only see the cluster object and the ENIs it places in your subnets.
- **Data plane** (the EC2 nodes). These run in your VPC, in your account, and run kubelet, containerd, and the pods.

Everything below is about connecting those two worlds safely: who may talk to whom, and under which IAM identity.

## 2. Boot sequence (in the order Terraform and AWS do it)

```text
1. IAM roles exist        cluster role, node role
2. Control plane          aws_eks_cluster  -> API server + etcd, ENIs in private subnets
3. Access entries         who may call the Kubernetes API
4. Add-ons (no nodes)     vpc-cni, kube-proxy registered as DaemonSets (0 pods yet)
5. Node group             ASG -> EC2 from launch template -> kubelet starts
6. Node joins             kubelet authenticates with the node role -> node registers (NotReady)
7. CNI starts             aws-node pod runs -> node turns Ready
8. Add-ons (need nodes)   coredns pods + pod-identity-agent DaemonSet schedule
9. Cluster usable         pods get VPC IPs, DNS works, Services route
```

**Why this order:**
- Step 4 before 5: a node only becomes `Ready` once a CNI plugin is running, and the DaemonSet objects must already exist for the pods to land on it as soon as it registers.
- Step 8 after 5: CoreDNS is a Deployment and needs a node to schedule onto. The Pod Identity agent is a DaemonSet that needs nodes to run on.
- Step 3 early: with `bootstrap_cluster_creator_admin_permissions = false`, nobody can use the cluster until an access entry exists.

### What actually happens when a node joins (step 5 to 7)

1. The managed node group tells its Auto Scaling Group to launch EC2 instances from the launch template, with the EKS-optimized AL2023 AMI picked by `ami_type`.
2. EKS injects the node bootstrap configuration (cluster endpoint, CA certificate, service CIDR). AL2023 uses `nodeadm` for this.
3. The instance starts with the **node role** as its instance profile. Kubelet reads that role's credentials from IMDS (this is why IMDS must stay reachable by the host).
4. Kubelet calls the cluster API server, authenticating as the node role. EKS recognises it through an **access entry of type `EC2_LINUX`**, which EKS creates automatically for a managed node group's role when API auth mode is on. This is why `access.tf` has no entry for the node role.
5. The node registers, but stays `NotReady` (`NetworkPluginNotReady`) until the VPC CNI (`aws-node`) writes its CNI config.
6. `aws-node` uses the node role's `AmazonEKS_CNI_Policy` to attach secondary IPs to the node's ENIs. Pods then get real VPC IPs.
7. The node turns `Ready`. Image pulls use the node role's ECR read policy: ECR API and layers through the interface and S3 gateway endpoints, public images through the NAT.

The nodes reach the API server over the **private endpoint**: the control plane's ENIs sit in the private subnets, and VPC DNS resolves the cluster endpoint name to those private IPs because private access is on.

## 3. Who can join what (the identity map)

| Direction | Mechanism | Identity | Where it is defined |
|---|---|---|---|
| EKS control plane to your AWS account (create ENIs, manage SGs) | Cluster role, assumed by `eks.amazonaws.com` | `sre-observability-cluster` | `iam.tf` |
| Node (kubelet) to Kubernetes API | Access entry type `EC2_LINUX`, auto-created | Node role | Created by EKS |
| Node to AWS APIs (ECR pull, ENI attach) | Instance profile from node role, via IMDS | `sre-observability-node` | `iam.tf` |
| Human operator to Kubernetes API | Access entry plus `AmazonEKSClusterAdminPolicy` | IAM user `OVR-Busayo` | `access.tf` |
| GitHub apply workflow to Kubernetes API | Access entry plus `AmazonEKSClusterAdminPolicy` | `eks-sre-github-deploy` | `access.tf` |
| GitHub plan workflow to Kubernetes API | Access entry plus `AmazonEKSViewPolicy` | `eks-sre-github-plan` | `access.tf` |
| Pod to AWS APIs | Pod Identity (later); today pods have none, because hop limit 1 blocks the node role | None yet | Identity days (30 to 31) |

**The two-layer rule (authentication vs authorization):**
- IAM decides whether you may call the EKS **AWS API** (`eks:DescribeCluster`, needed for `update-kubeconfig`).
- The access entry plus access policy decides what you may do **inside Kubernetes**.
- You need both. The plan role has `ReadOnlyAccess` in IAM, and `AmazonEKSViewPolicy` in the cluster. Neither alone is enough.

## 4. Roles and what each one needs

| Role | Trusted principal | Permissions | Why |
|---|---|---|---|
| Cluster role | `eks.amazonaws.com` | `AmazonEKSClusterPolicy` | Control plane manages ENIs, security groups and load balancer-related resources in your account |
| Node role | `ec2.amazonaws.com` | `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` | Kubelet registration and discovery; ECR image pulls; the CNI attaching ENIs and IPs |
| `AWSServiceRoleForAmazonEKS` | EKS service | Managed by AWS | Service-linked role, created automatically. Not in Terraform. |
| Operator / apply role | IAM principal | Admin in IAM (apply role is temporary) | Run Terraform and `kubectl` |
| Plan role | GitHub OIDC, `pull_request` only | `ReadOnlyAccess` plus state bucket access | Plan only; view-only in the cluster |

**Known shortcut:** `AmazonEKS_CNI_Policy` sits on the node role, so every node-level process shares it. The cleaner design gives the CNI its own identity through Pod Identity. This is queued for the identity days.

## 5. Resource by resource

### `iam.tf`
| Resource | What it does | Why it matters |
|---|---|---|
| `aws_iam_role.cluster` plus policy attachment | Role the control plane assumes | Without it the cluster cannot create ENIs in your subnets |
| `aws_iam_role.node` plus 3 attachments | Role of the worker nodes | Without it nodes cannot join, pull images, or give pods IPs |

### `cluster.tf`
| Setting | Effect |
|---|---|
| `version = 1.36` | Control plane Kubernetes version. Upgrades go one minor version at a time. |
| `bootstrap_self_managed_addons = false` | No default CNI/kube-proxy/CoreDNS installed behind Terraform's back; `addons.tf` owns them |
| `authentication_mode = "API"` | Access entries only; no `aws-auth` ConfigMap |
| `bootstrap_cluster_creator_admin_permissions = false` | The creator gets no hidden admin; access is only what `access.tf` declares |
| `subnet_ids` = private subnets | Control plane ENIs and nodes live in private subnets |
| `endpoint_private_access = true` | Nodes reach the API server privately inside the VPC |
| `endpoint_public_access = true`, CIDR `0.0.0.0/0` | Your laptop and GitHub runners can reach it; IAM plus access entries are the gate |

EKS also creates a **cluster security group** automatically. Nodes use it (the launch template names no SG), and it allows traffic between the control plane ENIs and the nodes.

### `access.tf`
| Resource | What it does |
|---|---|
| `aws_eks_access_entry` (x3) | Registers an IAM principal as allowed to authenticate to the cluster |
| `aws_eks_access_policy_association` (x3) | Attaches a Kubernetes permission set at cluster scope. `ClusterAdminPolicy` is full control. `ViewPolicy` is read most things, no Secrets. |

Locking yourself out is possible: if the operator ARN is wrong, no entry matches and nobody can run `kubectl`. Re-apply with the right ARN to recover.

### `node-group.tf`
| Resource | What it does |
|---|---|
| `aws_launch_template.node` | Defines 50 GiB encrypted gp3 root volume and IMDS settings (see section 6). No AMI named, so EKS picks it from `ami_type`. |
| `aws_eks_node_group.main` | Creates the ASG and instances. Scaling 2/2/3 on `t3.large`, `max_unavailable = 1` for rolling updates. |

### `addons.tf`
| Add-on | Runs as | What it does | Timing |
|---|---|---|---|
| `vpc-cni` (`aws-node`) | DaemonSet, host network | Gives pods VPC IP addresses from node ENIs | Before nodes |
| `kube-proxy` | DaemonSet, host network | Programs iptables so Service IPs reach pods | Before nodes |
| `coredns` | Deployment | Cluster DNS (`service.namespace.svc.cluster.local`) | After nodes |
| `eks-pod-identity-agent` | DaemonSet, host network | Serves pod credentials on a link-local address (used later) | After nodes |

Pinned versions are in `variables.tf`. `resolve_conflicts_on_create = "OVERWRITE"` lets Terraform take over a field if EKS already set it.

### `data.tf`
Reads `vpc/terraform.tfstate` with `terraform_remote_state`. This is the only link between the layers, which is why the VPC must be applied before the EKS plan can run.

## 6. IMDSv2 required, hop limit 1

**What IMDS is:** the instance metadata service at `169.254.169.254`. On a node, it hands out the temporary credentials of the **node role**.

**IMDSv2 required (`http_tokens = "required"`):** a caller must first `PUT` for a session token and send it with every request. Simple GET-only SSRF bugs can no longer steal credentials.

**Hop limit:** the token response is sent with an IP TTL equal to the hop limit. Each routing hop decrements it.
- Host processes (kubelet, host-network pods like `aws-node`) are zero hops away, so a limit of 1 works for them.
- A normal pod sits behind a veth pair, and its traffic is routed through the host. That is one extra hop, so the token response is dropped. **With hop limit 1, regular pods cannot reach IMDS.**

**Why it matters here:** with limit 2, any compromised pod could `curl` IMDS and obtain the node role. That role can pull ECR images and drive the CNI (attach ENIs). One pod compromise becomes node-level AWS access. Hop limit 1 removes that path. This is the EKS best-practices recommendation.

**What it breaks, and the workaround:**
| Thing that normally uses IMDS | What to do |
|---|---|
| AWS SDKs falling back to the node role | Give the pod its own identity (Pod Identity, Days 30 to 31) |
| AWS Load Balancer Controller auto-discovering VPC ID and region | Set `vpcId` and `region` in its Helm values (Day 36) |
| Anything reading instance metadata for region/AZ | Pass them as env vars or values |

**Pod Identity is not affected.** The agent runs on the host network and serves credentials on its own link-local address (`169.254.170.23`), a different path from IMDS, so hop limit 1 does not block it.

**How to verify (Day 10 or 12):**
- AWS side: `aws ec2 describe-instances --filters Name=tag:Name,Values=sre-observability-node --query 'Reservations[].Instances[].MetadataOptions'` should show `HttpTokens: required` and `HttpPutResponseHopLimit: 1`.
- Pod side: run a throwaway pod and confirm it times out.
```bash
  kubectl run imds-test --rm -it --image=curlimages/curl --restart=Never -- \
    curl -m 5 -s -X PUT http://169.254.169.254/latest/api/token \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 60"
```
  Expected: no token returned (timeout). If a token comes back, the hop limit is not effective.

## 7. Cost drivers
Control plane (hourly), 2 x `t3.large` (hourly), regional NAT (hourly plus data), ECR interface endpoints (hourly per AZ), two 50 GiB gp3 volumes. Destroy `eks`, then `vpc`, after each session.

## 8. Study questions (answer without notes)
1. Why does a node stay `NotReady` until `aws-node` runs, and why is `vpc-cni` created before the node group?
2. Why is there no access entry for the node role in `access.tf`?
3. What is the difference between the IAM permission `eks:DescribeCluster` and the `AmazonEKSViewPolicy` access policy?
4. Why does hop limit 1 block pods but not `aws-node` or kubelet?
5. What breaks if `bootstrap_cluster_creator_admin_permissions = false` and the operator ARN is wrong?
6. Why does the cluster still work with the VPC endpoints only covering ECR and S3, and what goes through the NAT?

## 9. Docs
- https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html
- https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html
- https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html
- https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html
- https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html