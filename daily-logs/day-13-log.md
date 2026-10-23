# Days 13–14: Terraform apply/destroy workflows and incident review

**Pass:** 1 | **Status:** Done | **Time:** Day 13 120 mins, Day 14 inc. in day 13

> Day 14 is recorded in this file (see the last section). Its daily log references this one.

## 1. Goal and acceptance criteria

- **Day 13:** add guarded apply and destroy workflows for the ephemeral layers. Done when destroy cannot be triggered by a push or PR, destroy has an explicit confirmation and a plan to review, bootstrap is never targeted, and the account, region and state checks are documented.
- **Deliberate deviation:** the plan said apply must be manual. I chose to also run apply on merge to `main` for `terraform/vpc/**` and `terraform/eks/**`, with a guard so a push never creates a layer from nothing.
- **Day 14:** analyse an incident, fix and verify it, and review the week. Done when the cause and fix are supported by evidence.

## 2. Commands and steps

| File | Purpose |
|---|---|
| `terraform-layer.yml` | Reusable job that runs one layer: identity check, state guard, `init`, `plan -out`, plan to summary, apply the saved plan |
| `terraform-apply.yml` | Push to `main` (vpc and eks paths) or `workflow_dispatch`; chain `vpc` then `eks` |
| `terraform-destroy.yml` | Dispatch only; chain `eks` then `vpc`; input `confirm` (`no` or `yes`, default `no`) |

Decisions and why:
- **Reusable workflow:** one copy of the nine steps instead of four, and a new layer costs one job per file.
- **Apply on push, with a guard:** the guard reads the layer's state from S3, and a push skips the layer if it has no resources. Dispatch always applies. A merge after a destroy therefore never rebuilds the stack.
- **Destroy preview:** `confirm = no` runs `plan -destroy` and shows it in the job summary. `yes` applies that same saved plan.
- **Safety:** bootstrap is refused inside the reusable workflow, apply and destroy share one concurrency group with no cancel-in-progress, jobs have a timeout, and plans use `-lock-timeout`.
- **No GitHub Environments:** a job using an environment gets a different OIDC subject, and the apply role trusts `main` only.
- **Role:** both workflows use the apply role (temporarily AdministratorAccess), so anything merged under the vpc or eks paths runs as admin until the roles are split.

Test runs:
1. `terraform-apply` by dispatch with the stack up: worked, no changes to apply.
2. `terraform-destroy` with `confirm = no`: worked, showed the destroy plan only.
3. The same again, cancelled halfway.
4. `terraform-destroy` with `confirm = yes`: destroyed `eks`, then `vpc`.

Docs:
- https://docs.github.com/en/actions/using-workflows/reusing-workflows
- https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onworkflow_dispatchinputs
- https://developer.hashicorp.com/terraform/language/backend/s3

## 3. Verification

- Apply dispatch: plan reported no changes, as expected with the infrastructure still up.
- Destroy preview: plan shown in the job summary and nothing was destroyed.
- Destroy with `yes`: `eks` finished before `vpc` started, and the run completed. [add: no clusters, NAT gateway gone, no network interfaces in the VPC]
- [add: no leftover `*.tflock` objects in the state bucket]
- Not yet tested: a push touching `terraform/vpc` or `terraform/eks` with the stack up (expect no changes) and with it down (expect both layers skipped). [add result when run]

## 4. Issues: symptom → root cause → fix

| Symptom | Root cause | Fix |
|---|---|---|
| Cancelled a destroy preview halfway and worried about state corruption | A plan-only run does not write state; at most it leaves a stale lock object | Later runs worked, so no stale lock; verified no `*.tflock` objects remain [confirm] |
| `terraform destroy` run locally on Oct 22 failed after the EKS cluster was destroyed: `lookup iam.amazonaws.com: no such host` | Local DNS or network drop; the request never left the machine (empty RequestID) | Restored DNS or connectivity, re-ran destroy; Terraform had recorded the cluster as destroyed and only tried what was left [confirm outcome] |

## 5. Learned

A reusable workflow lets two ordered chains of layers share one implementation, and a state-based guard stops a push from creating infrastructure that was deliberately destroyed. Cancelling a plan is harmless, but cancelling an apply or destroy can kill Terraform before it finishes and leave a stale lock, so the workflows never cancel a running job.

## Day 14: incident review and Week 1 summary

**Pass:** 1 | **Status:** Done (substituted) | **Time:** [X mins]

**Goal and acceptance criteria:** show one controlled troubleshooting cycle with evidence. The plan's version was a deliberately introduced Kubernetes fault. I did not do that; I analysed real incidents from this week instead.

**Commands and steps**
- Incident analysed: the local destroy failure above (DNS lookup error), including reading the error to find where it failed (client side, before any AWS response), and the state recovery path (re-run destroy).
- Related review: the cancelled destroy preview and its effect on state and locks (Issues table).
- [add: the deliberate-break exercise from Day 12, if it was done, with symptom, cause and fix]
- Commands to recall without notes: `kubectl get/describe/logs/logs --previous`, `kubectl get events --sort-by=.lastTimestamp`, `kubectl rollout status/history`, `kubectl get endpoints`, `aws eks update-kubeconfig`, `terraform plan -destroy -out`.

**Verification:** cause supported by the error text (no response ID, `no such host`), fix confirmed by the re-run. [add: the re-run's result]

**Issues:** none beyond those listed for Day 13.

**Learned:** An error with no request ID and a failed hostname lookup points at the client's network, not at AWS or Terraform. Terraform records each resource as it finishes, so a failed destroy can be re-run safely.

**Known gap:** no deliberate Kubernetes break/fix was recorded. I'll cover it in the Week 2 troubleshooting drill (Day 23).