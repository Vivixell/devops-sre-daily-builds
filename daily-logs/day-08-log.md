# Day 8: VPC/network layer

**Pass:** 1 | **Status:** Done | **Time:** [180 mins]

## 1. Goal and acceptance criteria

Define and validate the network layer for EKS. Done when the VPC, subnet, routing and endpoint design is documented, `terraform fmt`, `validate` and `plan` pass, and resource count and cost were reviewed before apply.

## 2. Commands and steps

Design decisions:

| Component | Decision | Why |
|---|---|---|
| VPC | `10.20.0.0/16`, DNS support and hostnames on | Avoids the common `10.0.0.0/16` if VPCs are ever peered; EKS nodes need DNS to register |
| Public subnets | `10.20.0.0/24`, `10.20.1.0/24` (2 AZs), no nodes | Only for the future ALB; tagged `kubernetes.io/role/elb` |
| Private subnets | `10.20.10.0/24`, `10.20.11.0/24` (2 AZs) | Nodes and pods (VPC CNI gives pods VPC IPs); tagged `kubernetes.io/role/internal-elb` |
| Egress | One regional NAT gateway, created with `vpc_id` and `availability_mode = "regional"` | Regional NAT lives in no subnet, expands across AZs with workload presence, one route entry covers all private subnets |
| Route tables | One public (to IGW), one shared private (to regional NAT) | A single NAT ID means no per-AZ private route tables |
| Endpoints | ECR `api` and `dkr` interface endpoints with private DNS; S3 gateway endpoint on the private route table | Private resolution for ECR; S3 serves image layers, so the free gateway endpoint keeps pull traffic off the NAT |
| Default SG | Emptied with `aws_default_security_group` | Nothing can use it by accident |

- **Layer.** `terraform/vpc/` with its own state key `vpc/terraform.tfstate` in the existing bucket. Provider constraint `~> 6.24` because regional NAT needs a recent 6.x.
- **Expansion room.** A third AZ is two new map entries (`10.20.2.0/24`, `10.20.12.0/24`). The private route table ID and endpoint security group ID are exported so future endpoints (S3 already, STS, logs) attach without restructuring.
- **CI.** Added `terraform/vpc` to both the `validate` and `plan` matrices in `terraform-pr.yml`.

Commands run locally (logged in with `aws login`, identity confirmed with `aws sts get-caller-identity` first):

```bash
cd terraform/vpc
terraform init
terraform fmt -check -recursive -diff
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform plan -destroy
terraform destroy
```

Cost review before apply:
- Everything is free except the regional NAT (hourly per active AZ plus data processing), its EIPs, and the interface endpoints (hourly per AZ plus data processing). Figures were estimates to confirm on the AWS VPC pricing page.
- Applied and destroyed locally in the same session to keep the bill to a short window.

Docs:
- https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html
- https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html
- https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html
- https://aws.amazon.com/vpc/pricing/

## 3. Verification

- `fmt`, `validate` and `plan` passed; plan reviewed before apply.
- Local `apply` succeeded and `destroy` completed successfully.
- [add: NAT gateway state and active AZs from `describe-nat-gateways`, if checked]
- [add: PR check results for `terraform/vpc`, if run]

## 4. Issues: symptom → root cause → fix

None.

## 5. Learned

A regional NAT gateway is attached to the VPC rather than a subnet, so public subnets are no longer needed for NAT; they exist here only for load balancers. ECR pulls split into small API calls (interface endpoints) and large layer downloads from S3 (gateway endpoint), which is why the free gateway endpoint carries most of the cost saving.