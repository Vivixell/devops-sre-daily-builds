# Architecture Decisions: Infrastructure, Helm and GitOps Ownership

## Decision in one paragraph

Terraform owns AWS infrastructure and bootstraps Argo CD. Helm packages the application. Argo CD deploys the application and the SRE/observability stack from Git. GitHub Actions builds, scans and publishes images, then proposes a Helm values change by pull request. Raw manifests and Kustomize are kept only as learning references.

## 1. The big picture

| Tool | Role | Does not do |
|---|---|---|
| **Terraform** | Creates AWS infrastructure (VPC, EKS, IAM, ECR, state) and installs Argo CD | Deploy the app or manage observability releases |
| **Helm** | Packages Kubernetes resources as a chart, with environment settings in values files | Run anything by itself; it is a packaging format |
| **Argo CD** | Watches Git and keeps the cluster matching it (sync, drift detection, rollback) | Build images or create AWS infrastructure |
| **GitHub Actions** | Validates, scans, builds and pushes images, runs Terraform workflows, opens the values-bump PR | Apply application manifests to the cluster directly |

## 2. Ownership boundaries

| Component | Created by | Configured / updated by | Removed by |
|---|---|---|---|
| VPC, EKS, IAM, node capacity | Terraform | Terraform (PR plan, manual apply workflow) | Terraform (manual destroy workflow) |
| ECR repositories, state backend | Terraform (persistent / bootstrap layer) | Terraform | Protected: not removed in normal teardown |
| Argo CD itself | Terraform (Helm provider) | Terraform | Terraform |
| Foundational add-ons that need IAM wiring (for example EBS CSI, AWS Load Balancer Controller) | Terraform | Terraform | Terraform |
| Online Boutique | Argo CD (syncs the Helm chart) | Git: chart and values via PR | Argo CD (delete the Application) |
| Prometheus, Grafana, Alertmanager, logging, tracing | Argo CD | Git: values, rules and dashboards via PR | Argo CD |
| Container images | GitHub Actions | New SHA-tagged image per build | ECR lifecycle policy |

**Rule:** every component has exactly one owner. If Terraform and Argo CD both managed one Helm release, they would overwrite each other's changes. Any change to this table is documented before implementation.

**Open decision (settle before Week 6):** how Argo CD consumes upstream charts such as Prometheus and Grafana. The options are a multi-source Application, or a small wrapper chart with your values.

## 3. Deployment flow: commit to running application

```
code commit / PR
      │
      ▼
GitHub Actions: Gitleaks → build/test → SonarQube → Trivy
      │
      ▼
push image to ECR (tag = git commit SHA)
      │
      ▼
CI opens a PR changing the image tag in the Helm values
      │  (review and merge)
      ▼
Argo CD detects the change in Git
      │
      ▼
Argo CD syncs the Helm release → rolling update in the cluster
      │
      ▼
verify in Prometheus / Grafana
```

CI never runs `kubectl apply` for the application. Argo CD is the only path into the cluster.

## 4. How changes propagate

| Change | Path |
|---|---|
| **New image tag** | Code merged → CI builds, scans, pushes SHA-tagged image → CI PR bumps the Helm values → merge → Argo CD syncs → rolling update |
| **Grafana dashboard** | Edit the dashboard definition or values in Git → PR → merge → Argo CD syncs the observability app → dashboard appears. No image build, no Terraform |
| **Prometheus config or alert rule** | Edit values or rule files in Git → PR → merge → Argo CD syncs → Prometheus picks up the new rules |
| **AWS infrastructure (for example node group size)** | Terraform PR → plan reviewed → manual apply workflow |
| **Manual change in the cluster** | Argo CD flags it as drift, and can revert it to what Git says |

Argo CD checks Git about every three minutes by default. A webhook makes it instant.

## 5. Teardown

Order matters, because controllers create AWS resources that Terraform does not know about (ALBs from Ingresses, EBS volumes from PVCs).

1. **Argo CD removes its applications first.** Delete the Applications (with cascade) so the Ingresses, PVCs and their ALBs and volumes are cleaned up.
2. **Terraform destroys the ephemeral layer:** add-ons and Argo CD, then EKS, then the VPC. Argo CD cannot remove itself, because Terraform installed it.
3. **Never destroyed in normal teardown:** the state backend (bootstrap) and the persistent layer (ECR, existing DNS zone).
4. **Verify cleanup:** check for leftover load balancers, ENIs, EBS volumes (watch for `Retain` reclaim policies) and NAT gateways.

## 6. Why this approach

- **GitOps practice:** Git is the single source of truth, and every change is reviewed and reversible.
- **Clear ownership:** one owner per component, so no tool fights another.
- **Repeatable deployment:** the same flow works for the app and the observability stack, and the environment can be rebuilt from Git.
- **Better for the SRE goal:** you practise changing Prometheus, Grafana and alert rules through Git and watching them reconcile, not just provisioning them once.
- **Portfolio evidence:** it shows a realistic DevSecOps delivery path, with scans, immutable tags, PR-based promotion and reconciliation.

## 7. Official documentation

- Terraform Helm provider: https://registry.terraform.io/providers/hashicorp/helm/latest/docs
- Helm: https://helm.sh/docs/
- Argo CD: https://argo-cd.readthedocs.io/
- Argo CD with Helm: https://argo-cd.readthedocs.io/en/stable/user-guide/helm/
- Argo CD multiple sources: https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/
- GitHub Actions OIDC for AWS: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
```

I put "foundational add-ons" under Terraform in the ownership table. That matches the decision text ("Terraform may manage selected foundational cluster components"), but it's a choice, so change it if you'd rather Argo CD own them.