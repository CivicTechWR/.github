# Gitleaks Response Guide

This guide helps CivicTechWR maintainers triage, contain, and remediate potential secrets flagged by automated Gitleaks scans.

## 1. Triage the Finding

- Review the redacted report attached to the workflow run or pull request comment to identify the file and rule triggered.
- Decide whether the flagged value is a real secret, test data, or a false positive.

## 2. Contain the Exposure

- Remove the secret from the repository immediately (revert the change or follow up with a commit that deletes it and uses environment variables or secrets instead).
- Rotate or invalidate the exposed credential in the affected service (Supabase, Slack, third-party API, etc.).

## 3. Clean the History (If Needed)

- If the secret landed in commit history, coordinate with maintainers before rewriting history.
- Use `git filter-repo` or provider tooling to purge historical references; ensure downstream forks are notified.

## 4. Communicate

- Document the remediation steps in the pull request or issue, noting credential rotation status.
- For high-risk disclosures, email `civictechwr@gmail.com` to escalate privately.

## 5. Verify and Prevent Recurrence

- Re-run the Gitleaks workflow (or trigger it manually) to confirm the repository is clean.
- Add new patterns to the `gitleaks.toml` allowlist only after validating they are false positives.

## Reference

- Repository workflow: `.github/workflows/reusable-gitleaks.yml`
- Team contact: `civictechwr@gmail.com`

Always assume a leaked credential is compromised until rotated and confirmed inactive.
