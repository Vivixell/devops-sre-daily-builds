# Day 4: Terraform bootstrap

**Pass:** Pass 1 (AI-assisted) | **Status:** Done | **Time:** [X mins]

## 1. Goal and acceptance criteria

Implement the bootstrap layer and move its state to the remote backend. Done when the configuration validates, a reviewed plan was applied, state lives in S3 and a plan after migration shows no changes.

## 2. Commands and steps

Files in `terraform/bootstrap/`: `versions.tf`, `providers.tf`, `variables.tf`, `state-bucket.tf`, `ecr.tf`, `oidc.tf`, `outputs.tf`, and `backend.tf` (added after the first apply).

```bash
git mv bootstrap terraform/bootstrap
terraform fmt -check
terraform init
terraform validate
terraform plan -out=tfplan      # reviewed before applying
terraform apply tfplan          # local state; included oidc.tf
# add backend.tf (bucket, key=bootstrap/terraform.tfstate, encrypt, use_lockfile)
terraform init -migrate-state   # answered yes to copy state
terraform plan                  # no changes expected
aws s3 ls s3://<state-bucket>/bootstrap/
```

- Resources: state bucket with versioning, encryption, public access block, lifecycle rule and TLS-only policy; one ECR repository per service with immutable tags, scan on push and a keep-last-20 lifecycle policy; GitHub OIDC provider; deploy role with no permissions yet.
- The OIDC trust is limited to the project repo on `refs/heads/main` and `pull_request`, with the `sts.amazonaws.com` audience.
- The GitHub repo variable was moved from `oidc.tf` to `variables.tf` with no change in behaviour.
- After confirming the state in S3, the local `terraform.tfstate` and backup were deleted. Both are gitignored.
- Versions: AWS provider 6.69.0. Terraform started at 1.14.7 and was upgraded to 1.16.6 (see Issue 2). `required_version = ">= 1.11.0, < 2.0.0"` was not changed. `.terraform.lock.hcl` is committed.

## 3. Verification

- `terraform validate` passed and the plan was reviewed before apply.
- `terraform init -migrate-state` reported: Successfully configured the backend "s3".
- State object present at `bootstrap/terraform.tfstate` in the bucket.
- Plan after migration: [No changes / paste result].
- Lock observed as a `.tflock` object during an apply prompt: [yes / not tested].

## 4. Issues: symptom → root cause → fix

Two separate problems overlapped on this day. The first (wrong profile) can happen on any new AWS account, so the diagnosis commands are collected here as a reusable reference.

### 4.1 Command reference (AWS identity troubleshooting)

**Where each command runs**

| Shell | Works there |
|---|---|
| Any shell (CMD, PowerShell, Git Bash, WSL) | `aws ...`, `terraform ...` |
| PowerShell only | `$env:AWS_PROFILE`, `Get-ChildItem Env:AWS*`, `[Environment]::...`, `Remove-Item Env:...` |
| Git Bash / WSL only | `env \| grep AWS`, `export`, `unset`, `eval "$(...)"`, `grep ... ~/.bashrc` |
| CMD only | `set AWS`, `set AWS_PROFILE=...` |

`env` and `grep` are Unix commands, so they fail in PowerShell and CMD.

**Diagnose: which identity am I, and why?**

| Command | Shell | What it tells me |
|---|---|---|
| `aws --version` | Any | CLI version. `aws login` needs 2.32.0 or later. |
| `aws sts get-caller-identity` | Any | The identity AWS actually sees: account, user or role. This is the final truth check. |
| `aws configure list` | Any | Which profile is active and **where it was chosen from**. Read the `TYPE` column: `env` means an environment variable picked it, `config-file` means the config file did. |
| `aws configure list-profiles` | Any | Every profile name the CLI knows about. |
| `aws configure list --profile <name>` | Any | Details for one profile. Also reports `Your session has expired` for `aws login` profiles that need a fresh login. |
| `env \| grep AWS` | Git Bash / WSL | All `AWS_*` variables set in this shell. |
| `Get-ChildItem Env:AWS*` | PowerShell | Same, for PowerShell. |
| `set AWS` | CMD | Same, for CMD. |
| `$env:AWS_PROFILE` | PowerShell | The profile variable in this terminal (Process scope). |
| `[Environment]::GetEnvironmentVariable("AWS_PROFILE","User")` | PowerShell | The permanent value for my Windows user. Empty output is normal if it isn't set there. |
| `[Environment]::GetEnvironmentVariable("AWS_PROFILE","Machine")` | PowerShell | The value for all users on the machine. |
| `[Environment]::GetEnvironmentVariable("AWS_DEFAULT_PROFILE","User")` | PowerShell | The legacy variable, in case it is set instead. |
| `grep -n AWS ~/.bashrc ~/.bash_profile ~/.profile` | Git Bash / WSL | Finds an `export AWS_PROFILE=...` line in a shell startup file. |
| `terraform version` | Any | Terraform version and provider versions. |

**Fix: choose the right identity**

| Command | Shell | Effect |
|---|---|---|
| `aws login` | Any | Fresh browser login for the selected profile. Sessions refresh automatically for up to 12 hours, then need this again. |
| `export AWS_PROFILE=<project-profile>` | Git Bash / WSL | Select the profile for this terminal only. |
| `$env:AWS_PROFILE = "<project-profile>"` | PowerShell | Same, for this terminal only. |
| `set AWS_PROFILE=<project-profile>` | CMD | Same, for this terminal only. |
| `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN` | Git Bash / WSL | Removes leftover credential variables. They override `AWS_PROFILE`. |
| `unset AWS_PROFILE` | Git Bash / WSL | Removes a stale profile variable from this terminal. |
| `Remove-Item Env:AWS_PROFILE` | PowerShell | Same, for PowerShell. |
| `[Environment]::SetEnvironmentVariable("AWS_PROFILE","<project-profile>","User")` | PowerShell | Permanent default for my Windows user. Only terminals opened **afterwards** see it. |
| `eval "$(aws configure export-credentials --profile <project-profile> --format env)"` | Git Bash / WSL | Exports temporary keys into this shell (they last about 15 minutes). A fallback, not a fix. |

**Never** run `aws configure` to make a profile permanent. It writes static access keys.

**How the CLI chooses (simplified)**

Which profile (first match wins):
1. `--profile <name>` on the command
2. `AWS_PROFILE` environment variable
3. `AWS_DEFAULT_PROFILE` environment variable (legacy)
4. The `[default]` profile

Which credentials (first source that provides them wins):
1. Command-line options
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
3. The selected profile (login session, SSO, assume-role, credentials file, config file)
4. Container credentials
5. EC2 instance metadata (IMDS)

The CLI never falls back to a different profile. If the selected profile's session expires, commands fail until `aws login`.

**Standard check before any Terraform command**

```bash
aws configure list                # right profile, and where does TYPE say it came from?
aws sts get-caller-identity       # right account?
```

### 4.2 Issue 1: the wrong profile is active

**Symptoms**

1. `aws login` failed: `Profile '<stale-profile>' is already configured with Access Key credentials.`
2. `terraform plan` failed with `InvalidClientTokenId` (403 on STS GetCallerIdentity).
3. `aws sts get-caller-identity` and `aws configure list` both reported the stale profile, with `TYPE: env`.

**Diagnosis (in the order I ran it)**

```bash
# any shell
aws --version
aws sts get-caller-identity
aws configure list
aws configure list-profiles
aws configure list --profile <project-profile>
```

```bash
# Git Bash / WSL only
env | grep AWS
```

```powershell
# PowerShell only
$env:AWS_PROFILE
[Environment]::GetEnvironmentVariable("AWS_PROFILE","User")
[Environment]::GetEnvironmentVariable("AWS_PROFILE","Machine")
[Environment]::GetEnvironmentVariable("AWS_DEFAULT_PROFILE","User")
```

**Findings**

- `TYPE: env` meant an environment variable was choosing the profile, not the config file.
- Git Bash showed `AWS_PROFILE` set to the stale profile while WSL showed nothing, so the value was per-terminal, not machine-wide.
- User and Machine scope returned nothing. The stale value was only in the Process environment the terminal had inherited, probably from a parent app that started earlier. How it got there was not confirmed.
- The project profile also reported `Your session has expired`, so it needed a fresh `aws login`.

**Fix**

```powershell
# PowerShell only: permanent default for my Windows user
[Environment]::SetEnvironmentVariable("AWS_PROFILE", "<project-profile>", "User")
```

Close the terminal and open a new one. The old terminal keeps its inherited environment, so `aws configure list` still shows the stale profile right after the Set command. Only a new terminal picks up the change.

**Recurrence (Day 5 morning)**

The stale profile showed up again in Git Bash, in `aws sts get-caller-identity`, `env | grep AWS` and `aws configure list`. A User-scope change only reaches processes started after it, and only if nothing else overrides it. If it happens again, find where the value comes from, in this order:

```bash
# Git Bash / WSL
env | grep AWS                                         # is it set in this shell?
grep -n AWS ~/.bashrc ~/.bash_profile ~/.profile       # exported by a startup file?
```

```powershell
# PowerShell
Get-ChildItem Env:AWS*                                 # Process scope
[Environment]::GetEnvironmentVariable("AWS_PROFILE","User")
[Environment]::GetEnvironmentVariable("AWS_PROFILE","Machine")
```

Also fully close the app that launched the terminal (editor, terminal host) and reopen it, because its children inherit its environment.

Found in: [fill in: startup file / parent app / Machine scope / other]
Fix applied: [fill in]

For a quick override in the current terminal, set `AWS_PROFILE` with the commands in 4.1.

**Verify**

```bash
# any shell, in a new terminal
aws configure list                # profile: <project-profile>
aws login
aws sts get-caller-identity       # expected account
```

**Learned**

Check three layers in order: selection (`aws configure list`, read `TYPE`), credentials, and validity (`aws sts get-caller-identity`). A stale environment variable beats the config file, and a permanent User-scope change only reaches terminals started after it.

### 4.3 Issue 2: `terraform init` fails with the IMDS error while the AWS CLI works

**Symptom**

```
Error: No valid credential sources found
Error: failed to refresh cached credentials, no EC2 IMDS role found,
operation error ec2imds: GetMetadata, request canceled, context deadline exceeded
```

**Evidence**

In the same Git Bash window, `aws sts get-caller-identity` returned the correct IAM user while `terraform init` failed. That rules out an expired session and a wrong profile for that occurrence. `plan` and `apply` had worked earlier.

**Root cause**

The S3 backend resolves credentials separately from the `provider "aws"` block. It uses the SDK built into the Terraform binary, not the provider. Terraform 1.14.7's backend cannot read `aws login` (`login_session`) profiles, while the provider (6.69.0) can, which is why `plan` and `apply` worked. With no usable credentials, the chain fell through to EC2 metadata (IMDS), which does not exist on a laptop, and timed out.

The first IMDS failure happened while the wrong profile was also active, so it may have had both causes. The second occurrence isolates the version cause.

**Fix**

Upgrade Terraform to 1.15.0 or later. The 1.15.0 release added `backend/s3: Support authentication via aws login`.

- Release notes: https://github.com/hashicorp/terraform/releases/tag/v1.15.0
- Feature issue: https://github.com/hashicorp/terraform/issues/37976

I downloaded the Windows zip from releases.hashicorp.com and replaced `terraform.exe` in the Terraform install folder. Then, in a new terminal:

```bash
terraform version                 # now v1.16.6
aws login
terraform init                    # worked, no other changes needed
```

No change was needed in `backend.tf`, `required_version`, or the credentials setup. Do not downgrade after applying with a newer version, because older Terraform refuses state written by a newer one.

**Fallbacks (not needed after the upgrade, not tested)**

```bash
# Git Bash only: export temporary keys
eval "$(aws configure export-credentials --profile <project-profile> --format env)"
```

```ini
# ~/.aws/config: a separate profile for Terraform.
# Never put credential_process on the login profile itself,
# because `aws login` refuses to run on such a profile.
[profile <project-profile>-tf]
credential_process = aws configure export-credentials --profile <project-profile> --format process
```

Use it with `AWS_PROFILE=<project-profile>-tf terraform init`.

**Learned**

The Terraform backend and the provider authenticate separately. If `aws sts get-caller-identity` works but `init` shows the IMDS error, suspect the Terraform version first, then the active profile.

### 4.4 What the backend stores, and what it doesn't

| Thing | Where it lives | What it holds |
|---|---|---|
| `required_version` | `.tf` files in git | A constraint, not a stored version. `>= 1.11.0, < 2.0.0` allows 1.14.7 and 1.16.6, so nothing needed changing. |
| `.terraform.lock.hcl` | the repo | The locked provider versions (AWS 6.69.0), not Terraform's. |
| State | the S3 bucket at `bootstrap/terraform.tfstate` | The managed resources and their attributes. |

Inside the state file, Terraform also records `terraform_version` (the version that last wrote it), a `serial` that increments on every change, and a `lineage` ID. During a run, the S3 backend creates a `.tflock` object to stop two runs changing state at once (`use_lockfile`). The local `.terraform/` folder caches the backend settings from the last `init`.

A newer Terraform can read state written by an older one. After applying with the newer version, `terraform_version` updates and an older Terraform will refuse that state. To enforce the new minimum for everyone, `required_version` could become `>= 1.15.0, < 2.0.0`. That is optional.

## 5. Learned

Native S3 locking removes the need for a DynamoDB table, and a locally applied bootstrap layer can move into its own bucket afterwards with `init -migrate-state`. State holds real resource data, so the bucket is versioned, encrypted and blocked from public access, and old versions are expired after 90 days.

When using a new AWS account, run `aws configure list` and `aws sts get-caller-identity` first. Then diagnose with the table in 4.1. The provider and the backend authenticate independently, so a working CLI does not guarantee a working `terraform init`.

Docs:
- S3 backend: https://developer.hashicorp.com/terraform/language/backend/s3
- AWS provider authentication: https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication-and-configuration
- OIDC provider resource: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider
- AWS CLI `aws login`: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html