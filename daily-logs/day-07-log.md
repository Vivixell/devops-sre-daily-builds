# Day 7: Terraform PR validation and plan

**Pass:** Pass 1 (AI-assisted) | **Status:** Done | **Time:** 120 mins

## 1. Goal and acceptance criteria

Add safe Terraform checks to pull requests. Done when formatting and validation run on PRs, a plan is visible without applying anything, and AWS access uses GitHub OIDC with no static keys.

## 2. Commands and steps

New file in the project repo: `.github/workflows/terraform-pr.yml`.

```bash
git switch main && git pull
git switch -c feat/terraform-pr-checks
# add .github/workflows/terraform-pr.yml
git add .github/workflows/terraform-pr.yml
git commit -m "ci: add terraform PR validation and plan workflow"
git push -u origin feat/terraform-pr-checks
# open PR #7, review checks and job summary, merge
```

Workflow design:

| Job | What it does | Why |
|---|---|---|
| `fmt` | `terraform fmt -check -recursive -diff` over `terraform/` | Recursive, so a new layer is format-checked even if it isn't in the matrices yet |
| `validate` | `init -backend=false`, then `validate`, per layer in a matrix | Validation needs no AWS credentials or state |
| `plan` | OIDC login with the plan role, `init`, `plan -out=tfplan`, plan written to the job summary | Read-only drift check against the remote state; never applies |

- `plan` needs `fmt` and `validate`, so a failed format or validation check stops it.
- Triggers: `pull_request` with path filters on `terraform/**` and the workflow file. Concurrency group per PR number with cancel-in-progress.
- Permissions: `contents: read` at workflow level, plus `id-token: write` on the `plan` job only.
- Role: `vars.AWS_PLAN_ROLE_ARN`, region `us-east-1`. Terraform `~> 1.15`, wrapper disabled.
- Matrix is hardcoded: `dir: [terraform/bootstrap]` in both `validate` and `plan`. Each new layer must be added to both, and needs its own backend key (for example `vpc/terraform.tfstate`).
- No `-var` or `TF_VAR_*` is passed, because every variable has a default. No tfvars file is used.
- Not a required check on `main`: the path filter means a PR without Terraform changes never triggers it, and matrix check names include the directory.

## 3. Verification

- PR #7 ran `fmt`, `validate (terraform/bootstrap)` and `plan (terraform/bootstrap)` successfully.
- Job summary: "No changes. Your infrastructure matches the configuration." This confirms the plan role can read the remote state and refresh every bootstrap resource.
- No apply step exists in the workflow, and the plan role can't apply.
- Not tested: the `fmt` failure path (a badly formatted `.tf` file), and lock behaviour during a concurrent run.

## 4. Issues: symptom → root cause → fix

No workflow failures on this day. Two wrong assumptions in the AI-drafted version were caught before pushing:

1. **`-lock=false` on plan.**
   - Symptom: the draft disabled locking because the plan role was assumed to be unable to write the lock object.
   - Root cause: the assumption was wrong. `roles.tf` gives the plan role `s3:PutObject` and `s3:DeleteObject` on `*.tflock`, because S3 native locking writes a `.tflock` object.
   - Fix: removed the flag, so locking stays on, and a PR plan can't overlap a local apply.
2. **Passing `github_sub_prefix` as an input.**
   - Symptom: the draft passed it as `TF_VAR_github_sub_prefix` from a GitHub variable.
   - Root cause: `variables.tf` already gives it a default, so no plumbing is needed.
   - Fix: removed the env block and the extra GitHub variable.

## 5. Learned

- S3 native locking is a write, so a read-only role still needs write on `*.tflock` and nothing else.
- `terraform validate` only needs `init -backend=false`, so it can run without credentials, and a plan job can then fail on its own for AWS reasons.
- A clean PR plan on bootstrap doubles as drift detection, since bootstrap was applied from the workstation.
- Path-filtered workflows shouldn't be required checks, or PRs that skip them stay "pending".
- Verify the real files before assuming how variables, backends and roles are set up.

Docs:
- S3 backend: https://developer.hashicorp.com/terraform/language/backend/s3
- GitHub OIDC for AWS: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- setup-terraform: https://github.com/hashicorp/setup-terraform
- configure-aws-credentials: https://github.com/aws-actions/configure-aws-credentials