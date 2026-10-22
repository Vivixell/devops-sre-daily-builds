# Days 10–12: EKS access, first deployment, application verification

**Pass:** 1 | **Status:** Done | **Time:** [240 mins] (covers Days 10, 11 and 12)

> Days 11 and 12 were completed in this same session. Their daily logs reference this file.

## 1. Goal and acceptance criteria

- **Day 10:** reach the cluster through EKS access entries and verify `kubectl` works. Done when `kubectl get nodes`, `get namespaces` and `cluster-info` succeed and the auth path is understood.
- **Day 11:** deploy Online Boutique from plain manifests. Done when the resources exist and each workload's state is understood.
- **Day 12:** verify the application and practise core diagnostics. Done when workload health and remaining gaps are known.
- **Inserted prerequisite:** the manifests needed images, so an ECR build pipeline was built first (details below).

## 2. Commands and steps

**Day 10: access** (access entries were created in Day 9 via `terraform/eks`)

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --region us-east-1 --name sre-observability
kubectl get nodes
kubectl get namespaces
kubectl cluster-info
aws eks list-access-entries --cluster-name sre-observability
```

- Operator access comes from the access entry for IAM user `OVR-Busayo` with `AmazonEKSClusterAdminPolicy`. No `aws-auth` ConfigMap is involved.
- [add: whether a fourth entry of type `EC2_LINUX` exists for the node role]
- [add: IMDS hop-limit check, if run: pod-side `curl` to IMDS times out]

**Image pipeline** (`.github/workflows/build-images.yml`, new PR)

| Piece | Decision |
|---|---|
| Triggers | Push to `main` with `src/**`, plus `workflow_dispatch` for a one-time build of every service |
| Change detection | `dorny/paths-filter`, one filter per service |
| Build | Matrix per changed service, buildx, `linux/amd64`, GHA layer cache, `provenance: false` |
| Tag | Commit SHA, pushed to `eks-sre-observability/<service>` (ECR is immutable) |
| Auth | GitHub OIDC with the apply role (temporarily admin); no stored AWS keys |
| Manifest update | `update-manifests` job rewrites `image:` lines and opens a bump PR via `peter-evans/create-pull-request` |
| cartservice | Dockerfile is one level down, so its context is `src/cartservice/src` |

- Dispatched once from `main`: all 11 images built and pushed. `shoppingassistantservice` is not built or deployed.
- Repo settings: allowed GitHub Actions to create pull requests; ruleset keeps an admin bypass for pull requests only.
- The bump PR's `secret-scan` run needed approval; I approved it and it passed, so the bypass was not needed.
- Merged the bump PR so `kubernetes-manifests/*.yaml` point at the new account's ECR images.
- Change-detection test: [add: file committed under `src/<service>`, result: only that service rebuilt]

**Day 11: deploy**

- Commented out the frontend `Ingress` (old ALB, `external-dns`, ACM and hostname from a previous project). The frontend Service stays `ClusterIP`, so no public load balancer is created.
- Grepped the manifests for leftovers from the old project; only the `image:` lines and the Ingress matched.

```bash
kubectl apply -f kubernetes-manifests/
kubectl get pods
```

**Day 12: verify and diagnose**

```bash
kubectl get pods,svc,deploy
kubectl get pods -l app=frontend
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
kubectl get endpoints
kubectl get endpointslices
kubectl port-forward svc/frontend 8080:80
```

- Cluster nodes are in private subnets with no public IPs, so access to the app is by `kubectl port-forward` through the API server, not by NodePort.

Docs:
- https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html
- https://kubernetes.io/docs/reference/kubectl/
- https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-tag-mutability.html

## 3. Verification

- Day 10: `kubectl get nodes` [add: 2 nodes Ready], `get namespaces` and `cluster-info` all succeeded.
- Pipeline: dispatch run green; 11 images present in ECR [add: `aws ecr describe-images` spot check]; bump PR merged with only `image:` lines changed.
- Day 11: `kubectl get pods` showed all 12 pods `1/1 Running`, 0 restarts, after about 80 seconds: 11 services plus `redis-cart`.
- Day 12: [add: Services have endpoints; frontend reachable at `http://localhost:8080` through port-forward; one finding from `describe`, `logs` or `events`]
- Image pulls from ECR succeeded from the private nodes, which also exercised the ECR and S3 VPC endpoints from Day 8.

## 4. Issues: symptom → root cause → fix

| Symptom | Root cause | Fix |
|---|---|---|
| Bump PR's required `secret-scan` check did not start [confirm wording] | Workflow runs on the bot-created PR waited for approval | Approved the run; the check passed, so no bypass needed |

[add any other issues from the first pipeline run, or state "none"]

## 5. Learned

- **Day 10:** With API authentication mode, an access entry plus an access policy is the only way into Kubernetes, and it is separate from the IAM permission to call the EKS API.
- **Day 11:** An immutable ECR tag plus a per-service path filter means only changed services build, and a new tag is a new image, so manifests must be bumped to deploy it.
- **Day 12:** `describe`, `logs` and `events` answer different questions (why scheduling or pulling failed, what the app said, what the cluster did), and `port-forward` reaches a private cluster through the API server.