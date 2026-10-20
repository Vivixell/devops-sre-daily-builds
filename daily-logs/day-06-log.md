# Day 6: Gitleaks secret scanning


## 1. Goal and acceptance criteria
Add an early secret-scanning gate: a local pre-commit hook and a PR workflow that block secrets without exposing values.
Done when:
- the full-history scan is reviewed
- the hook blocks a staged secret
- the CI check fails on a canary and passes on clean code
- the check is required on `main`

## 2. Commands and steps

**Install gitleaks (PowerShell, Windows):** download the latest Windows x64 release zip via the GitHub API, extract it to `$HOME\gitleaks`, add that folder to the User PATH, and verify.
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\gitleaks"
$LatestRelease = Invoke-RestMethod -Uri "https://api.github.com/repos/gitleaks/gitleaks/releases/latest"
$DownloadUrl = ($LatestRelease.assets | Where-Object { $_.name -like "*windows_x64.zip" }).browser_download_url
Invoke-WebRequest -Uri $DownloadUrl -OutFile "$HOME\gitleaks\gitleaks.zip"
Expand-Archive -Path "$HOME\gitleaks\gitleaks.zip" -DestinationPath "$HOME\gitleaks" -Force
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$HOME\gitleaks", "User")
$env:PATH += ";$HOME\gitleaks"
Remove-Item "$HOME\gitleaks\gitleaks.zip" -Force
gitleaks version
```
The User PATH change applies to new terminals, so reopen the shell. The binary then also works from Git Bash.

**Local full-history scan** (Git Bash): `gitleaks git -v` scanned 6 commits; no leaks found.

**Install pre-commit and add the hook:**
```bash
pip install pre-commit
pre-commit --version
git switch -c feature/gitleaks-scanning
# create .pre-commit-config.yaml (local hook: gitleaks git --pre-commit --staged --redact -v)
pre-commit install
```
Used a `repo: local` hook (`language: system`) that runs the installed binary. It avoids pre-commit building gitleaks from source on Windows, but the hook version is not pinned in the repo.

**Workflow:** added `.github/workflows/secret-scan.yml` (`gitleaks/gitleaks-action@v3`, `fetch-depth: 0`, `contents: read`, `pull-requests: read`, concurrency group). It triggers on `pull_request`, `push` to `main`, and `workflow_call`, with no path filters. Deleted the Day 5 `oidc-check.yml`. Merged via PR #[N].

**Required check:** added the `gitleaks` check to the `main` ruleset.

**Canary test** (throwaway branch `canary/secret-test`, PR #6, closed unmerged):
- Committed a fake `ghp_...` token in `canary.txt`.
- The hook blocked the commit.
- Bypassed it once with `SKIP=gitleaks git commit`, then pushed and opened the PR.
- Deleted the branch locally and remotely.

## 3. Verification
- `gitleaks version` works; the local history scan found nothing.
- The pre-commit hook passed on the clean commit and failed on the canary.
- Clean PR: `secret-scan / gitleaks` green, merged.
- Canary PR #6: the job failed with `RuleID: github-pat`, `leaks found: 1`, and the value redacted. The merge button was blocked by the required check.
- Evidence: screenshots of the blocked hook, the failed check, and the blocked merge button.
- Versions: local gitleaks [version]; CI gitleaks 8.24.3.

## 4. Issues: symptom → root cause → fix
- **Symptom:** `Resource not accessible by integration` when the action tried to comment on the PR.
  **Root cause:** commenting needs `pull-requests: write`, and the workflow has only `read`. The action's message blaming "too large of a diff" is misleading.
  **Fix:** none on purpose. Findings are still in the job summary and the SARIF artifact, and the scan doesn't need write access.
- **Symptom:** `url.parse()` deprecation warning.
  **Root cause:** the action's own Node code.
  **Fix:** none (ignored).

## 5. Learned
A pre-commit hook gives fast feedback but can be bypassed (`SKIP=`), so the required CI check is the real gate. It runs on every PR without path filters, and `workflow_call` lets the future app pipeline reuse it as its first job. If a real secret is ever exposed, revoke and rotate it first, because rewriting git history does not un-compromise it.

Docs:
- https://github.com/gitleaks/gitleaks
- https://github.com/gitleaks/gitleaks-action
- https://pre-commit.com
- https://docs.github.com/en/actions/sharing-automations/reusing-workflows
- https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets