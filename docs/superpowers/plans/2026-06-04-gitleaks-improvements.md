# Gitleaks Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scope gitleaks scans to new commits, add push-to-main team notification, add a weekly full-history audit job, harden the break-glass doc, and apply minor cleanup across the workflow and docs.

**Architecture:** Two jobs in the existing workflow file — `scan` (scoped, runs on push/PR) and `audit` (full history, runs on schedule/dispatch). Four small doc edits across three markdown files.

**Tech Stack:** GitHub Actions, `actions/github-script@v7`, gitleaks v8.30.1, Bash, Markdown.

**Spec:** `docs/superpowers/specs/2026-06-04-gitleaks-improvements-design.md`

---

## File Map

| File | What changes |
|------|-------------|
| `.github/workflows/gitleaks.yml` | Add `issues: write` permission; add `schedule` trigger; scope `scan` job with `if:`; add `Set scan range` step; add `--config` + `--log-opts` to detect; add push-findings issue step; dedup PR comment; add `audit` job |
| `docs/governance/codeowners-branch-protection.md` | Add recovery note + `--admin` CI warning to break-glass section |
| `docs/gitleaks-response.md` | Add CI re-run step; add `--config` to local verify command |
| `README.md` | Fix imprecise CODEOWNERS bullet |

---

## Task 1: Create branch

- [ ] **Create and switch to feature branch**

  ```bash
  git checkout -b fix/gitleaks-improvements
  ```

---

## Task 2: Update `on:` triggers, permissions, and scan job `if:`

**File:** `.github/workflows/gitleaks.yml`

- [ ] **Replace the `on:` block (lines 3–6) to add `schedule`**

  Replace:
  ```yaml
  on:
    push:
    pull_request:
    workflow_dispatch:
  ```
  With:
  ```yaml
  on:
    push:
    pull_request:
    schedule:
      - cron: '0 9 * * 1'
    workflow_dispatch:
  ```

- [ ] **Add `issues: write` to the permissions block (after line 10)**

  Replace:
  ```yaml
  permissions:
    contents: read
    pull-requests: write
  ```
  With:
  ```yaml
  permissions:
    contents: read
    pull-requests: write
    issues: write
  ```

- [ ] **Add `if:` condition to the `scan` job (after `runs-on: ubuntu-latest` on line 15)**

  Replace:
  ```yaml
    name: Scan for Leaked Secrets
    runs-on: ubuntu-latest
    env:
  ```
  With:
  ```yaml
    name: Scan for Leaked Secrets
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    env:
  ```

- [ ] **Commit**

  ```bash
  git add .github/workflows/gitleaks.yml
  git commit -m "ci: add schedule trigger, issues permission, scope scan job to push/PR"
  ```

---

## Task 3: Add scan range step and update `gitleaks detect` command

**File:** `.github/workflows/gitleaks.yml`

- [ ] **Insert `Set scan range` step after the `Install Gitleaks CLI` step (after line 35)**

  The new step goes between `Install Gitleaks CLI` and `Run Gitleaks (redacted)`. Add it so the file reads:

  ```yaml
        env:
          GITLEAKS_VERSION: ${{ env.GITLEAKS_VERSION }}

      - name: Set scan range
        id: range
        env:
          EVENT_NAME: ${{ github.event_name }}
          BASE_REF: ${{ github.base_ref }}
          BEFORE: ${{ github.event.before }}
          AFTER: ${{ github.event.after }}
        run: |
          if [ "$EVENT_NAME" = "pull_request" ]; then
            echo "log_opts=origin/${BASE_REF}..HEAD" >> "$GITHUB_OUTPUT"
          elif [ "$BEFORE" = "0000000000000000000000000000000000000000" ]; then
            echo "log_opts=HEAD~1..HEAD" >> "$GITHUB_OUTPUT"
          else
            echo "log_opts=${BEFORE}..${AFTER}" >> "$GITHUB_OUTPUT"
          fi

      - name: Run Gitleaks (redacted)
  ```

  The `0000000000000000000000000000000000000000` check handles new-branch pushes where `before` is a null SHA.

- [ ] **Update the `Run Gitleaks (redacted)` step to add `--config` and `--log-opts`**

  Replace:
  ```yaml
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
        run: |
          gitleaks detect \
            --source . \
            --no-banner \
            --redact \
            --report-format json \
            --report-path gitleaks-report.json
  ```
  With:
  ```yaml
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
          LOG_OPTS: ${{ steps.range.outputs.log_opts }}
        run: |
          gitleaks detect \
            --source . \
            --config .gitleaks.toml \
            --no-banner \
            --redact \
            --report-format json \
            --report-path gitleaks-report.json \
            --log-opts "$LOG_OPTS"
  ```

  `LOG_OPTS` is passed via env rather than interpolated directly in the `run:` block to avoid shell injection.

- [ ] **Commit**

  ```bash
  git add .github/workflows/gitleaks.yml
  git commit -m "ci: scope scan to new commits via --log-opts, add explicit --config flag"
  ```

---

## Task 4: Add push-findings GitHub issue step

**File:** `.github/workflows/gitleaks.yml`

- [ ] **Insert new step after `Fail on scan error` and before `Add pull request comment`**

  Add after the `exit 1` line and before `- name: Add pull request comment`:

  ```yaml
      - name: Open security issue (push findings)
        if: ${{ steps.summarize.outputs.found == 'true' && github.event_name == 'push' }}
        uses: actions/github-script@v7
        env:
          LEAK_COUNT: ${{ steps.summarize.outputs.count }}
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const findings = JSON.parse(fs.readFileSync('gitleaks-report.json', 'utf8'));
            const responseGuide = process.env.RESPONSE_GUIDE_URL;
            const sha = context.sha.substring(0, 7);

            const lines = findings.slice(0, 10).map((f) => {
              const file = f.file ? `\`${f.file}\`` : 'unknown file';
              const line = f.startLine ? ` (line ${f.startLine})` : '';
              const rule = f.ruleId || f.rule || 'potential secret';
              return `- ${file}${line} — ${rule}`;
            }).join('\n');

            const remainder = findings.length > 10
              ? `\n\n…and ${findings.length - 10} more finding(s) in the workflow artifact.`
              : '';

            const body = [
              '<!-- gitleaks-push-findings -->',
              `@CivicTechWR/organizers — Gitleaks flagged **${process.env.LEAK_COUNT}** potential secret(s) in a direct push to \`${context.ref}\` by @${context.actor} (commit \`${sha}\`).`,
              '',
              lines,
              remainder,
              '',
              `See the [Gitleaks response guide](${responseGuide}) for next steps.`,
              '',
              '_Secret values are redacted by the scanner._',
            ].filter(Boolean).join('\n');

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[Security] Gitleaks: ${process.env.LEAK_COUNT} potential secret(s) on push by @${context.actor} (${sha})`,
              body,
            });
  ```

- [ ] **Commit**

  ```bash
  git add .github/workflows/gitleaks.yml
  git commit -m "ci: open GitHub issue when secrets detected on push to main"
  ```

---

## Task 5: Dedup PR comment (search-update-or-create)

**File:** `.github/workflows/gitleaks.yml`

- [ ] **Replace the entire `Add pull request comment` step script**

  Replace the `script: |` block content (lines 82–119) with:

  ```yaml
          script: |
            const fs = require('fs');
            const findings = JSON.parse(fs.readFileSync('gitleaks-report.json', 'utf8'));
            const responseGuide = process.env.RESPONSE_GUIDE_URL;
            const MARKER = '<!-- gitleaks-findings -->';

            if (!Array.isArray(findings) || findings.length === 0) {
              return;
            }

            const lines = findings.slice(0, 10).map((finding) => {
              const file = finding.file ? `\`${finding.file}\`` : 'unknown file';
              const line = finding.startLine ? ` (line ${finding.startLine})` : '';
              const rule = finding.ruleId || finding.rule || 'potential secret';
              return `- ${file}${line} — ${rule}`;
            }).join('\n');

            const remainder = findings.length > 10
              ? `\n\n…and ${findings.length - 10} more finding${findings.length - 10 === 1 ? '' : 's'} in the attached report.`
              : '';

            const body = [
              MARKER,
              `@CivicTechWR/organizers Gitleaks flagged **${process.env.LEAK_COUNT}** potential secret${findings.length === 1 ? '' : 's'} in this pull request.`,
              '',
              `Please review the [Gitleaks response guide](${responseGuide}) for next steps.`,
              '',
              lines,
              remainder,
              '',
              '_Secret values are redacted by the scanner. Follow the guide before merging._',
            ].filter(Boolean).join('\n');

            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const existing = comments.data.find(
              c => c.user.login === 'github-actions[bot]' && c.body.includes(MARKER)
            );

            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: existing.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body,
              });
            }
  ```

- [ ] **Commit**

  ```bash
  git add .github/workflows/gitleaks.yml
  git commit -m "ci: dedup PR comment — update existing instead of creating new each push"
  ```

---

## Task 6: Add `audit` job

**File:** `.github/workflows/gitleaks.yml`

- [ ] **Append the `audit` job at the end of the file**

  Add after the last line of the `scan` job:

  ```yaml

    audit:
      name: Weekly Audit — Full History Scan
      runs-on: ubuntu-latest
      if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        RESPONSE_GUIDE_URL: https://github.com/CivicTechWR/.github/blob/main/docs/gitleaks-response.md
        GITLEAKS_VERSION: v8.30.1
      steps:
        - name: Check out code
          uses: actions/checkout@v4
          with:
            fetch-depth: 0

        - name: Install Gitleaks CLI
          run: |
            VERSION_NUMBER="${GITLEAKS_VERSION#v}"
            ARCHIVE="gitleaks_${VERSION_NUMBER}_linux_x64.tar.gz"
            curl -sSfL "https://github.com/gitleaks/gitleaks/releases/download/${GITLEAKS_VERSION}/${ARCHIVE}" -o gitleaks.tar.gz
            tar -xzf gitleaks.tar.gz gitleaks
            sudo install -m 0755 gitleaks /usr/local/bin/gitleaks
            rm -f gitleaks.tar.gz gitleaks
          env:
            GITLEAKS_VERSION: ${{ env.GITLEAKS_VERSION }}

        - name: Run full history audit
          id: gitleaks
          continue-on-error: true
          env:
            GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
          run: |
            gitleaks detect \
              --source . \
              --config .gitleaks.toml \
              --no-banner \
              --redact \
              --report-format json \
              --report-path gitleaks-report.json

        - name: Summarize findings
          id: summarize
          run: |
            if [ ! -f gitleaks-report.json ]; then
              echo "[]" > gitleaks-report.json
            fi
            count=$(jq 'length' gitleaks-report.json 2>/dev/null || echo 0)
            if [ "$count" -gt 0 ]; then
              echo "found=true" >> "$GITHUB_OUTPUT"
              echo "count=$count" >> "$GITHUB_OUTPUT"
              echo "::warning::Gitleaks audit detected ${count} potential secret(s)."
            else
              echo "found=false" >> "$GITHUB_OUTPUT"
              echo "count=0" >> "$GITHUB_OUTPUT"
              echo "::notice::Gitleaks audit found no potential secrets."
            fi

        - name: Fail on scan error
          if: ${{ steps.gitleaks.outcome == 'failure' && steps.summarize.outputs.found == 'false' }}
          run: |
            echo "::error::Gitleaks audit scan ended with an error and produced no report. Review the action logs."
            exit 1

        - name: Open security issue
          if: ${{ steps.summarize.outputs.found == 'true' }}
          uses: actions/github-script@v7
          env:
            LEAK_COUNT: ${{ steps.summarize.outputs.count }}
          with:
            github-token: ${{ secrets.GITHUB_TOKEN }}
            script: |
              const fs = require('fs');
              const findings = JSON.parse(fs.readFileSync('gitleaks-report.json', 'utf8'));
              const responseGuide = process.env.RESPONSE_GUIDE_URL;

              const lines = findings.slice(0, 10).map((f) => {
                const file = f.file ? `\`${f.file}\`` : 'unknown file';
                const line = f.startLine ? ` (line ${f.startLine})` : '';
                const rule = f.ruleId || f.rule || 'potential secret';
                return `- ${file}${line} — ${rule}`;
              }).join('\n');

              const remainder = findings.length > 10
                ? `\n\n…and ${findings.length - 10} more finding(s) in the workflow artifact.`
                : '';

              const body = [
                '<!-- gitleaks-audit-findings -->',
                `@CivicTechWR/organizers — The weekly Gitleaks audit flagged **${process.env.LEAK_COUNT}** potential secret(s) in the full repository history.`,
                '',
                lines,
                remainder,
                '',
                `See the [Gitleaks response guide](${responseGuide}) for next steps.`,
                '',
                '_Secret values are redacted by the scanner._',
              ].filter(Boolean).join('\n');

              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: `[Security] Gitleaks weekly audit: ${process.env.LEAK_COUNT} potential secret(s) detected`,
                body,
              });

        - name: Upload redacted report
          if: ${{ steps.summarize.outputs.found == 'true' }}
          uses: actions/upload-artifact@v4
          with:
            name: gitleaks-audit-report
            path: gitleaks-report.json
            retention-days: 30

        - name: Record audit summary
          if: ${{ always() }}
          env:
            LEAK_COUNT: ${{ steps.summarize.outputs.count }}
            HAS_FINDINGS: ${{ steps.summarize.outputs.found }}
            SCAN_OUTCOME: ${{ steps.gitleaks.outcome }}
          run: |
            if [ "$HAS_FINDINGS" = "true" ]; then
              {
                echo "## Gitleaks Audit Findings"
                echo ""
                echo "Potential secrets detected in full history: $LEAK_COUNT"
                echo "Report: gitleaks-audit-report (uploaded as artifact)"
                echo "Guide: $RESPONSE_GUIDE_URL"
              } >> "$GITHUB_STEP_SUMMARY"
            elif [ "$SCAN_OUTCOME" = "failure" ]; then
              {
                echo "## Gitleaks Audit"
                echo ""
                echo ":x: Audit scan error — gitleaks exited with an error and produced no report. Review the action logs."
              } >> "$GITHUB_STEP_SUMMARY"
            else
              {
                echo "## Gitleaks Audit"
                echo ""
                echo "Full history scan complete. No potential secrets detected."
              } >> "$GITHUB_STEP_SUMMARY"
            fi
  ```

- [ ] **Commit**

  ```bash
  git add .github/workflows/gitleaks.yml
  git commit -m "ci: add weekly full-history audit job (schedule + workflow_dispatch)"
  ```

---

## Task 7: Break-glass doc improvements

**File:** `docs/governance/codeowners-branch-protection.md`

- [ ] **Add `--admin` CI bypass warning before the merge command (before line 101)**

  Replace:
  ```markdown
  3. Merge with admin privileges:
     ```bash
     gh pr merge NUMBER --admin --merge
     ```
  ```
  With:
  ```markdown
  3. Merge with admin privileges:

     > **Note:** `--admin` overrides *all* branch protection rules including required CI checks (gitleaks, lint, tests). Before using it, manually confirm the change has been reviewed and the gitleaks scan has passed on this branch.

     ```bash
     gh pr merge NUMBER --admin --merge
     ```

     > **If the merge fails:** Run step 4 (re-enable `enforce_admins`) immediately before investigating the failure. Do not leave branch protection disabled while you troubleshoot.
  ```

- [ ] **Commit**

  ```bash
  git add docs/governance/codeowners-branch-protection.md
  git commit -m "docs: add break-glass recovery note and --admin CI bypass warning"
  ```

---

## Task 8: Response guide updates

**File:** `docs/gitleaks-response.md`

- [ ] **Add `--config` flag to the local verify command (line 20)**

  Replace:
  ```markdown
  - **Confirm the fix.** Run `gitleaks detect --redact` locally to verify the repository is clean before merging.
  ```
  With:
  ```markdown
  - **Confirm the fix.** Run `gitleaks detect --config .gitleaks.toml --redact` locally to verify the repository is clean before merging. Then push the cleaned branch and confirm the gitleaks workflow passes in CI before merging.
  ```

- [ ] **Add `--config` flag to the false-positive re-run command (line 28)**

  Replace:
  ```markdown
  - **Re-run locally.** Validate that `gitleaks detect --redact` reports no findings after the allowlist entry is added.
  ```
  With:
  ```markdown
  - **Re-run locally.** Validate that `gitleaks detect --config .gitleaks.toml --redact` reports no findings after the allowlist entry is added.
  ```

- [ ] **Commit**

  ```bash
  git add docs/gitleaks-response.md
  git commit -m "docs: add --config flag to local gitleaks commands, add CI re-run step"
  ```

---

## Task 9: Fix README CODEOWNERS wording

**File:** `README.md`

- [ ] **Replace the CODEOWNERS bullet (line 20)**

  Replace:
  ```markdown
  - **CODEOWNERS** - Default repository ownership and review responsibilities
  ```
  With:
  ```markdown
  - **CODEOWNERS** - Ownership for *this* repository only. GitHub does not propagate this file to other repos — each project must add its own. See [codeowners-branch-protection.md](docs/governance/codeowners-branch-protection.md).
  ```

- [ ] **Commit**

  ```bash
  git add README.md
  git commit -m "docs: clarify CODEOWNERS does not auto-propagate to other repos"
  ```

---

## Task 10: Push and open PR

- [ ] **Push branch**

  ```bash
  git push -u origin fix/gitleaks-improvements
  ```

- [ ] **Open PR**

  ```bash
  gh pr create \
    --title "fix: gitleaks workflow hardening and doc cleanup (issues #11–#14)" \
    --body "$(cat <<'EOF'
  ## Summary

  - **Scoped scans** (#13): scan job now uses `--log-opts` to scan only new commits on PRs and pushes, eliminating recurring false alerts from previously-rotated secrets
  - **Weekly audit** (#13): new `audit` job runs full history scan on Mondays 9am UTC (and `workflow_dispatch`) — separates "did this change introduce a secret?" from "is the repo clean?"
  - **Push-to-main notification** (#11): when secrets are found on a direct push, opens a GitHub issue mentioning `@CivicTechWR/organizers` (PR comment step is PR-only)
  - **PR comment dedup** (#14): search for existing gitleaks comment and update it rather than creating a new one per push
  - **Break-glass hardening** (#12): added `--admin` CI bypass warning and merge-failure recovery note
  - **Doc cleanup** (#14): `--config` flag added to local verify commands; CI re-run step added to response guide; README CODEOWNERS wording corrected

  ## Closes

  Closes #11, #12, #13, #14

  ## Test plan

  - [ ] Push a branch with a test credential and confirm PR comment is posted (not a new one on re-push)
  - [ ] Trigger `workflow_dispatch` on the audit job and confirm it runs full history
  - [ ] Confirm scan job skips on `schedule` event (audit-only)
  - [ ] Review break-glass section for accuracy

  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  EOF
  )"
  ```
