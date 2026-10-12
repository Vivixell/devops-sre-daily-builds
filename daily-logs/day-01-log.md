# Day 1: Audit application source and clean baseline

**Pass:** 1 | **Status:** Done | **Time:** 140 min | **Date:** 2026-10-12

## 1. Goal and acceptance criteria
Clean, application-only baseline in `eks-sre-observability`. Done when the app source is retained, old infra/pipeline artifacts are removed, and the repo is committed.

## 2. Commands and steps
- Cloned `Vivixell/microservices-demo`, removed old Terraform and pipeline files.
- Kept `src/`, `protos/`, `kubernetes-manifests/` (reference only), `.trivyignore`, `sonar-project.properties`, `.editorconfig`, `.gitattributes`.
- `git add .`, then committed directly to `main` (deviation: plan says feature branch; branch protection and PR flow start Day 2).

## 3. Verification
- `git status` showed only the intended files staged.
- Source tree contains the services and their Dockerfiles under `src/<service>/`.

## 4. Issues: symptom → root cause → fix
- CRLF warning on `logger.js` → Windows autocrlf → `.gitattributes` handles line endings; none needed.

## 5. Learned
Online Boutique is 11 gRPC services; `shoppingassistantservice` is an optional upstream extra, not part of the core set. Shell scripts need LF endings to run in Linux containers, so `.gitattributes` should enforce that regardless of the Windows checkout.