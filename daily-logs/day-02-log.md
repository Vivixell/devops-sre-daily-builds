# Day 2: Repository structure and PR workflow



## 1. Goal and acceptance criteria

Establish a repeatable Git workflow and the initial repository structure.

Done when:
- A change reaches `main` through a pull request.
- The main folders exist with a short purpose note.
- Sensitive and local files are ignored.
- `main` is protected from direct pushes.

## 2. Commands and steps

### Branch protection (GitHub ruleset `protect-main`)
- Settings → Rules → Rulesets → New branch ruleset
- Enforcement: Active. Bypass list: empty (applies to the owner too).
- Target: default branch.
- Enabled: restrict deletions, block force pushes, require a pull request (0 approvals, since this is a solo project).
- Left off: required status checks (no workflows exist yet; add after Days 6 and 7) and linear history (kept merge commits so individual commits stay revertable).

### Git commands used

| Command | Purpose |
|---|---|
| `git remote -v` | Show which remote URLs push and fetch go to |
| `git branch -vv` | Show local branches, latest commit, upstream tracking and ahead/behind |
| `git status` | Show working tree state |
| `git switch -c <branch>` | Create a branch and switch to it; fails if the branch already exists |
| `git switch <branch>` | Move to an existing branch |
| `git add <files>` | Stage only the intended files |
| `git commit -m "..."` | Commit using conventional-commit style (`chore:`, `fix:`) |
| `git push -u origin <branch>` | Push and set the upstream so later pushes need only `git push` |
| `git reset --hard origin/main` | Discard local commits and match the remote (used to drop the test commit) |
| `git check-ignore -v <paths>` | Show which `.gitignore` rule matches a path |
| `git pull` | Sync local `main` after the merge |
| `git branch -d <branch>` | Delete a local branch safely; refuses if it has unmerged commits |
| `git branch -D <branch>` | Force-delete a local branch even if unmerged (use with care) |
| `git push origin --delete <branch>` | Delete the branch on GitHub |
| `git fetch --prune` | Fetch from the remote and remove local references to remote branches that no longer exist |
| `git branch -a` | List local and remote-tracking branches |
| `git checkout -b <branch>` | Older equivalent of `git switch -c`; also fails if the branch exists |
| `git switch -C <branch>` | Create or reset a branch to the current commit; overwrites an existing branch |
| `git revert -m 1 <merge-sha>` | Undo a whole merged PR by reverting its merge commit |


### Branch cleanup after a merged PR

```bash
git switch main
git pull
git branch -d feat/repo-structure
git push origin --delete feat/repo-structure
git fetch --prune
```

- `git pull` on `main` brings the PR's merge commit into the local branch before the feature branch is deleted.
- Deleting a merged branch removes only the branch pointer. The commits stay in `main` through the merge.
- Start each new work package from updated `main` with a fresh `git switch -c <branch>`.

### Concepts

- `git remote -v`: `origin` is the default name for the GitHub remote; fetch and push URLs are listed separately.
- `git branch -vv`: `[origin/main]` is the upstream branch, and `ahead N` or `behind N` appears when the branch has diverged.
- Without `-c`, `git switch <name>` only moves to an existing branch and errors if it doesn't exist.
- A root `.gitignore` applies recursively to all subdirectories, but it doesn't affect files Git already tracks.

### Structure
- Created placeholder `README.md` files in `bootstrap/`, `terraform/`, `helm/`, `gitops/` and `kustomize/`, because Git does not track empty directories.
- `kubernetes-manifests/` already had a README. `docs/` and `.github/workflows/` were deferred to Days 3 and 6.
- Extended the root `.gitignore` for Terraform state/cache/plans/tfvars, kubeconfig files, `.env` files and keys. `.terraform.lock.hcl` is intentionally not ignored.

### PR
- Branch `feat/repo-structure` → PR [#X](link) → merged with a merge commit.

## 3. Verification

- Direct push to `main` rejected:
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: - Changes must be made through a pull request.
- Local test commit removed with `git reset --hard origin/main`; `git status` clean.
- `git check-ignore -v terraform.tfstate .env kubeconfig` matched the expected rules.
- Merged PR: [link]

## 4. Issues: symptom → root cause → fix

None.

## 5. Learned

A ruleset is enforced by GitHub on the server, so `main` stays protected whatever a local client or workflow does, and with an empty bypass list it applies to the owner too. A root `.gitignore` applies recursively and keeps state files, kubeconfigs and keys out of history, where they would be hard to remove once committed.

Notes:
- Merge methods: merge commit keeps all commits and allows `git revert -m 1 <merge-sha>` for a whole PR. Squash collapses a PR into one commit, so a revert undoes it all. Rebase replays commits linearly.
- `switch` and `restore` replaced the overloaded `checkout`, and `switch` only touches branches.
- CI-created PRs are not blocked by the ruleset. They need the "Allow GitHub Actions to create and approve pull requests" setting, and PRs opened with `GITHUB_TOKEN` don't trigger other workflows, which matters once status checks are required (Day 50).

