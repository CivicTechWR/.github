# Gitleaks Response Guide

When this workflow comments on your pull request, it means Gitleaks spotted something that looks like a credential. The build stays green, but secrets must be handled quickly and carefully.

## Quick Checklist

1. Stop merging the pull request until the investigation is complete.
2. Open the PR comment or downloaded `gitleaks-report.json` artifact to review the redacted findings.
3. Confirm whether each finding is a real secret or a false positive.
4. Follow the appropriate section below and leave a short update on the PR comment.

## If the Secret Is Real

- **Rotate / revoke the credential immediately.** Use the relevant provider console or contact the owner of the secret.
- **Purge the secret from Git history.** Remove it locally (including from previous commits) and force-push a clean branch.
  - Prefer [`git filter-repo`](https://github.com/newren/git-filter-repo) (or `git filter-branch` as a fallback) to rewrite history.
  - After rewriting history, regenerate the artifact (`git commit --amend` or new commit) and push with `--force-with-lease`.
- **Document replacements.** Update `.env.example` or secrets management notes so others know the new credential.
- **Confirm the fix.** Run `gitleaks detect --redact` locally to be sure the repository is clean before merging.
- **Update the PR comment.** Reply to the workflow comment summarizing what was rotated and how the history was cleaned (no need to paste secrets).

## If It Is a False Positive

- **Verify carefully.** Make sure the redacted value truly is benign (example: test data, dummy keys, or hashed values).
- **Mask the pattern going forward.** Add a rule or `allowlist` entry in your repository-level `.gitleaks.toml` or configuration file, and commit that change with a note explaining the rationale.
- **Re-run locally.** Validate that `gitleaks detect --redact` reports no findings after the rule is added.
- **Reply on the PR.** Note that the finding is a false positive and link to the configuration change.

## Need Help?

- Mention `@CivicTechWR/organizers` in the pull request comment if you need support rotating credentials or cleaning history.
- For high-impact secrets (production credentials, user data access), escalate immediately in the organizers' channel or via the documented security contact.

Keeping secrets out of Git protects our community projects. Thank you for responding quickly when Gitleaks speaks up!
