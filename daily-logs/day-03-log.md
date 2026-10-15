# Day 3: Terraform layer and state design


## 1. Goal and acceptance criteria

Decide how the infrastructure is split before provisioning. Done when it is clear which resources can be destroyed and which must survive, and the state approach is explicit.

## 2. Commands and steps

Design decisions (done together with Day 4):

| Layer | Path | Owns | Applied by | Destroyed |
|---|---|---|---|---|
| bootstrap | `terraform/bootstrap/` | S3 state bucket, GitHub OIDC provider, deploy role, ECR repositories | Locally, manually | Only deliberately, at project end |
| main | `terraform/` (per-component folders added later) | VPC, EKS, node group, access entries, Pod Identity, Argo CD install | GitHub Actions | Routinely |

- **Two layers instead of three.** The master plan's `persistent` layer would only have held ECR, so it was folded into bootstrap. It can be split out when other long-lived resources appear.
- **OIDC provider and role in bootstrap.** A pipeline cannot create the role it needs in order to run, so these must exist before any workflow.
- **ECR in bootstrap.** Destroying a repository deletes its images, so ECR sits outside the layer that is torn down routinely.
- **State.** One S3 bucket with a key per layer and native S3 locking (`use_lockfile`), so no DynamoDB table. Bucket has versioning, SSE, public access block, a 90-day noncurrent-version expiry and a TLS-only policy.
- **Bootstrap circularity.** The bucket cannot hold its own state before it exists, so bootstrap is applied with local state, then migrated into the bucket.
- **Safety rails.** `allowed_account_ids` in the provider, `prevent_destroy` on the state bucket, default tags (`Project`, `Layer`, `ManagedBy`).
- **Teardown order.** Migrate state back to local, remove `prevent_destroy` and set `force_destroy = true` on the bucket, then `terraform destroy`. ECR uses `force_delete = true`, which removes the safety net on image deletion in exchange for easy teardown.

Trade-offs accepted:
- Any change to ECR or OIDC needs a local apply. Acceptable because they change rarely.
- Terraform plan and apply roles will be separate (Day 5), so a PR workflow cannot apply infrastructure.

Docs: https://developer.hashicorp.com/terraform/language/backend/s3

## 3. Verification

- Design is complete when the table above answers what can be destroyed and what must survive: bootstrap survives, main is disposable.
- Layer placement was challenged and revised during the session (OIDC moved from persistent to bootstrap, three layers reduced to two).

## 4. Issues: symptom → root cause → fix

None.

## 5. Learned

Resources that the pipeline depends on, such as the state bucket and the OIDC role, cannot be managed by the pipeline, so they form their own layer applied by hand. Layer boundaries should follow lifecycle (what gets destroyed and when), not resource type.