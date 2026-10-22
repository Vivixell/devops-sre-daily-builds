# New AI Chat Handoff

Attach these three files when opening a fresh chat or switching AI accounts: `master-plan.md`, the current `week-NN-plan.md`, and `progress.md`. Then paste:

> We are continuing my DevOps/SRE Daily Build Challenge. Treat `master-plan.md` as the source of truth, use the current weekly plan for the next task, and use `progress.md` for my exact position.I am on Day [13], Pass [1], status [starting]. Help only with the current work package and keep it within its acceptance criteria. During Pass 1, explain and assist while making sure I understand. During Pass 2, give no solution upfront: ask me to attempt it and give only hints or official-doc pointers. Do not assume earlier infrastructure or pipeline artifacts exist unless the files say so.

For an ongoing chat, send only the Day, Pass, what was attempted and the exact error/output. Attach a file or log only when needed.

**Maintenance rule:** this file is a current-state snapshot. Update the relevant bullet in place; do not append per-day entries. History lives in `daily-logs/` and `progress.md`.

## Architecture decisions

- `master-plan.md` is the source of truth. Small deviations are recorded here and in `progress.md`; the master plan is not rewritten for each one.
- Terraform owns AWS infrastructure and the initial Argo CD install. Argo CD manages the application and observability Helm releases. Never assign the same Helm release to both.
- Helm is the target deployment method for the application pipeline. Until Week 5, `kubernetes-manifests/` is the active deployment path (applied manually with `kubectl apply -f`) and a bot PR keeps its image tags current. `kustomize/` stays reference only.
- GitHub Actions builds and publishes immutable images to ECR and opens a PR bumping image tags in `kubernetes-manifests/` (this becomes a Helm values bump once Helm exists). Argo CD reconciles merged changes later.
- GitHub Actions builds, tests, scans and publishes immutable images to ECR, and proposes Helm values updates in Git. Argo CD reconciles merged changes.
- Keep existing repo names and structure; restructure only for a concrete problem. Before suggesting changes, check the current work package, repo structure and these decisions, and keep changes small.

## Working rules

- Ask for the relevant files before assuming contents, and verify against the real files (`variables.tf`, `backend.tf`, `roles.tf`, and so on). Do not generate files while something is unverified.
- Give file contents as markdown code blocks. Keep explanations concise and include the "why".
- The project repo `eks-sre-observability` holds code, config and existing READMEs only. Design notes, architecture decisions, troubleshooting, concept notes and daily logs go in the docs repo `devops-sre-daily-builds` (`notes/`, `daily-logs/`), even if a plan says to create a doc in the project repo.

## Current state

**Terraform layout**
- Three layers so far: `bootstrap`, `vpc`, `eks`. No `persistent` layer (add one if long-lived resources appear). `terraform/bootstrap/` was applied manually from the workstation with state in S3. It owns the state bucket, GitHub OIDC provider, two OIDC roles and ECR repos.
- Each layer has its own folder under `terraform/` and its own state key in the same bucket, split by lifecycle and provider needs, not by resource type:
  - `vpc`: network (done).
  - `eks`: cluster, node group, add-ons and access entries (done, Day 9). Workload IAM roles, Pod Identity associations and EBS CSI are deferred to their own work packages.
  - `Argo CD` (planned): initial Argo CD install and any other Terraform-owned Helm release. It is separate because the `kubernetes`/`helm` providers need the cluster to exist first.
- Pipeline direction: every layer must be deployable locally and through one ordered workflow (vpc, eks, helm), with a matching destroy workflow in reverse order. Bootstrap stays manual and is never destroyed by a workflow.
- Layers are planned in PRs and will be applied by manually triggered workflows using the apply role (Day 13). Until then local apply is fine (remote state, S3 lock). Apply lower layers before opening the PR, because the `eks` plan reads the `vpc` remote state. Ephemeral layers are destroyed after each session (`eks` first, then `vpc`); only bootstrap persists.

**CI**
- Secret scanning: standalone `.github/workflows/secret-scan.yml` (job `gitleaks`, `gitleaks-action@v3`, gitleaks 8.24.3; triggers `pull_request`, push to `main`, `workflow_call`; no path filters). Required check on `main`. No license key (personal account). It has `pull-requests: read` only, so PR comments fail by design; read the job summary. The local pre-commit hook is `repo: local` using the installed binary.
- Terraform PR checks: `.github/workflows/terraform-pr.yml` with jobs `fmt`, `validate`, `plan`. `fmt` runs `terraform fmt -check -recursive` over `terraform/`. `validate` runs `init -backend=false` and `validate` per layer. `plan` runs `init` and `plan` per layer with the plan role over OIDC, locking on, output to the job summary (no PR comments). Nothing is applied from a PR. The matrix is hardcoded, currently `dir: [terraform/bootstrap, terraform/vpc]`. These checks are not required on `main` (path filter and matrix check names).
- **Rule: whenever a new `terraform/<layer>` folder is created, add it to BOTH the `validate` and `plan` matrices and give it its own backend key (for example `eks/terraform.tfstate`). Remind me if I forget.**
- Layers that read lower layers (`eks` reads vpc state, `helm` needs a live cluster) can only plan while those layers are up. Bring the lower layers up before opening a PR that touches them; a plan error about missing outputs or an unreachable cluster means a lower layer is down, not a code bug.

**AWS and Terraform setup**
- Account `<account-id>`, region `us-east-1`. Terraform 1.16.6 locally (needed for the S3 backend with `aws login`), `required_version = ">= 1.15.0, < 2.0.0"`, CI `~> 1.15`. AWS provider `~> 6.0` in bootstrap (locked 6.69.0), `~> 6.24` in vpc.
- State bucket `ovr-eks-sre-tfstate-<account-id>-us-east-1`, native locking (`use_lockfile`), no DynamoDB. Keys: `bootstrap/terraform.tfstate`, `vpc/terraform.tfstate`.
- Variables are hardcoded defaults: no tfvars, no `TF_VAR_*` in workflows. The only GitHub Actions variables are `AWS_APPLY_ROLE_ARN` and `AWS_PLAN_ROLE_ARN` (variables, not secrets).
- Auth: `aws login` at the start of each day, then switch to a new branch. No named profile is used or hardcoded in any `.tf`. Run `aws sts get-caller-identity` before any `terraform` command.
- OIDC roles (in bootstrap): apply role `eks-sre-github-deploy`, trusted for `main` only, temporarily `AdministratorAccess` (scope it before project end; not a design choice). Plan role `eks-sre-github-plan`, trusted for `pull_request` only, with `ReadOnlyAccess`, state bucket read, and write on `*.tflock`. Trust uses the immutable subject prefix `github_sub_prefix` (includes numeric owner and repo IDs).
- Project-end teardown: migrate state back to local, remove `prevent_destroy` and set `force_destroy` on the state bucket, then destroy. ECR uses `force_delete = true`.

**VPC layer** (`terraform/vpc/`; applied and destroyed locally on Day 8, not currently deployed)
- 2 AZs (us-east-1a/b). VPC `10.20.0.0/16`. Public /24s `10.20.0.0` and `10.20.1.0` (ALB only, no nodes). Private /24s `10.20.10.0` and `10.20.11.0` (nodes and pods) share one private route table.
- One regional NAT gateway (`vpc_id`, `availability_mode = "regional"`, lives in no subnet), one EIP per active AZ. The IGW is still required.
- Endpoints: `ecr.api` and `ecr.dkr` interface endpoints (private DNS on, SG allows 443 from the VPC CIDR) and an S3 gateway endpoint on the private route table. Public images (Docker Hub, quay.io, registry.k8s.io) still use the NAT until an ECR pull-through cache is added, which will reuse the same endpoints.
- Subnet tags for cluster `sre-observability`: `kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb`, `kubernetes.io/cluster/sre-observability = shared`.
- Outputs: `vpc_id`, `vpc_cidr`, `public_subnet_ids`, `private_subnet_ids`, `private_route_table_id`, `nat_gateway_id`, `vpc_endpoint_security_group_id`.

**EKS layer** (`terraform/eks/`; applied and destroyed locally on Days 9 to 12, not currently deployed)
- Cluster `sre-observability`, Kubernetes 1.36 (variable, fallback 1.35), raw resources, no community module. VPC values come from `terraform_remote_state` (`vpc/terraform.tfstate`).
- `authentication_mode = "API"`, `bootstrap_cluster_creator_admin_permissions = false`. Access entries: IAM user `OVR-Busayo` and apply role `eks-sre-github-deploy` get `AmazonEKSClusterAdminPolicy`; plan role `eks-sre-github-plan` gets `AmazonEKSViewPolicy`. The apply role's cluster-admin is temporary, like its AdministratorAccess.
- API endpoint public + private, public CIDR `0.0.0.0/0` on purpose (GitHub runners). No customer-managed KMS key, no control plane logging.
- Managed node group: 2 x t3.large On-Demand (min 2, desired 2, max 3), private subnets, AL2023, 50 GiB gp3 via launch template, IMDSv2 required, hop limit 1. AMI release unpinned. VPC CNI permissions sit on the node role for now.
- Hop limit 1 means the AWS Load Balancer Controller needs `vpcId` and `region` set explicitly (Day 36).
- Add-ons pinned: vpc-cni v1.23.2-eksbuild.3, kube-proxy v1.36.0-eksbuild.48, coredns v1.14.7-eksbuild.11, eks-pod-identity-agent v1.4.0-eksbuild.4. EBS CSI is deferred to the storage days.

**Application deployment (Days 11 to 12)**
- Applied manually with `kubectl apply -f kubernetes-manifests/`. 11 services plus `redis-cart` ran. The frontend Ingress is commented out; access is by `kubectl port-forward svc/frontend 8080:80`. Ingress or Gateway API comes later in the plan.