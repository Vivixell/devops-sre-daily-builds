# DevOps and SRE Daily Build Challenge: Master Plan

## 1. Purpose

Build practical confidence in Kubernetes and Amazon EKS, then deepen observability and SRE skills through a portfolio-quality project. The project should demonstrate that I can design, build, secure, operate, troubleshoot, document, and explain a realistic cloud-native platform.

**Primary learning priorities**
1. Refresh and deepen Kubernetes/EKS knowledge, especially practical engineering and interview topics.
2. Build a solid metrics and observability stack: Prometheus, PromQL, Grafana, alerting, logs, tracing, and retention.
3. Practice reliability thinking: symptoms, signals, SLIs/SLOs, alert quality, incident investigation, and failure testing.
4. Use a sensible DevSecOps workflow so the final project is credible as a portfolio piece.
5. Improve retention through daily documentation and a later solo rebuild.

This is not CKA exam preparation. Use CKAD course material as a quick reference when useful, but focus on practical engineering, troubleshooting, official documentation, and being able to explain why a choice was made.

## 2. How the daily units work

- One numbered **Day** is a work package designed to fit a minimum of about 30 minutes. Days are not calendar dates.
- Build five days per week, normally Monday through Friday. Saturday is reserved for other tasks. Sunday is for review, recall, cost decisions, and planning the next week.
- Every work package has one main deliverable and a clear “done when” acceptance criterion. Aim for 4–6 small tasks maximum.
- If a task takes longer than 30 minutes, stop at a safe checkpoint, document what is done, and continue in the next available work package. Do not hide several large tasks inside one “day.”
- If a work package finishes early, use the “If time remains” section or begin the next numbered day. Keep the sequence and log what was actually completed.
- Documentation is part of each weekday build session. Do not defer all notes to the weekend.
- The first pass can take more than 30 minutes when useful, but 30 minutes is the minimum sustainable commitment, not a ceiling.
- Week numbers are thematic groupings, not dates. Keep the continuous Day number in the tracker.

## 3. Learning passes

Track a **Pass** field for each work package.

### Pass 1: AI-assisted build
Use AI to help plan, explain errors, compare options, and accelerate implementation. Do not blindly paste commands or code. Understand each change, verify it, and record the relevant official documentation.

### Pass 2 and later: deliberate solo rebuild
Rebuild the project slowly from a clean starting point using official documentation and your own notes. Do not ask AI to generate solutions or commands during the solo pass. Use AI only after the attempt if you need to compare your reasoning or review a specific gap, and record that as post-attempt review rather than solo execution.

The goal is not just to have a working repo. It is to retain the knowledge and be able to reproduce and explain the design.

## 4. Starting application and repositories

The only existing project material to reuse is the application source:
https://github.com/Vivixell/microservices-demo

This is Google Online Boutique, an 11-microservice demo. Clone it and remove prior infrastructure, pipeline, deployment, and observability artifacts that are not part of the application source. Verify what is retained before deleting anything. Do not assume any old cloud resources or automation still exist.

Create two repositories:

1. **Code and infrastructure:** `eks-sre-observability-lab`
   - Application source, Terraform, Kubernetes manifests, Helm, GitHub Actions, GitOps, and observability/security configuration.
2. **Learning and evidence:** `devops-sre-daily-builds-2026`
   - Daily logs, weekly plans, progress tracker, troubleshooting case studies, review notes, and portfolio evidence checklist.

Keep the learning repo separate so the project repository remains clean and employer-facing. Do not commit credentials, secret values, state files, kubeconfig files, or other sensitive artifacts.

## 5. Daily documentation standard

Write the log during the build session using these five sections. Keep it concise, but specific enough to reproduce the work.

1. **Goal and acceptance criteria** — what this work package should achieve and how “done” is verified.
2. **Commands and steps** — commands actually run and a short explanation of important changes.
3. **Verification** — tests, outputs, screenshots, or checks that demonstrate success.
4. **Issues: symptom → root cause → fix** — include “none” if there were no issues. For longer incidents, create a separate file under `troubleshooting/`.
5. **Learned** — two sentences explaining why the design or command works, plus links to official documentation.

Do not include a “Next task” section in every daily log. The tracker and current week plan are the source for what comes next. A daily log should also record the Pass, Day number, status, and time spent in its metadata or heading.

## 6. Progress tracker

Keep `progress.md` up to date. Suggested columns:

| Day | Theme | Pass | Status | Time | Deliverable / evidence | Main gap |
|---|---|---|---|---|---|---|
| 1 | Source audit and repository baseline | Pass 1 | Not started | — | — | — |

Use status values such as `Not started`, `In progress`, `Done`, and `Blocked`. Record only what actually happened. A completed day means its acceptance criteria were met, not merely that time was spent.

## 7. Git, CI/CD, infrastructure and safety standards

- Use Git and GitHub Actions from the beginning.
- Work on feature branches and merge through pull requests. Protect `main` from direct pushes using the strongest branch protection or repository rules available. As a solo project, self-merge is acceptable; do not require a second reviewer if one is not available.
- If the repository plan does not support a desired protection feature, use the strongest available equivalent and document the limitation.
- Use GitHub OIDC to assume an AWS IAM role. Do not store long-lived AWS access keys in GitHub secrets.
- Run validation and Terraform plan on pull requests where practical. Do not automatically apply infrastructure from a pull request.
- Terraform apply and destroy workflows must be manually triggered with `workflow_dispatch`. Destroy must have explicit confirmation, show/review the planned destruction, and target only the intended ephemeral layer.
- Use concurrency controls to avoid simultaneous conflicting Terraform operations.
- Separate Terraform state bootstrap, persistent resources, and ephemeral infrastructure. Never casually destroy the state backend.
- Use an S3 remote state backend with native state locking when supported by the chosen Terraform version and backend configuration. Pin Terraform/provider versions and upgrade deliberately.
- Use EKS access entries / the EKS access API for new cluster access configuration rather than starting with the legacy `aws-auth` ConfigMap.
- Prefer EKS Pod Identity for new supported workload identity use cases. Learn IRSA conceptually because it remains interview-relevant and may be needed in some environments.
- Use supported EKS/Kubernetes versions and consult current official AWS and Kubernetes documentation before implementation. Do not rely on stale copied commands.
- Keep cost awareness visible: NAT gateways, load balancers, public IPv4 addresses, EBS volumes, managed observability services, VPC endpoints, and idle clusters can all incur charges.
- Never run a destroy command without checking the workspace, account, region, state, and planned resources.

### Suggested repository layout

```text
eks-sre-observability-lab/
├── application/
├── terraform/
│   ├── bootstrap/
│   ├── persistent/
│   └── ephemeral/
├── kubernetes/
│   ├── base/
│   └── overlays/
├── helm/
├── argocd/
├── observability/
│   ├── prometheus/
│   ├── grafana/
│   ├── alertmanager/
│   ├── logging/
│   └── tracing/
├── security/
├── scripts/
├── docs/
└── .github/workflows/
```

```text
devops-sre-daily-builds-2026/
├── master-plan.md
├── week-01-plan.md
├── progress.md
├── daily-logs/
├── weekly-reviews/
├── troubleshooting/
└── portfolio-evidence/
```

## 8. DevSecOps and portfolio standard

The final project should follow a credible DevSecOps flow. Security is not the main learning focus, so introduce controls in small work packages that fit the existing roadmap.

The previous `Vivixell/microservices-demo` project used Gitleaks, SonarQube, and Trivy. Treat those tools as a reference for what to reimplement and improve, but do not assume any of their configuration exists in this clean build.

### Target application pipeline

`commit → Gitleaks → build/test → SonarQube (SAST) → Trivy (image and IaC/config scans) → push to ECR → deploy through Argo CD → optional OWASP ZAP baseline scan`

The final exact order may vary based on when an artifact exists to scan. For example, Trivy image scanning happens after an image is built, while Terraform and Kubernetes configuration scanning can happen before deployment. Do not push an image that failed required security gates.

### Security work-package placement

- **Early, around Days 6–10:** Gitleaks pre-commit setup and a GitHub Actions secret-scanning gate. Ensure the workflow itself does not expose secret values in logs.
- **When application CI and container builds are introduced:** Trivy for container images and Terraform/Kubernetes configuration. Treat findings deliberately; document justified exceptions rather than silently ignoring them.
- **After the main application CI exists:** SonarQube as a SAST/quality gate. Since hosting a SonarQube server is heavier, choose a cost-conscious setup and do not keep unnecessary infrastructure running.
- **After ingress makes the application reachable:** optional OWASP ZAP baseline scan. It is a later portfolio enhancement, not a prerequisite for Kubernetes or observability progress.
- **Optional later security extensions:** dependency scanning / OWASP Dependency-Check, Pod Security Standards, Kyverno or a similar admission policy, and runtime threat detection. Add these only after the core roadmap is stable.

Do not let security tooling consume the time intended for Kubernetes and observability. The goal is a clean, understandable pipeline, not a long list of tools.

### Portfolio evidence to collect

- README with a clear architecture diagram and explanation of major design choices.
- Pipeline diagram showing security checks, image publication, and GitOps deployment.
- Screenshot or link to a successful CI run, plus evidence of a deliberately caught failure.
- Terraform and Kubernetes validation/scanning evidence.
- Observability screenshots: useful dashboards, meaningful alerts, and a demonstrated troubleshooting scenario.
- Daily build logs and a few polished troubleshooting case studies from the documentation repo.
- A final write-up explaining trade-offs, cost controls, security boundaries, reliability signals, and what I would improve next.

## 9. Weekly routine and AWS deep dives

- **Monday–Friday:** complete one or more numbered build work packages and log each completed package.
- **Saturday:** other tasks; no scheduled build requirement.
- **Sunday:** review the week, perform recall without looking at notes first, check costs/resources, decide whether to keep the cluster up, and plan the next week.
- **Before a break longer than two days:** destroy expensive ephemeral infrastructure unless there is a clear reason to keep it running and the cost is understood. Verify persistent resources and state backend are separated first.
- AWS Skill Builder service deep dives (EKS and related services) happen outside the minimum 30-minute build block. They may inform the project, but do not replace the hands-on deliverable.
- Use the CKAD course as a quick reference where it helps. Learn Helm through the separate Helm course and learn Argo CD through targeted documentation/videos as needed.
- CloudFormation may be used for isolated AWS service exercises during deep dives; the primary project infrastructure uses Terraform.

## 10. Roadmap overview

The roadmap below is a sequence of small numbered work packages. Each Day should remain a roughly 30-minute unit. Detailed weekly plans are maintained separately and should be expanded as the next week approaches. If a package proves too large, split it into a new numbered Day rather than silently expanding the scope.

### Week 1 — Baseline, Terraform workflow, EKS foundation and first deployment
- **Day 1:** Audit the application repo; create a clean application-only baseline.
- **Day 2:** Establish project repo structure, feature-branch/PR workflow, and Terraform version/provider conventions.
- **Day 3:** Design the Terraform layers and safe state/bootstrap approach; document the dependency diagram.
- **Day 4:** Create the state backend/bootstrap layer and verify state storage/locking behavior.
- **Day 5:** Configure GitHub OIDC trust and a least-privilege deployment role; verify role assumption without long-lived keys.
- **Day 6:** Add Gitleaks pre-commit scanning and a PR secret-scanning workflow.
- **Day 7:** Add Terraform formatting, validation, and plan-on-PR workflow.
- **Day 8:** Build the VPC/network layer and validate the Terraform plan.
- **Day 9:** Provision the EKS cluster and core node capacity in the ephemeral layer.
- **Day 10:** Configure EKS access using access entries and verify least-privilege access.
- **Day 11:** Deploy Online Boutique using plain Kubernetes manifests.
- **Day 12:** Verify pods, services, endpoints, and application behavior with `kubectl`.
- **Day 13:** Add manually triggered apply/destroy workflows with guardrails for the ephemeral layer.
- **Day 14:** Troubleshoot a deliberately introduced deployment issue and document the incident.

### Week 2 — Kubernetes workload behavior and a small Helm exercise
- **Day 15:** Practice Deployments, ReplicaSets, rollout status, history, undo, and rollout restart.
- **Day 16:** Set requests/limits and investigate scheduling/resource behavior.
- **Day 17:** Configure startup, readiness, and liveness probes; observe their different effects.
- **Day 18:** Practice ConfigMaps and Secrets safely; learn updates, mounts, and environment-variable behavior.
- **Day 19:** Create a Job and CronJob; inspect completion, failures, and cleanup behavior.
- **Day 20:** Configure HPA and understand its metrics requirements and limits.
- **Day 21:** Build a tiny Helm test chart for a simple workload, not the full Online Boutique.
- **Day 22:** Install, upgrade, inspect, roll back, and uninstall the test chart; document values and templates.
- **Day 23:** Run a Kubernetes troubleshooting drill using events, logs, describe, and selectors.
- **Day 24:** Review the week's commands from memory and update the knowledge-gap list.

### Week 3 — Identity, storage, and cluster access
- **Day 25:** Create a StorageClass/PVC and understand binding and provisioning.
- **Day 26:** Test persistence across pod replacement and distinguish pod lifecycle from volume lifecycle.
- **Day 27:** Create a ServiceAccount and understand its identity inside a pod.
- **Day 28:** Implement namespace-scoped RBAC and test allowed and denied actions.
- **Day 29:** Review EKS access entries, Kubernetes RBAC, and the authentication-versus-authorization distinction.
- **Day 30:** Configure EKS Pod Identity for a small workload that accesses a narrowly scoped S3 resource.
- **Day 31:** Compare Pod Identity with IRSA conceptually and document when each may be appropriate.
- **Day 32:** Troubleshoot an intentionally broken permission or identity configuration.
- **Day 33:** Complete an access-control review and capture the evidence in the learning repo.

### Week 4 — Kubernetes networking and traffic
- **Day 34:** Trace traffic through Pod, Service, DNS, and EndpointSlice.
- **Day 35:** Investigate service discovery and DNS failures.
- **Day 36:** Learn the AWS Load Balancer Controller's role and prerequisites.
- **Day 37:** Expose the application through Ingress and verify the external path.
- **Day 38:** Review NetworkPolicy behavior and implement a small policy where the cluster networking supports it.
- **Day 39:** Investigate EKS private/public connectivity, VPC endpoints, and PrivateLink concepts.
- **Day 40:** Compare Ingress with Gateway API at a conceptual and documentation level.
- **Day 41:** Run a networking break/fix drill and document packet/traffic reasoning.
- **Day 42:** Week review: reproduce core `kubectl` and networking diagnostics from memory.

### Week 5 — Full application Helm and GitOps
- **Day 43:** Review the Online Boutique deployment and define chart boundaries/values.
- **Day 44:** Create or adapt a Helm chart for the application in small increments.
- **Day 45:** Validate chart templates and values; test install/upgrade/rollback.
- **Day 46:** Install Argo CD and understand its components and reconciliation loop.
- **Day 47:** Define a GitOps application pointing to the intended repository path.
- **Day 48:** Test sync, drift detection, health, rollback/revert, and a deliberately broken manifest.
- **Day 49:** Add Trivy scanning for application images and IaC/Kubernetes configuration in CI.
- **Day 50:** Connect the CI image-build/push flow to the GitOps deployment process; document immutable image tags.

### Week 6 — Prometheus and metrics foundations
- **Day 51:** Learn metrics, labels, cardinality, counters, gauges, histograms, and summaries.
- **Day 52:** Deploy Prometheus using a maintained Helm chart/operator approach and verify targets.
- **Day 53:** Inspect Kubernetes and application targets; troubleshoot one failed scrape.
- **Day 54:** Learn PromQL selectors, filters, aggregation, and rate/increase.
- **Day 55:** Query CPU, memory, pod availability, and restart metrics.
- **Day 56:** Add or expose a small application metric if the existing demo does not provide a useful example.
- **Day 57:** Investigate metric labels/cardinality and record safe query patterns.
- **Day 58:** Create a short PromQL practice set and answer it without notes.

### Week 7 — Grafana, alerting, and SRE thinking
- **Day 59:** Connect Grafana to Prometheus and build a first useful dashboard.
- **Day 60:** Create dashboard panels for workload health and resource usage.
- **Day 61:** Learn alert rules, evaluation, pending periods, and labels.
- **Day 62:** Configure Alertmanager routing/receivers in a safe test setup.
- **Day 63:** Create one actionable alert and test the end-to-end path.
- **Day 64:** Define a user-facing service indicator and draft an SLI/SLO with a sensible window.
- **Day 65:** Explore error budgets, burn rate, and why not every metric should page someone.
- **Day 66:** Create a dashboard/alert case study showing symptom, diagnosis, and resolution.

### Week 8 — Centralized logging
- **Day 67:** Compare logs, metrics, and traces; identify useful log fields and correlation IDs.
- **Day 68:** Select and document a cost-conscious logging path (for example Fluent Bit plus Loki, or OpenSearch if justified).
- **Day 69:** Deploy the chosen log collector and verify collection from workloads.
- **Day 70:** Query logs by namespace, workload, severity, and useful fields.
- **Day 71:** Troubleshoot missing, duplicated, malformed, or overly noisy logs.
- **Day 72:** Set retention and resource limits appropriate for a learning environment.
- **Day 73:** Correlate a known application failure with logs and metrics.
- **Day 74:** Document the logging architecture, cost trade-offs, and a troubleshooting case.

### Week 9 — Tracing and observability integration
- **Day 75:** Learn distributed tracing concepts, spans, context propagation, and sampling.
- **Day 76:** Deploy or configure an OpenTelemetry Collector in a small scope.
- **Day 77:** Send a sample trace through the chosen backend.
- **Day 78:** Trace a request across multiple services where the app supports it.
- **Day 79:** Correlate a trace with metrics and logs where instrumentation allows.
- **Day 80:** Review sampling, data volume, retention, and managed versus self-hosted options.
- **Day 81:** Add a troubleshooting walkthrough using more than one telemetry signal.
- **Day 82:** Document the observability stack and the limits of the current instrumentation.

### Week 10 — Reliability, security gates, and operational polish
- **Day 83:** Add SonarQube SAST/quality analysis to application CI using a cost-aware setup.
- **Day 84:** Review and tune Gitleaks/Trivy/SonarQube gates; document false-positive handling.
- **Day 85:** Add an optional OWASP ZAP baseline scan after the app is reachable through ingress.
- **Day 86:** Test resource pressure, a failed pod, and an unhealthy dependency in a controlled way.
- **Day 87:** Review disruption handling, replicas, probes, resource requests, and autoscaling trade-offs.
- **Day 88:** Add or refine runbooks for the most important alerts.
- **Day 89:** Review IAM scope, Kubernetes RBAC, network exposure, and secret handling.
- **Day 90:** Produce a portfolio-ready architecture and pipeline diagram.

### Week 11 — Solo rebuild and recall
- **Day 91:** Rebuild the Terraform base from notes and official documentation, without AI-generated solutions.
- **Day 92:** Rebuild the EKS access and workload deployment path.
- **Day 93:** Rebuild workload configuration, storage, and RBAC exercises.
- **Day 94:** Rebuild the Helm and Argo CD flow.
- **Day 95:** Rebuild Prometheus targets and key PromQL queries.
- **Day 96:** Rebuild a Grafana dashboard and actionable alert.
- **Day 97:** Rebuild or explain the logging and tracing paths.
- **Day 98:** Compare the solo rebuild with Pass 1 and record knowledge gaps.

### Week 12 — Final portfolio and interview readiness
- **Day 99:** Clean the main README and explain the architecture and design choices.
- **Day 100:** Verify diagrams, CI evidence, security scans, and screenshots.
- **Day 101:** Polish two or three troubleshooting case studies.
- **Day 102:** Prepare concise explanations of EKS access, Pod Identity/IRSA, RBAC, networking, metrics, alerts, and SLOs.
- **Day 103:** Run a mock incident: identify symptoms, select signals, isolate cause, mitigate, and write a short postmortem.
- **Day 104:** Complete a final cost/security review and clean up resources.
- **Day 105:** Review the full tracker, identify remaining gaps, and define the next learning cycle.

## 11. Rules for future updates

- `master-plan.md` is the source of truth for goals, conventions, architecture principles, pass rules, and the full roadmap.
- `week-NN-plan.md` is the operational plan for the current week. Flesh it out with exact tasks, commands to investigate, acceptance criteria, official documentation, and “if time remains” items before that week starts.
- When approaching Week 5 and later, update the master plan if decisions materially change, and create/update that week's detailed plan. Do not rewrite the whole master plan for minor daily changes.
- Preserve the continuous Day numbers unless the roadmap is deliberately restructured. If work expands, add a Day and update the tracker rather than cramming extra work into an existing one.
- Prefer current official AWS, Kubernetes, Terraform, GitHub Actions, Prometheus, Grafana, Helm, and Argo CD documentation. Verify current versions and commands when beginning a new phase.

## 12. Starting a new AI chat without overwhelming it

When starting a fresh chat or a new AI account, share only these files first:

1. `master-plan.md` — goals, standards, full roadmap, and working rules.
2. The current `week-NN-plan.md` — exact tasks and acceptance criteria for the active week.
3. `progress.md` — completed work, current Day, Pass, blockers, and next planned work package.

Then send a short message like:

> We are continuing my DevOps/SRE Daily Build Challenge. Treat `master-plan.md` as the source of truth, use the current weekly plan for the next task, and use `progress.md` to determine my exact position. I am on Day [N], Pass [1 or 2], with status [status]. Help only with the current work package and keep it within its acceptance criteria. During Pass 1, explain and assist while making sure I understand. During Pass 2, do not give me a solution upfront; ask me to attempt it and provide hints or official documentation pointers only. Do not assume previous infrastructure or pipeline artifacts exist unless the files say so.

For an ongoing chat, avoid reposting all daily logs. Share the current Day and Pass, what you tried, the exact error/output, and the relevant log or file only when needed. If the chat begins to lose context, re-share the three core files.

## 13. Definition of done for the overall project

The project is complete when I can explain and demonstrate:
- Reproducible infrastructure deployment and safe manual teardown.
- GitHub OIDC and least-privilege AWS access.
- EKS access entries, Kubernetes RBAC, workload identity, storage, networking, and troubleshooting.
- CI security gates, container publication, and GitOps delivery.
- Prometheus/PromQL, Grafana dashboards, actionable alerting, centralized logs, and tracing.
- A practical SLI/SLO example and at least one documented incident investigation.
- A clean public portfolio repo plus a separate, useful learning record.
- A solo rebuild or equivalent recall demonstration showing that I can reproduce the core design without relying on generated answers.
