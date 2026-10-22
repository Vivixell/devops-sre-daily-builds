# Image build pipeline: design and decisions

**Workflow:** `.github/workflows/build-images.yml` | **Written:** Days 10–12

## Flow

```text
push to main (src/**) ──► changes (paths-filter) ──► build matrix (changed services) ──► ECR
                                                         └──► update-manifests ──► bump PR ──► merge ──► kubectl apply
workflow_dispatch ─────► all services (first run only)
```

## Decisions and why

| Decision | Why | Trade-off |
|---|---|---|
| Path filter, not a script | Ready-made action, one line per service | Unchanged services keep an older tag, so tags across services differ |
| Tag = commit SHA | Matches the old project; ECR is immutable so a tag always means one image | A service built from the same SHA twice fails on re-push; re-run failed jobs only |
| Considered and dropped: tree-hash tags | Content-addressed, builds only missing images, survives squash merges | Needs bash for hashing and a manifest-update script; more logic than this stage needs. Revisit later |
| `workflow_dispatch` kept | The first run must build everything, and no `src/` change exists to trigger it | One-time use; a second dispatch on the same SHA fails on immutable tags |
| OIDC with the apply role | No stored keys; role trusts `main` only | The role is temporary admin; split out a push-only role later |
| Manifests bumped by bot PR | Keeps the deploy files in step with images | PRs from `GITHUB_TOKEN` do not auto-run other workflows; the required check needed manual approval |
| No PR build, no scan | Keep scope small | A broken Dockerfile shows up after merge; Trivy arrives Day 49 |
| `provenance: false` | One plain manifest per image | No attestation manifest |
| cartservice context | Its Dockerfile is in `src/cartservice/src` | Special case in the workflow |

## Operating it

- First time or after a wipe: dispatch from `main`.
- Normal: merge a `src/<service>` change, the workflow builds only that service, merge the bump PR, apply the manifests.
- New service: update the ECR `services` variable, the filter list and the dispatch list.

## Known gaps and next steps

1. Replace the apply role with a scoped, push-only role.
2. Add a PR build (no push) and Trivy scanning before push.
3. Replace the manifest bump with a Helm values bump when Helm lands (Week 5).
4. Consider an app token for the bump PR so checks run without approval.