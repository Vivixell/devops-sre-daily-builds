# Progress Tracker

**Current Day:** Day 5
**Current status:** Not started
**Last updated:** 2026-10-14
**Streaks**: 3
**Missed days**: 0
**Longest Streak**: 3

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
| 4 | Terraform bootstrap | Pass 1 | Done | 120 mins | PR #X merged; state bucket, ECR and OIDC applied; state migrated to S3 (bootstrap/terraform.tfstate), clean plan | AWS credential chain precedence (env vars vs AWS_PROFILE) |
| 5 | GitHub OIDC and AWS role | Pass 1 | Not started | — | — | — |
| 6 | Gitleaks secret scanning | Pass 1 | Not started | — | — | — |
| 7 | Terraform PR validation/plan | Pass 1 | Not started | — | — | — |
| 8 | VPC/network layer | Pass 1 | Not started | — | — | — |
| 9 | EKS provisioning | Pass 1 | Not started | — | — | — |
| 10 | EKS access entries and kubectl | Pass 1 | Not started | — | — | — |
| 11 | Online Boutique plain-manifest deployment | Pass 1 | Not started | — | — | — |
| 12 | Application verification and kubectl drills | Pass 1 | Not started | — | — | — |
| 13 | Manual Terraform apply/destroy workflows | Pass 1 | Not started | — | — | — |
| 14 | Break/fix and Week 1 review | Pass 1 | Not started | — | — | — |

Continue the table for later Days as the detailed weekly plans are created. The full roadmap and continuous Day numbering live in `master-plan.md`.

## Current checkpoint

Day 5. Bootstrap is applied and its state is in S3. The OIDC provider and deploy role exist with no permissions. Next: split plan and apply roles and verify role assumption from a workflow.

## Blockers / decisions

- Two Terraform layers instead of three: `terraform/bootstrap/` (local apply, owns state bucket, OIDC, role, ECR) and the main layer(s) under `terraform/` (pipeline).
- Design notes and write-ups live in the documentation repo only, never in the project repo.
- `master-plan.md` is the overall flow; small implementation-driven changes are recorded here and in the handoff file.

## Weekly review

### Week 1
- Recall score / concepts remembered:
- Most useful troubleshooting lesson:
- Outstanding gaps:
- AWS resources/cost decision:
- Next week's plan:
