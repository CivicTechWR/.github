# Gitleaks Response Guide

When this workflow comments on your pull request, it means Gitleaks spotted something that looks like a credential. The build stays green, but secrets must be handled quickly and carefully.

## Quick Checklist

1. **Stop merging** the pull request until the investigation is complete.
2. Open the PR comment or downloaded `gitleaks-report.json` artifact to review the redacted findings.
3. Confirm whether each finding is a real secret or a false positive.
4. Follow the appropriate section below and leave a short update on the PR comment.

## If the Secret Is Real

- **Rotate / revoke the credential immediately.** Use the relevant provider console (Supabase, Slack, third-party API, etc.) or contact the owner of the secret.
- **Purge the secret from Git history.** Remove it locally (including from previous commits) and force-push a clean branch.
  - Prefer [`git filter-repo`](https://github.com/newren/git-filter-repo) (or `git filter-branch` as a fallback) to rewrite history.
  - After rewriting history, regenerate the artifact (`git commit --amend` or new commit) and push with `--force-with-lease`.
  - If the secret landed in commit history already merged to the default branch, coordinate with maintainers before rewriting — downstream forks may need notification.
- **Document replacements.** Update `.env.example` or secrets management notes so others know the new credential.
- **Confirm the fix.** Run `gitleaks detect --redact` locally to verify the repository is clean before merging.
- **Update the PR comment.** Reply to the workflow comment summarizing what was rotated and how history was cleaned (no need to paste secrets).
- **For high-impact secrets** (production credentials, user data access), escalate immediately in the organizers' channel or email `civictechwr@gmail.com`.

## If It Is a False Positive

- **Verify carefully.** Make sure the redacted value is truly benign (example: test data, dummy keys, or hashed values).
- **Mask the pattern going forward.** Add an `allowlist` entry in your repository-level `.gitleaks.toml` and commit that change with a note explaining the rationale.
- **Re-run locally.** Validate that `gitleaks detect --redact` reports no findings after the allowlist entry is added.
- **Reply on the PR.** Note that the finding is a false positive and link to the configuration change.

## Need Help?

- Mention `@CivicTechWR/organizers` in the pull request comment if you need support rotating credentials or cleaning history.

## Reference

- Repository workflow: `.github/workflows/gitleaks.yml`
- Team contact: `civictechwr@gmail.com`

Always assume a leaked credential is compromised until rotated and confirmed inactive.
