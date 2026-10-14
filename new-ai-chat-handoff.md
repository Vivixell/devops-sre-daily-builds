# New AI Chat Handoff

Use this short handoff when opening a fresh chat or switching to a new AI account. Attach these three files:
1. `master-plan.md`
2. The current `week-NN-plan.md`
3. `progress.md`

Then paste:

> We are continuing my DevOps/SRE Daily Build Challenge. Treat `master-plan.md` as the source of truth, use the current weekly plan for the next task, and use `progress.md` to determine my exact position. I am on Day [N], Pass [1 or 2], with status [status]. Help only with the current work package and keep it within its acceptance criteria. During Pass 1, explain and assist while making sure I understand. During Pass 2, do not give me a solution upfront; ask me to attempt it and provide hints or official documentation pointers only. Do not assume previous infrastructure or pipeline artifacts exist unless the files say so.

For an ongoing chat, usually send only the Day number, Pass, what you attempted, and the exact error/output. Attach the relevant file or log only when needed. Avoid pasting every historical daily log into every new chat.

Important architecture decisions:

- Use `master-plan.md` as the source of truth for the overall flow. Small changes can occur when implementation limits or better designs appear. The handoff and `progress.md` record those changes, and `master-plan.md` is not rewritten for each one.
- Terraform owns AWS infrastructure and the initial Argo CD installation.
- Argo CD manages the application and observability Helm releases.
- Helm is the only deployment method used for the application pipeline.
- `kubernetes-manifests/` and `kustomize/` are retained for reference and learning, not for deploying the application.
- GitHub Actions builds, tests, scans, and publishes immutable application images to ECR. It proposes Helm values updates in Git; Argo CD reconciles merged changes.
- Never assign the same Helm release to both Terraform and Argo CD.
- Use the existing repository names and folder structure. Do not propose a restructuring unless a concrete problem justifies it.
- Before suggesting changes, check the current work package, existing repository structure, and architecture decisions. Keep changes small and consistent with the plan.

Documentation rules:

- The project repo (`eks-sre-observability`) holds only code, configuration and the existing README/placeholder READMEs. Do not create design docs, notes or write-ups in it.
- Design notes, architecture decisions, troubleshooting, concept notes and daily logs go in the documentation repo (`devops-sre-daily-builds`), under `notes/` for personal study notes and `daily-logs/` for logs.
- If a work package or plan says to create a doc in the project repo (for example `docs/infrastructure-design.md`), create it in the documentation repo instead.

Deviations from `master-plan.md` made so far:

- Terraform uses two layers, not three. `terraform/bootstrap/` is applied locally, manually, and not through the pipeline. It owns the S3 state bucket, the GitHub OIDC provider, the deploy role and the ECR repositories. A separate `persistent` layer is not used, and can be added later if long-lived resources appear.
- Everything else (VPC, EKS, node group, access entries, Pod Identity, Argo CD install) goes in its own layer(s) under `terraform/`, deployed by GitHub Actions. Separate folders or modules per component can be added when each is built.
- Week 1 Day 3 (design) and Day 4 (bootstrap) were done together. They are logged as separate rows with separate acceptance checks.
- Day 5 now covers verifying role assumption from a workflow and splitting the roles: a read-only plan role trusted for pull requests, and an apply role trusted only for `main`. The deploy role exists but has no permissions yet.

Infrastructure as built:

- AWS account `<account-id>`, region `us-east-1`. Terraform 1.14.x, AWS provider 6.x.
- State: S3 bucket `ovr-eks-sre-tfstate-<account-id>-us-east-1`, key `bootstrap/terraform.tfstate`, native locking (`use_lockfile`), no DynamoDB. Bootstrap was applied with local state first, then migrated with `terraform init -migrate-state`.
- Auth: `aws login` profile `ovr-busayo`. Run `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN`, then `export AWS_PROFILE=ovr-busayo`, and confirm `aws sts get-caller-identity` before any `terraform` command. Leftover credential variables override the profile.
- OIDC role `eks-sre-github-deploy` trusts `Vivixell/eks-sre-observability` on `main` and `pull_request`.
- Teardown order: migrate state back to local, remove `prevent_destroy` and set `force_destroy` on the state bucket, then destroy. ECR uses `force_delete = true`.
