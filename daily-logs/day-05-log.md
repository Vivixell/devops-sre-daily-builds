# Day 5: GitHub OIDC and AWS roles

**Pass:** Pass 1 (AI-assisted) | **Status:** Done | **Time:** [X mins]

## 1. Goal and acceptance criteria

Authenticate GitHub Actions to AWS with OIDC and no stored keys. Done when trust is restricted to this repo's `main` and PRs, and role assumption is verified from a workflow.

## 2. Commands and steps

- Split into two roles in `terraform/bootstrap/`:

| Role | Trusted for | Permissions |
|---|---|---|
| apply (`eks-sre-github-deploy`) | `main` only | `AdministratorAccess` (temporary) |
| plan (`eks-sre-github-plan`) | `pull_request` only | `ReadOnlyAccess` + state read and `.tflock` write/delete |

- A PR runs the workflow file from the PR branch, so a role that PRs can assume must not be able to write. That is why the roles are split.
- Role ARNs stored as GitHub Actions **variables** (`AWS_APPLY_ROLE_ARN`, `AWS_PLAN_ROLE_ARN`). They are not sensitive, and secrets would mask the account ID in logs.
- Added `.github/workflows/oidc-check.yml`: picks the role by event and runs `aws sts get-caller-identity`.
- Replaced `github_repo` with `github_sub_prefix` in the trust policies (see Issues).

```bash
terraform fmt -check && terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform output
```

## 3. Verification

- PR run of `oidc-check` assumed the plan role: [paste ARN from log / pass]
- Manual run from `main` assumed the apply role: [pass / not yet run]
- No AWS keys stored in GitHub.

## 4. Issues: symptom → root cause → fix

### `Not authorized to perform sts:AssumeRoleWithWebIdentity`

**Symptom**

```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**Root cause**

The repo uses immutable subject claims (enabled automatically for repos created or renamed after July 15, 2026). The token `sub` starts with `repo:<owner>@<owner-id>/<repo>@<repo-id>`, not `repo:<owner>/<repo>`. The trust policies matched the old format.

**Fix**

Copy the prefix from repo Settings → Actions → OIDC into a `github_sub_prefix` variable and use it in both trusts (`<prefix>:ref:refs/heads/main` and `<prefix>:pull_request`). The suffixes did not change.

If it recurs, CloudTrail's `AssumeRoleWithWebIdentity` event shows the exact `sub` that was sent.

**Learned**

The numeric IDs are GitHub's permanent account and repo IDs. A trust policy on them cannot be taken over by someone who renames the repo or re-registers its old name.

## 5. Learned

Trust policy conditions on `sub` are the real security boundary of OIDC, so split roles by trigger: a PR job should never hold a write role. The `sub` format depends on the repo's claim settings, so check it before writing the trust policy.

**Temporary exception:** the apply role has `AdministratorAccess` to ease troubleshooting. It must be replaced with a scoped policy before the project ends.

Docs:
- GitHub OIDC in AWS: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- OIDC reference (immutable claims): https://docs.github.com/en/actions/reference/security/oidc