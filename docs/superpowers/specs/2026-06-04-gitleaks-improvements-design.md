# Gitleaks Improvements Design

**Date:** 2026-06-04
**Issues:** #11, #12, #13, #14
**Branch:** to be created from `main`

## Problem Statement

Four gaps remain after PR #10:

1. Secrets found on a direct push to `main` produce no team notification.
2. The break-glass procedure has no recovery path if the merge step fails, and does not warn that `--admin` bypasses CI checks.
3. Gitleaks scans the full git history on every run, causing recurring false alerts for previously-rotated secrets.
4. Minor cleanup: PR comment spam, missing CI re-run step in response guide, imprecise README CODEOWNERS wording, implicit `--config` pickup.

---

## Section 1 — Workflow Restructure (Issues #11, #13)

### Approach

Split the single workflow into two jobs within the same file:

**Job: `scan`** — triggers on `push` and `pull_request`

Scan scope:
- `pull_request`: `--log-opts "origin/${{ github.base_ref }}..HEAD"` — new commits only
- `push`: `--log-opts "${{ github.event.before }}..${{ github.event.after }}"` — pushed range only

Notification on findings:
- `pull_request`: post PR comment (existing behaviour, unchanged)
- `push`: open a GitHub issue labelled `security`, mentioning `@CivicTechWR/organizers`, with the same redacted findings list format used in PR comments

**Job: `audit`** — triggers on `schedule` (Mondays 9am UTC) and `workflow_dispatch`

- Full `gitleaks detect --source . --config .gitleaks.toml` with no `--log-opts`
- On findings: open a GitHub issue labelled `security`
- No PR comment step (no PR context on scheduled runs)
- `workflow_dispatch` allows a manual full-history scan at any time

### Why two jobs

PR/push scanning and periodic auditing are different jobs with different purposes:
- **Scan**: "did this change introduce a secret?" — fast, scoped, blocking
- **Audit**: "is the repo clean?" — thorough, non-blocking to developer workflow

Conflating them causes alert fatigue on every PR from previously-addressed historical findings.

### Issue format for push findings

```
## Gitleaks detected secrets on a push to main

@CivicTechWR/organizers — Gitleaks flagged **N** potential secret(s) in a
direct push to `main` by @<actor> (commit <sha>).

<redacted findings list>

See the [Gitleaks response guide](<RESPONSE_GUIDE_URL>) for next steps.
```

---

## Section 2 — Break-Glass Doc Improvements (Issue #12)

Two targeted additions to `docs/governance/codeowners-branch-protection.md`:

**After the `gh pr merge` command (step 3):**

> **If the merge fails:** Run step 4 (re-enable `enforce_admins`) immediately before investigating the failure. Do not leave branch protection disabled while you troubleshoot.

**Before the merge command:**

> **Note:** `--admin` overrides *all* branch protection rules including required CI checks (gitleaks, lint, tests). Before using it, manually confirm the change has been reviewed and the gitleaks scan has passed on this branch.

---

## Section 3 — Cleanup Bundle (Issue #14)

### 3a. PR comment deduplication (`gitleaks.yml`)

Before creating a PR comment, search for an existing comment from `github-actions[bot]` containing the HTML marker `<!-- gitleaks-findings -->`. If found, update it with `updateComment`. If not found, create a new one with `createComment`. The marker is embedded as an HTML comment in the body so it is invisible to readers.

### 3b. CI re-run step in response guide (`docs/gitleaks-response.md`)

After the existing "Confirm the fix" step (local `gitleaks detect --redact`), add:

> Push the cleaned branch and confirm the gitleaks workflow passes in CI before merging.

### 3c. README CODEOWNERS wording (`README.md`)

Move `CODEOWNERS` out of the "Default files that appear in all repositories" bullet list. Replace with an inline note:

> **CODEOWNERS** — not propagated by GitHub to other repositories; each project must add its own. See [codeowners-branch-protection.md](docs/governance/codeowners-branch-protection.md).

### 3d. Explicit `--config` flag

Add `--config .gitleaks.toml` to:
- The `gitleaks detect` command in `gitleaks.yml` (both `scan` and `audit` jobs)
- The `gitleaks detect --redact` verification command in `docs/gitleaks-response.md`

---

## Files Changed

| File | Change |
|------|--------|
| `.github/workflows/gitleaks.yml` | Scope scan triggers; add push-findings issue step; add `audit` job; add `--config` flag; dedup PR comments |
| `docs/governance/codeowners-branch-protection.md` | Add break-glass recovery note and `--admin` CI warning |
| `docs/gitleaks-response.md` | Add CI re-run step; add `--config` to local command |
| `README.md` | Fix CODEOWNERS wording |

## Out of Scope

- Slack notifications (no available app slots on free plan)
- Changes to `.gitleaks.toml` or other workflow files
- Any changes to branch protection settings
