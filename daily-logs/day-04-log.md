# Day 4: Terraform bootstrap

**Pass:** Pass 1 (AI-assisted) | **Status:** Done | **Time:** [X mins]

## 1. Goal and acceptance criteria

Implement the bootstrap layer and move its state to the remote backend. Done when the configuration validates, a reviewed plan was applied, state lives in S3 and a plan after migration shows no changes.

## 2. Commands and steps

Files in `terraform/bootstrap/`: `versions.tf`, `providers.tf`, `variables.tf`, `state-bucket.tf`, `ecr.tf`, `oidc.tf`, `outputs.tf`, and `backend.tf` (added after the first apply).

```bash
git mv bootstrap terraform/bootstrap
terraform fmt -check
terraform init
terraform validate
terraform plan -out=tfplan      # reviewed before applying
terraform apply tfplan          # local state; included oidc.tf
# add backend.tf (bucket, key=bootstrap/terraform.tfstate, encrypt, use_lockfile)
terraform init -migrate-state   # answered yes to copy state
terraform plan                  # no changes expected
aws s3 ls s3://ovr-eks-sre-tfstate-<account-id>-us-east-1/bootstrap/
```

- Resources: state bucket with versioning, encryption, public access block, lifecycle rule and TLS-only policy; one ECR repository per service with immutable tags, scan on push and a keep-last-20 lifecycle policy; GitHub OIDC provider; deploy role `eks-sre-github-deploy` with no permissions yet.
- The OIDC trust is limited to `repo:Vivixell/eks-sre-observability` on `refs/heads/main` and `pull_request`, with the `sts.amazonaws.com` audience.
- The GitHub repo variable was moved from `oidc.tf` to `variables.tf` with no change in behaviour.
- After confirming the state in S3, the local `terraform.tfstate` and backup were deleted. Both are gitignored.
- Versions: Terraform 1.14.7, AWS provider 6.69.0. `.terraform.lock.hcl` is committed.

## 3. Verification

- `terraform validate` passed and the plan was reviewed before apply.
- `terraform init -migrate-state` reported: Successfully configured the backend "s3".
- State object present at `bootstrap/terraform.tfstate` in the bucket.
- Plan after migration: [No changes / paste result].
- Lock observed as a `.tflock` object during an apply prompt: [yes / not tested].

## 4. Issues: symptom → root cause → fix

### `terraform init -migrate-state` fails: "No valid credential sources found"

**Symptom**

```
Error: No valid credential sources found
Error: failed to refresh cached credentials, no EC2 IMDS role found,
operation error ec2imds: GetMetadata, request canceled, context deadline exceeded
```

**Root cause**

The S3 backend resolves credentials separately from the `provider "aws"` block, using the standard AWS credential chain: environment variables, then the shared profile selected by `AWS_PROFILE`, then the EC2 metadata service (IMDS). The working profile (`ovr-busayo`) was created with `aws login`, not a static access key, and was not the active profile in that shell. Terraform found nothing it could use, fell through to IMDS, which does not exist on a laptop, and timed out.

A second problem sat behind it. My default profile and an earlier profile held static access keys, and credential environment variables from earlier work were still set. Environment variables take precedence over `AWS_PROFILE`, so they would have pointed Terraform at the wrong identity and disturbed the plan.

**Fix**

First attempt, which unblocked `init`:

```bash
eval "$(aws configure export-credentials --profile ovr-busayo --format env)"
env | grep AWS_ACCESS_KEY_ID      # shows an ASIA... key, a temporary session credential
terraform init -migrate-state
```

Clean fix, so the right identity is used every time:

```bash
env | grep AWS                    # see what is set
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
export AWS_PROFILE=ovr-busayo
aws sts get-caller-identity       # must succeed and show the expected account
terraform plan
```

Exported credentials are temporary and last only for that shell session.

**Learned**

The provider and the backend authenticate independently, and environment credentials beat `AWS_PROFILE`. Clear stale `AWS_*` variables, set the profile, and run `aws sts get-caller-identity` before any Terraform command. `allowed_account_ids` in the provider adds a second check against the wrong account.

## 5. Learned

Native S3 locking removes the need for a DynamoDB table, and a locally applied bootstrap layer can safely move into its own bucket afterwards with `init -migrate-state`. State holds real resource data, so the bucket is versioned, encrypted and blocked from public access, and old versions are expired after 90 days.

Docs:
- S3 backend: https://developer.hashicorp.com/terraform/language/backend/s3
- AWS provider authentication: https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication-and-configuration
- OIDC provider resource: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider