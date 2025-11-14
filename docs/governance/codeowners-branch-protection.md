# Branch Protection and CODEOWNERS Guidance

## Goals

- Give every repository an explicit review path so contributors know who approves changes.
- Provide consistent guardrails (branch protection, required checks) that match CivicTechWR's volunteer-friendly workflow.
- Offer a starting point that new projects can override without losing the safety net.

## Recommended Branch Protection Defaults

Apply these settings to the organization's default branch rule (settings > Code security and analysis > Branch protections). Individual repos can tighten the requirements, but they should not weaken them without organizer sign-off.

1. **Target branches:** `main` (and `master` for legacy repositories).
2. **Require a pull request before merging.**
3. **Required reviewers:**
   - At least **1 approval**, prefer **2 approvals** for active codebases.
   - **Require review from Code Owners** (once the default CODEOWNERS file lands).
4. **Status checks:** enable the projects primary CI build (e.g., `lint`, `test`, `deploy-preview`). Start with the checks that already exist; new projects should add them before their first release.
5. **Additional safeguards:**
   - Require conversation resolution before merging.
   - Require linear history.
   - Allow force pushes only for administrators handling emergency fixes.
   - (Optional) Require signed commits if the project already enforces them.
6. **Administrator enforcement:** keep the rule enforced for admins, with a documented emergency break-glass process.

Review the branch rule quarterly to confirm the status check list matches reality and that teams still have capacity for review.

## Organization Teams Snapshot

| Team | Status | Scope / Suggested Repositories |
| --- | --- | --- |
| `@CivicTechWR/organizers` | Active | Default reviewers for repos without a dedicated team; governance repos like `.github`, `CTWR-Organization-Documentation`, `CTWR-Docs`. |
| `@CivicTechWR/website` | Active | `ctwr-web`, `CTWR-Website`, `blog`. Add `People-Library-Project` if communications wants shared ownership. |
| `@CivicTechWR/wrvotes` & `@CivicTechWR/wrvotesiteadmin` | Active | Election assets: `WRvotes`, `WRVotesPlaceholder`, `WRVotesMunicipal2022`, `WRVotesProv2025`, `WRVotesFed2025`, `WRVotesFed2025old`, `WRVotesFed`. |
| `@CivicTechWR/go-train-pass-project-team` | Active | `go-train-group-pass`. |
| `@CivicTechWR/connected-kw` | Active | `connectedkw`. |
| `@CivicTechWR/midtown-radio-app` | Active | `MidtownRadioApp`, `midtown-radio-launch-2025`. |
| `@CivicTechWR/project-ploughshares-dev-team` & `@CivicTechWR/project-ploughshares-leads` | Active | `project-ploughshares`. Dev team handles code; leads handle partner coordination. |
| `@CivicTechWR/zonechanges` | Active | `WaterlooZoneChangeRequests`. |
| `@CivicTechWR/ctwr-member-directory` | Active | `ctwr-member-directory`. |
| `@CivicTechWR/project-pech` | Proposed | `project-pech`. Seed with `j2fyi` and an organizer for continuity. |
| `@CivicTechWR/mappingwr` | Proposed | `mappingwr`, `municipal-engagement`. Pair `pnijjar` with supporting reviewers. |
| `@CivicTechWR/storytelling` | Proposed | `storytelling`. Invite `ToddTurnbull` plus a communications lead. |
| `@CivicTechWR/project-templates` | Proposed | `CTWR-ProjectTemplate`, `CTWR-Project-Template-New`, `CTWR-Template`, `CTWR-Docs`. Blend template maintainers (`BreakableHoodie`, `KristinaTaylor`, `livialima`) with organizers. |
| `@CivicTechWR/improved-housing` | Proposed | `improved_housing_portal`. Ensure `keriwarr` and an organizer are maintainers. |

Projects stay under `@CivicTechWR/organizers` until their project team exists **and** a repo-level CODEOWNERS file assigns the new team. Organizers should create teams when projects become active and revisit this table quarterly.

## Default CODEOWNERS Strategy

Create `.github/CODEOWNERS` (this repository) so GitHub applies it to every CivicTechWR repo that does not define its own CODEOWNERS file.

```text
# Default owners for every file in repos without project-specific CODEOWNERS
* @CivicTechWR/organizers

# Documentation stays under organizer review until project teams override it
*.md @CivicTechWR/organizers
*.yml @CivicTechWR/organizers
*.yaml @CivicTechWR/organizers
*.json @CivicTechWR/organizers
.github/** @CivicTechWR/organizers
```

Key considerations:

- Patterns in this default file must be generic. Repo-specific overrides belong in that repo.
- Github evaluates CODEOWNERS top to bottom; place broader matches last.
- Encourage projects to commit their own CODEOWNERS file as soon as they have a stable team. Provide them with a template (see below) and remind them to keep `@CivicTechWR/organizers` as a secondary owner for continuity.

### Sample project CODEOWNERS template

```text
# Primary reviewers
* @CivicTechWR/project-team

# Keep organizers looped in on governance docs
/docs/** @CivicTechWR/organizers
/.github/workflows/** @CivicTechWR/organizers @CivicTechWR/project-team
```

## Implementation Checklist

1. **Confirm team access:** Ensure each team above has triage or write permissions on its corresponding repositories.
2. **Add default CODEOWNERS:** Commit the file described above to this repo and roll it out.
3. **Update branch protection:** Apply the recommended rule via the organization's default branch protection settings.
4. **Communicate the change:** Post in `#organizers`, update `CTWR-Organization-Documentation`, and mention during the weekly meetup.
5. **Repository follow-up:** Track which repos still rely solely on organizers and open issues inviting teams to add their own CODEOWNERS entries.
6. **Before adding reviewers:** Confirm the repo does **not** already contain a CODEOWNERS file to avoid accidental overrides or duplicate entries.

## Maintenance Cadence

- **Quarterly:** Review team membership, permissions, and branch protection rules.
- **Season kickoff:** For each project sprint (12-week cycle), confirm the CODEOWNERS file lists current maintainers.
- **After project sunset:** Archive the repo or revert CODEOWNERS to organizers-only ownership to avoid stale reviewers.

Maintaining these guardrails keeps contributor expectations clear while still letting squads iterate quickly once their teams are established.

## Risks and Pitfalls

- **Stale team membership:** CODEOWNERS blocks merges when listed reviewers are inactive. Reconfirm roster changes every quarter and any time a maintainer steps back.
- **Existing CODEOWNERS files:** Repository-level files override the defaults. Always check for a local CODEOWNERS before assuming the organizers are the only reviewers.
- **Automation permissions:** Workflows that create teams or modify permissions require an organization-level Personal Access Token with `admin:org`; keep the token scoped and rotate it periodically.
- **Template repositories:** Some templates are public yet seldom updated. If a project template lacks dedicated maintainers, leave organizers as owners to prevent PRs from stalling.
- **Archived or legacy repos:** Election archives and similar repos may still receive security updates. Confirm whether they need active CODEOWNERS or should be archived to remove review pressure.
