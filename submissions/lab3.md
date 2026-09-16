# Lab 3 — Secure Git

Branch: `feature/lab3`.

Status: in progress. Global SSH signing and the local pre-commit hook are
configured. Commit signature verification, the fake-token test, GitHub
verification, and the history-rewrite exercise remain to be completed.

## Task 1

### SSH signing evidence

Global Git signing configuration:

```text
gpg.format=ssh
user.signingkey=C:/Users/Михаил/.ssh/id_ed25519.pub
commit.gpgsign=true
tag.gpgsign=true
gpg.ssh.allowedSignersFile=C:/Users/Михаил/.config/git/allowed_signers
```

The allowed-signers file associates the public key with
`m.akhmarov@innopolis.university` in the `git` namespace. A separate local test
file was signed with `ssh-keygen -Y sign` and verified with `ssh-keygen -Y verify`:

```text
Good "git" signature for m.akhmarov@innopolis.university with ED25519 key SHA256:XT+TBD/baTrzkrpwNRA6JMjHhzyhyGXSUh1yGdtm+M0 # gitleaks:allow
```

The trailing `# gitleaks:allow` is an added report annotation, not terminal
output: it exempts this public SSH fingerprint from a `generic-api-key` false
positive. It does not exclude the report or any directory from scanning.

This checks key usability, not a commit signature. The first verification
attempt through a PowerShell text pipeline failed because the pipeline changed
the input bytes; verification using the original file bytes succeeded.

Pending: signed commit, `git log --show-signature -1` output, and GitHub commit
link with verification status. The public GitHub signing-key list for
`etern1ty22` was empty when checked; the key still needs registration as a
Signing Key.

### Repudiation

In this repository, someone could set an arbitrary Git author name and email
and attribute a weakened Juice Shop deployment configuration or an altered
Lab 2 threat model to another student. A verified signature ties the signed
commit to a key associated with the GitHub account, making author-line forgery
alone insufficient; it does not establish that the changes are secure or that
the signing key was not compromised.

## Task 2

### Hook configuration

The configuration is [`.pre-commit-config.yaml`](../.pre-commit-config.yaml).
The remote tags were checked with `git ls-remote --tags`:

```text
83d9cd684c87d95d656c1458ef04895a7f1cbd8e  refs/tags/v8.30.1
3e8a8703264a2f4a69428a0aa4dcb512790b2c8c  refs/tags/v6.0.0
```

It enables `gitleaks`, `detect-private-key`, and `check-added-large-files`.
Installed tools: pre-commit `4.6.2` and git-filter-repo `2.47.0`, in the ignored
repository-local `.venv`. The gitleaks hook was built from the pinned tag above;
its `version` command reports `version is set by build process` rather than a
release number for this source build.

Commands executed on Windows:

```powershell
./.venv/Scripts/pre-commit.exe install
./.venv/Scripts/pre-commit.exe run --all-files
```

Observed output (installation progress omitted):

```text
pre-commit installed at .git\hooks\pre-commit
Detect hardcoded secrets.................................................Passed
detect private key.......................................................Failed
- hook id: detect-private-key
- exit code: 1

Private key found: labs/lab6/vulnerable-iac/ansible/configure.yml

check for added large files..............................................Passed
```

The all-files run exited with code 1. The flagged file is an existing Lab 6
exercise: lines 10–14 explicitly demonstrate plaintext secrets using a
truncated private-key block. Its directory README confirms that these are
intentionally vulnerable educational fixtures. The fixture was preserved and
no exclusion was added to suppress the finding.

The pinned gitleaks hook invokes `gitleaks git --pre-commit --redact --staged
--verbose`; it scans staged changes even when pre-commit receives `--all-files`.
The index was empty for this run, so its Passed result is not evidence that the
entire working tree or history is free of secrets. The two newly created Lab 3
files were still untracked at this point.

Pending: blocked `github-pat` commit and confirmation that HEAD did not change.

### Initial commit attempt: public-fingerprint false positive

The first `git commit -m "test: first signed commit"` was blocked before a
commit was created. Gitleaks reported `RuleID: generic-api-key` for the public
SSH SHA256 fingerprint in this report (then line 28); `detect-private-key`
and `check-added-large-files` both passed. A line-scoped `gitleaks:allow`
annotation was added for the known public fingerprint. This is not the required
`github-pat` test, which remains pending.

### Allowlisting documentation examples

An allowlist entry in `.gitleaks.toml` should match only a reviewed, deliberately
fake example as narrowly as possible. It becomes unsafe if a broad expression
also matches usable credentials or if an allowlisted value is later used as a
real credential; an `AKIA` prefix alone is not a safe exception.

Excluding all of `docs/` suppresses scanning for any credential accidentally
placed there, including copied commands, logs, and configuration examples.
It becomes unsafe as soon as real secrets can enter that directory; narrowly
allowlisting a known fake example retains more protection for other content.

## Bonus

Pending: isolated sandbox, before/after logs, the refusal message from
`git filter-repo`, measured counts, and two observations from actual execution.

The exposed credential must be revoked or rotated, and consumers updated to
use the replacement. Rewriting history alone cannot invalidate copies already
obtained from clones, forks, caches, or logs.
