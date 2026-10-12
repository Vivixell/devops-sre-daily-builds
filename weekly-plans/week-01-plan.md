# Week 1 Plan — Baseline, Terraform Workflow, EKS Foundation, First Deployment

**Theme:** Build a clean baseline and establish a safe, repeatable path toward EKS.
**Pass:** Pass 1 (AI-assisted).
**Daily minimum:** one 30-minute work package. If a package runs long, stop at a safe checkpoint and carry the unfinished work forward.
**Source of truth:** `master-plan.md`; update `progress.md` after each completed Day.

> Do not try to finish all infrastructure in one sitting. A Day is a small work package, not a promise that an entire AWS component will be finished in 30 minutes.

## Day 1 — Audit the application source

**Goal and acceptance criteria**
- Establish a clean, understandable application-only baseline from `https://github.com/Vivixell/microservices-demo`.
- Done when the application source is retained, old non-application project artifacts are identified and removed only after review, and the repo can be described clearly.

**Tasks**
1. Clone the source into a temporary working directory.
2. Inspect the root tree and identify application code versus old deployment, infrastructure, pipeline, and observability artifacts.
3. Confirm the 11-service Online Boutique structure and locate the app's build/container instructions.
4. Create the new `eks-sre-observability-lab` repository and copy only the reviewed application source into it.
5. Add a minimal README note that this repository is being rebuilt from scratch; do not claim infrastructure exists.

**Deliverable:** clean application baseline in the project repository.

**Done when:** the app source and service list are clear, old infra/pipeline artifacts are not being treated as current, and the repo is committed on a feature branch.

**If time remains:** draw a simple service interaction map from the app's existing documentation.

## Day 2 — Repository workflow and project structure

**Goal and acceptance criteria**
- Establish a repeatable Git workflow and the initial repository structure.
- Done when a feature branch and pull request can be used, and the main folders exist with a short purpose note.

**Tasks**
1. Create folders for `terraform/bootstrap`, `terraform/persistent`, `terraform/ephemeral`, `kubernetes/base`, `helm`, `argocd`, `observability`, `security`, `scripts`, and `.github/workflows`.
2. Add a `.gitignore` for Terraform state/cache, kubeconfig, local environment files, credentials, and generated artifacts.
3. Create a feature branch and make a small structural commit.
4. Open a PR and merge it yourself after checking the diff.
5. Protect `main` from direct pushes using available branch rules; do not require a second reviewer for this solo project.

**Deliverable:** clean repo structure and a working PR-based workflow.

**Done when:** changes reach `main` through a reviewed PR, and sensitive/local files are ignored.

**If time remains:** document the branch strategy in the README.

## Day 3 — Terraform design and state plan

**Goal and acceptance criteria**
- Decide how the infrastructure will be split before provisioning resources.
- Done when a small architecture note explains the state backend, persistent layer, ephemeral layer, and dependency boundaries.

**Tasks**
1. Check current Terraform and AWS provider documentation before selecting versions.
2. Define three layers: `bootstrap` for state prerequisites, `persistent` for resources intended to survive cluster teardown, and `ephemeral` for VPC/EKS/temporary compute.
3. Sketch which layer owns each resource and which layers depend on others.
4. Decide on an S3 remote state backend and current supported native locking behavior.
5. Write down the account, region, naming, tagging, and destroy-safety conventions.

**Deliverable:** `docs/infrastructure-design.md` with the layer diagram and decisions.

**Done when:** it is clear which resources can be destroyed and which must survive.

**If time remains:** identify likely costs such as NAT gateways, public IPv4, load balancers, and EBS.

## Day 4 — Terraform bootstrap foundation

**Goal and acceptance criteria**
- Implement the smallest safe state/bootstrap layer.
- Done when the bootstrap configuration formats and validates, and the remote-state approach is documented. If the remote backend cannot be safely created and tested in this session, stop after a validated configuration and document the next checkpoint.

**Tasks**
1. Add the Terraform required-version and provider constraints.
2. Create a small bootstrap configuration for the remote-state prerequisites, following current AWS/Terraform documentation.
3. Run `terraform fmt -check` and `terraform validate` where configuration is ready.
4. Review the plan carefully before creating any resources.
5. Record how state is protected and why bootstrap resources are not part of routine destroy.

**Deliverable:** validated bootstrap configuration and documented state plan.

**Done when:** formatting/validation passes and the state design is explicit. Do not create a risky circular dependency between the backend and the configuration that creates it.

**If time remains:** implement/test backend locking behavior and record the result.

## Day 5 — GitHub OIDC and AWS role design

**Goal and acceptance criteria**
- Plan and, if time allows, configure GitHub Actions authentication to AWS without long-lived access keys.
- Done when the trust relationship is restricted to the intended repository/branch or environment, and the least-privilege role permissions are documented.

**Tasks**
1. Read the current GitHub OIDC and AWS IAM official docs.
2. Define the OIDC trust conditions for the intended repository and branch/environment.
3. Create or draft the IAM role and narrow permissions needed for the next infrastructure step.
4. Store no AWS access keys in GitHub secrets.
5. Test role assumption from a minimal manual workflow if the bootstrap resources and permissions are ready.

**Deliverable:** OIDC trust/role configuration or a reviewed implementation plan if AWS setup is not ready.

**Done when:** the role design is least-privilege and the authentication method is OIDC, not long-lived keys.

**If time remains:** add a short workflow job that prints the assumed identity (never secrets) to verify which role was assumed.

## Day 6 — Gitleaks early secret scanning

**Goal and acceptance criteria**
- Add an inexpensive early security gate.
- Done when a local scan can run and a PR workflow runs Gitleaks without exposing secret values.

**Tasks**
1. Read the current Gitleaks documentation and select a maintained installation/action approach.
2. Add local pre-commit guidance or a lightweight local scan command.
3. Add a GitHub Actions job for pull requests and pushes to the working branch.
4. Test against the repository and review findings rather than blindly suppressing them.
5. Document how to handle a real exposed secret: revoke/rotate it first, then clean history as appropriate.

**Deliverable:** Gitleaks local/CI scan and a brief security note.

**Done when:** the workflow completes and the repository has a documented response for a detected secret.

**If time remains:** test the detector using a clearly fake canary string in a temporary branch and remove it after confirming the detection.

## Day 7 — Terraform validation and PR plan workflow

**Goal and acceptance criteria**
- Add safe Terraform checks to pull requests.
- Done when formatting/validation runs on PRs and a plan is available for review without automatically applying infrastructure.

**Tasks**
1. Add a workflow for Terraform formatting and validation.
2. Add a plan step for the relevant Terraform layer only when credentials/state are ready.
3. Use GitHub OIDC for AWS access; do not use static access keys.
4. Ensure PR workflows never run `terraform apply`.
5. Review permissions, triggers, path filters, and workflow logs for accidental exposure.

**Deliverable:** Terraform PR validation workflow.

**Done when:** PR validation is repeatable and infrastructure changes require a separate manual apply.

**If time remains:** add concurrency controls and clear plan output/artifact handling.

## Day 8 — VPC/network layer plan

**Goal and acceptance criteria**
- Define and validate the network layer for EKS.
- Done when the VPC/subnet/routing/security design is documented and Terraform plan is reviewed.

**Tasks**
1. Read current AWS EKS networking guidance and select a subnet/addressing design.
2. Decide public/private subnet placement and egress approach with cost implications in mind.
3. Implement or scaffold the VPC layer in `terraform/ephemeral`.
4. Run format, validate, and plan.
5. Review the proposed resource count and costs before applying.

**Deliverable:** reviewed network configuration and plan.

**Done when:** the network design is coherent, validated, and cost risks are understood.

**If time remains:** compare NAT gateway versus alternative egress patterns for a learning cluster.

## Day 9 — EKS cluster provisioning

**Goal and acceptance criteria**
- Provision the minimum viable EKS cluster using the established Terraform layers.
- Done when Terraform apply completes or a precise blocker is recorded, and the cluster/version/node capacity decisions are documented.

**Tasks**
1. Check current EKS version support and Terraform provider/module guidance.
2. Configure the cluster and minimum viable compute in the ephemeral layer.
3. Run formatting, validation, and a careful plan.
4. Confirm AWS account, region, state, and expected costs before manual apply.
5. Apply only after reviewing the plan; verify cluster creation in AWS.

**Deliverable:** EKS cluster provisioned, or a documented blocker with a safe checkpoint.

**Done when:** the cluster exists in the intended account/region and the deployed version/capacity are recorded.

**If time remains:** record the key cluster resources and cost drivers.

## Day 10 — EKS access entries and kubectl verification

**Goal and acceptance criteria**
- Configure modern EKS cluster access and verify the client can reach the cluster.
- Done when access is granted through EKS access entries/API and basic `kubectl` queries work.

**Tasks**
1. Read current EKS access-entry documentation.
2. Configure an access entry for the intended operator role.
3. Configure the required Kubernetes permissions deliberately; do not grant broad admin access without a reason.
4. Update kubeconfig using the correct region and cluster.
5. Run `kubectl get nodes`, `kubectl get namespaces`, and a basic `kubectl cluster-info` check.

**Deliverable:** documented working EKS access path.

**Done when:** expected commands work and the authentication path is understood.

**If time remains:** record the difference between AWS authentication and Kubernetes authorization.

## Day 11 — Deploy Online Boutique with plain manifests

**Goal and acceptance criteria**
- Deploy the application without Helm or Argo CD yet, so Kubernetes behavior is visible.
- Done when the core workloads are applied and their status is understood, even if a first deployment needs troubleshooting.

**Tasks**
1. Inspect the application's deployment manifests and required configuration.
2. Adapt only what is necessary for the current cluster; keep changes small.
3. Apply the manifests using `kubectl`.
4. Inspect pods, Deployments, Services, events, and logs.
5. Record image-pull, resource, configuration, or scheduling problems without guessing at the cause.

**Deliverable:** first manifest-based deployment attempt and recorded status.

**Done when:** the application resources are created and the state of each core workload is understood.

**If time remains:** investigate a simple local/cluster service path without adding ingress yet.

## Day 12 — Verify the application and practice core kubectl

**Goal and acceptance criteria**
- Verify application health and practice the diagnostic commands used every day.
- Done when a short command sequence can identify workload status and the likely next troubleshooting step.

**Tasks**
1. Inspect resources by namespace and label selector.
2. Use `kubectl get`, `describe`, `logs`, `logs --previous`, `events`, and rollout status/history where applicable.
3. Verify Services and Endpoints/EndpointSlices.
4. Test the app using the available internal or external path.
5. Record one troubleshooting finding or a clear statement of what is still blocked.

**Deliverable:** verification notes and a short kubectl command reference.

**Done when:** the deployment's health and remaining gaps are known.

**If time remains:** deliberately break a non-destructive setting and restore it.

## Day 13 — Manual apply/destroy workflows

**Goal and acceptance criteria**
- Establish safe, manually triggered Terraform apply/destroy workflows for the ephemeral layer.
- Done when both workflows require `workflow_dispatch`, and destroy has explicit confirmation and a plan/review guardrail.

**Tasks**
1. Add or refine the ephemeral apply workflow with OIDC and a concurrency group.
2. Add a separate destroy workflow triggered only manually.
3. Require an explicit confirmation input such as the exact target environment name.
4. Generate and review the destroy plan before executing destruction; never target bootstrap/state resources.
5. Document account/region/state checks and any repository/environment approval features available.

**Deliverable:** guarded manual apply/destroy workflows.

**Done when:** neither workflow can be triggered accidentally by an ordinary push or PR, and destroy targets only ephemeral resources.

**If time remains:** test the workflows against a safe no-op/plan path before using them on real resources.

## Day 14 — Deliberate break/fix and weekly review

**Goal and acceptance criteria**
- Demonstrate a controlled troubleshooting cycle and review the week's knowledge.
- Done when one issue is reproduced or a simulated incident is analyzed, fixed, verified, and logged.

**Tasks**
1. Choose a reversible issue, such as a bad label selector or a noncritical manifest setting.
2. Capture the symptom before changing anything.
3. Use events, describe, logs, and manifests to form a hypothesis.
4. Apply the smallest fix and verify recovery.
5. Write the five-section daily log and update `progress.md`.

**Deliverable:** one troubleshooting record and Week 1 review.

**Done when:** the cause/fix is supported by evidence and the core week-one commands can be recalled.

**If time remains:** answer without notes: How does GitHub OIDC work? How does EKS access differ from Kubernetes RBAC? What is in the ephemeral layer, and what must never be casually destroyed?

## Official documentation starting points

Verify current pages/versions when you use them:
- Kubernetes concepts and `kubectl`: https://kubernetes.io/docs/concepts/ and https://kubernetes.io/docs/reference/kubectl/
- Amazon EKS: https://docs.aws.amazon.com/eks/latest/userguide/
- EKS access entries: https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html
- EKS Pod Identity: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- GitHub Actions OIDC for AWS: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- Terraform language and CLI: https://developer.hashicorp.com/terraform/docs
- Gitleaks: https://github.com/gitleaks/gitleaks

## End-of-week review (Sunday)

- First, recall key commands and concepts without looking at notes.
- Check `progress.md`; mark only work packages whose acceptance criteria were met as Done.
- Review unresolved issues and turn any large one into a `troubleshooting/` case study.
- Check AWS costs and running resources. Decide whether the cluster stays up or is destroyed.
- Before a break longer than two days, destroy expensive ephemeral resources after verifying the state backend and persistent resources are separate.
- Plan the next week's work packages. Saturday remains free for other tasks.
