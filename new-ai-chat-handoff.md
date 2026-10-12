# New AI Chat Handoff

Use this short handoff when opening a fresh chat or switching to a new AI account. Attach these three files:
1. `master-plan.md`
2. The current `week-NN-plan.md`
3. `progress.md`

Then paste:

> We are continuing my DevOps/SRE Daily Build Challenge. Treat `master-plan.md` as the source of truth, use the current weekly plan for the next task, and use `progress.md` to determine my exact position. I am on Day [N], Pass [1 or 2], with status [status]. Help only with the current work package and keep it within its acceptance criteria. During Pass 1, explain and assist while making sure I understand. During Pass 2, do not give me a solution upfront; ask me to attempt it and provide hints or official documentation pointers only. Do not assume previous infrastructure or pipeline artifacts exist unless the files say so.

For an ongoing chat, usually send only the Day number, Pass, what you attempted, and the exact error/output. Attach the relevant file or log only when needed. Avoid pasting every historical daily log into every new chat.

Important architecture decisions:

- Use `master-plan.md` as the source of truth.
- Terraform owns AWS infrastructure and the initial Argo CD installation.
- Argo CD manages the application and observability Helm releases.
- Helm is the only deployment method used for the application pipeline.
- `kubernetes-manifests/` and `kustomize/` are retained for reference and learning, not for deploying the application.
- GitHub Actions builds, tests, scans, and publishes immutable application images to ECR. It proposes Helm values updates in Git; Argo CD reconciles merged changes.
- Never assign the same Helm release to both Terraform and Argo CD.
- Use the existing repository names and folder structure. Do not propose a restructuring unless a concrete problem justifies it.
- Before suggesting changes, check the current work package, existing repository structure, and architecture decisions. Keep changes small and consistent with the plan.
