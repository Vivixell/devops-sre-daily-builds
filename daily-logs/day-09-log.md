# Day 9: EKS cluster provisioning

**Pass:** 1 | **Status:** Done | **Time:** 180 mins

## 1. Goal and acceptance criteria

Provision the minimum viable EKS cluster in the `terraform/eks` layer. Done when plan was reviewed before apply, the cluster exists in account `657092010570` / `us-east-1`, and the version, node capacity and cost drivers are recorded.

## 2. Commands and steps

| Component | Decision | Why |
|---|---|---|
| Layer | `terraform/eks/`, state key `eks/terraform.tfstate`, VPC read via `terraform_remote_state` | Own lifecycle, destroyed before the VPC |
| Kubernetes | 1.36 (variable, fallback 1.35) | Current supported version, add-ons verified for it |
| Auth | `authentication_mode = "API"`, creator admin off, explicit access entries | No `aws-auth` ConfigMap; no hidden admin |
| Access | Operator user + apply role: cluster admin. Plan role: view | Plan role stays read-only |
| Nodes | Managed node group, 2 x t3.large On-Demand, private subnets, AL2023, 50 GiB gp3 | Enough for the demo app; no Karpenter yet |
| IMDS | IMDSv2 required, hop limit 1 | Pods cannot reach node credentials |
| Add-ons | vpc-cni, kube-proxy before nodes; coredns, pod-identity-agent after nodes | DaemonSets can go active with no nodes; CoreDNS needs nodes |
| API endpoint | Public + private, public CIDR open | GitHub runners need public access later |

Commands (after `aws sts get-caller-identity`):

```bash
cd terraform/eks
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

- VPC layer applied first so the remote state outputs exist.
- Added `terraform/eks` to both PR matrices; renamed `vpc/output.tf` to `outputs.tf`.
- Cost drivers: control plane, 2 nodes, regional NAT, ECR interface endpoints, EBS volumes.

Docs:
- https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html
- https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html
- https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html

## 3. Verification

- Plan reviewed before apply; apply completed with no errors.
- Cluster version: 1.36, node capacity: 2 x t3.large.
- [add: `aws eks describe-cluster` status and `list-addons` output]
- [add: PR check results for `terraform/eks`]
- [add: teardown done or cluster left up]

## 4. Issues: symptom → root cause → fix

None.

## 5. Learned

Add-ons that run as DaemonSets can be created before any nodes exist, and nodes only become Ready once the CNI is running, so the ordering is cluster, then vpc-cni and kube-proxy, then nodes, then CoreDNS. With API authentication mode and creator admin turned off, the access entries are the only way in, which is why the operator ARN had to be correct before apply.