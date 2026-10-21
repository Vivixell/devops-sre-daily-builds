# Progress Tracker

**Current Day:** Day 10
**Current status:** Not started
**Last updated:** 2026-10-21
**Streaks**: 9
**Missed days**: 0
**Longest Streak**: 9

## How to use this tracker

- Update it after every build work package.
- A Day is complete only when its acceptance criteria are met.
- If work is incomplete, keep the Day `In progress` or `Blocked`; record the exact checkpoint in the daily log.
- Record actual time spent. The work package targets a minimum 30-minute session; it is not a hard maximum.
- Pass 1 is AI-assisted. Pass 2 and later are deliberate solo rebuilds using official docs and personal notes, without AI-generated solutions upfront.
- **Deliverable / evidence:** what the day produced, plus proof it works. Use a link or short reference, such as a merged PR, commit, workflow run, or a command output you can point to. Example: `PR #3 merged; terraform validate passes`.
- **Main gap:** the one thing you didn't fully understand or want to revisit, in a few words. Example: `OIDC trust conditions`. Write `none` if there isn't one. These entries become your focus list in Pass 2.

## Work-package status

| Day | Theme | Pass | Status | Time | Deliverable / evidence | Main gap |
|---:|---|---|---|---|---|---|
| 1 | Audit application source and clean baseline | Pass 1 | Done | 140 mins | Baseline committed to main; app source + 11 services identified | shoppingassistantservice purpose; cartservice store options |
| 2 | Repository structure and PR workflow | Pass 1 | Done |120 mins | PR 1 merged; direct push to main rejected (GH013) | None |
| 3 | Terraform layer and state design | Pass 1 | Done | 120 mins | Two-layer design (bootstrap + main) with OIDC/ECR placement and teardown order, logged in daily-logs/day-03.md | none |
| 4 | Terraform bootstrap | Pass 1 | Done | 120 mins | PR #3 merged; state bucket, ECR and OIDC applied; state migrated to S3 (bootstrap/terraform.tfstate), clean plan | AWS credential chain precedence (env vars vs AWS_PROFILE) |
| 5 | OIDC role assumption and plan/apply role split | Pass 1 | Done | 120min | PR #4 merged; plan role assumed from PR run; trust limited to repo main/PR; no stored keys | immutable OIDC subject claim format |
| 6 | Gitleaks secret scanning | Pass 1 | Done | [X mins] | PR #5 merged; `secret-scan` required on main; canary PR #6 failed with github-pat finding and merge blocked; hook blocked canary locally | Local hook and CI gitleaks versions can differ |
| 7 | Terraform PR validation/plan | Pass 1 | Done | 120 mins | PR #7 merged; `terraform-pr` runs fmt, validate and plan on PRs; bootstrap plan clean (no changes); no apply in PR | Why the plan role needs write on `*.tflock` for native locking |
| 8 | VPC/network layer | Pass 1 | Done | 180 mins | `terraform/vpc` applied and destroyed locally: 2-AZ VPC, regional NAT, ECR/S3 endpoints; PR matrix updated; log in `day-08-log.md` | None |
| 9 | EKS provisioning | Pass 1 | Done | 180 mins | PR #8 merged; `terraform/eks` applied: K8s 1.36, 2 x t3.large, 4 add-ons, 3 access entries; PR matrices updated; log in `day-09-log.md` | Overall EKS setup |
| 10 | EKS access entries and kubectl | Pass 1 | Not started | — | — | — |
| 11 | Online Boutique plain-manifest deployment | Pass 1 | Not started | — | — | — |
| 12 | Application verification and kubectl drills | Pass 1 | Not started | — | — | — |
| 13 | Manual Terraform apply/destroy workflows | Pass 1 | Not started | — | — | — |
| 14 | Break/fix and Week 1 review | Pass 1 | Not started | — | — | — |

Continue the table for later Days as the detailed weekly plans are created. The full roadmap and continuous Day numbering live in `master-plan.md`.

## Current checkpoint

Day 9. `terraform/eks` is in place (cluster, node group, add-ons, access entries). Next: verify access with `kubectl` (Day 10).

## Blockers / decisions

- Two Terraform layers instead of three: `terraform/bootstrap/` (local apply, owns state bucket, OIDC, role, ECR) and the main layer(s) under `terraform/` (pipeline).
- Design notes and write-ups live in the documentation repo only, never in the project repo.
- `master-plan.md` is the overall flow; small implementation-driven changes are recorded here and in the handoff file.
- TEMPORARY: the apply role has AdministratorAccess for troubleshooting. Replace with a scoped policy before the project ends.
- Day 3/4/5 minimum Terraform version raised to `>= 1.15.0` (S3 backend needs it for `aws login`).
- Secret scanning is a standalone `secret-scan.yml` (no path filters, `workflow_call` enabled) instead of being folded into one pipeline. It stays required on `main` and the future app pipeline calls it as its first job.
- Gitleaks workflow has read-only permissions, so it can't comment on PRs. Findings are read from the job summary instead.

## Weekly review

### Week 1
- Recall score / concepts remembered:
- Most useful troubleshooting lesson:
- Outstanding gaps:
- AWS resources/cost decision:
- Next week's plan:
