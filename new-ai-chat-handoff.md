# New AI Chat Handoff

Use this short handoff when opening a fresh chat or switching to a new AI account. Attach these three files:
1. `master-plan.md`
2. The current `week-NN-plan.md`
3. `progress.md`

Then paste:

> We are continuing my DevOps/SRE Daily Build Challenge. Treat `master-plan.md` as the source of truth, use the current weekly plan for the next task, and use `progress.md` to determine my exact position. I am on Day [8], Pass [1], with status [starting]. Help only with the current work package and keep it within its acceptance criteria. During Pass 1, explain and assist while making sure I understand. During Pass 2, do not give me a solution upfront; ask me to attempt it and provide hints or official documentation pointers only. Do not assume previous infrastructure or pipeline artifacts exist unless the files say so.

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

Working rules for the assistant:

- Ask for the relevant files before assuming their contents. Do not generate files while something is still unverified.
- Verify against the real files (`variables.tf`, `backend.tf`, `roles.tf`, and so on) instead of guessing defaults, variables or permissions.
- Give file contents as markdown code blocks. Keep explanations concise, and include the "why" behind each decision.

Documentation rules:

- The project repo (`eks-sre-observability`) holds only code, configuration and the existing README/placeholder READMEs. Do not create design docs, notes or write-ups in it.
- Design notes, architecture decisions, troubleshooting, concept notes and daily logs go in the documentation repo (`devops-sre-daily-builds`), under `notes/` for personal study notes and `daily-logs/` for logs.
- If a work package or plan says to create a doc in the project repo (for example `docs/infrastructure-design.md`), create it in the documentation repo instead.

Deviations from `master-plan.md` made so far:

- Terraform uses two layers, not three. `terraform/bootstrap/` was applied manually from the workstation, not through the pipeline, and its state lives in the S3 bucket. It owns the S3 state bucket, the GitHub OIDC provider, the two OIDC roles and the ECR repositories. A separate `persistent` layer is not used, and can be added later if long-lived resources appear.
- Everything else (VPC, EKS, node group, access entries, Pod Identity, Argo CD install) goes in its own layer(s) under `terraform/`, each with its own state key in the same bucket. Later layers are planned in PRs and applied through manually triggered GitHub Actions workflows using the apply role (built on Day 13). Until then, a local apply is also acceptable, because the state is remote and shared and the S3 lock prevents concurrent runs.
- Week 1 Day 3 (design) and Day 4 (bootstrap) were done together. They are logged as separate rows with separate acceptance checks.
- Day 5 was done as: two OIDC roles (read-only plan role for pull requests, apply role for `main`), with role assumption verified from a workflow.
- Day 6: secret scanning is a standalone workflow `.github/workflows/secret-scan.yml` (job `gitleaks`, `gitleaks/gitleaks-action@v3`, triggers: `pull_request`, push to `main`, `workflow_call`), not a job inside the app pipeline. It is a required check on `main`. The pre-commit hook is a local hook (`repo: local`, installed gitleaks binary), not pinned in the repo. The scan runs on all PRs with no path filters.
- Day 7: Terraform PR checks are in `.github/workflows/terraform-pr.yml` with three jobs: `fmt`, `validate` and `plan`.
  - `fmt` runs `terraform fmt -check -recursive` over all of `terraform/`.
  - `validate` runs `init -backend=false` and `validate` per layer. `plan` runs `init` and `plan` per layer with the plan role over OIDC, with locking on, and writes the plan to the job summary. Nothing is ever applied from a PR.
  - Both jobs use a hardcoded matrix, currently `dir: [terraform/bootstrap]`.
  - **Rule: whenever a new `terraform/<layer>` folder is created, add it to BOTH the `validate` and `plan` matrices, and give it its own backend key** (for example `vpc/terraform.tfstate`). Remind me if I forget.
  - These checks are not required on `main`, because of the path filter and the matrix check names.

Sample of adding a layer (`terraform/vpc`):

```yaml
  validate:
    strategy:
      matrix:
        dir: [terraform/bootstrap, terraform/vpc]
  plan:
    strategy:
      matrix:
        dir: [terraform/bootstrap, terraform/vpc]
```

Infrastructure as built:

- AWS account `<account-id>`, region `us-east-1`. Terraform 1.16.6 locally (upgraded from 1.14.7 so the S3 backend supports `aws login`), `required_version = ">= 1.15.0, < 2.0.0"`, CI uses `~> 1.15`. AWS provider 6.x.
- State: S3 bucket `ovr-eks-sre-tfstate-<account-id>-us-east-1`, key `bootstrap/terraform.tfstate`, native locking (`use_lockfile`), no DynamoDB. Bootstrap was applied with local state first, then migrated with `terraform init -migrate-state`.
- Variables: all Terraform variables are hardcoded defaults (including `github_sub_prefix`). There is no tfvars file and no `TF_VAR_*` in workflows. The only GitHub Actions variables are the two role ARNs.
- Auth: `aws login` profile `ovr-busayo`. Run `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN`, then `export AWS_PROFILE=ovr-busayo`, and confirm `aws sts get-caller-identity` before any `terraform` command. Leftover credential variables override the profile. No profile is hardcoded in any `.tf` file.
- Two OIDC roles in `terraform/bootstrap/`. Apply role `eks-sre-github-deploy` (variable `AWS_APPLY_ROLE_ARN`): trusted for `main` only, temporarily `AdministratorAccess` (replace with a scoped policy before project end, do not treat as design). Plan role `eks-sre-github-plan` (variable `AWS_PLAN_ROLE_ARN`): trusted for `pull_request` only, with `ReadOnlyAccess`, state bucket read, and write on `*.tflock` only (S3 native locking needs it).
- Trust policies use the immutable subject prefix from repo Settings → Actions → OIDC (`github_sub_prefix`). It includes numeric owner and repo IDs, not just names.
- Role ARNs are GitHub Actions variables, not secrets.
- Teardown order: migrate state back to local, remove `prevent_destroy` and set `force_destroy` on the state bucket, then destroy. ECR uses `force_delete = true`.
- Gitleaks: local pre-commit hook plus the required `secret-scan` check. CI uses gitleaks 8.24.3 via the action; the repo is under a personal account, so no license key is needed.
- The `secret-scan` workflow has `pull-requests: read` only, so PR comments fail by design; read findings from the job summary. `terraform-pr` also posts no PR comments and uses the job summary.