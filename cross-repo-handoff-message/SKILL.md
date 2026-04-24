---
name: cross-repo-handoff-message
description: >
  Use this skill when the user pastes a cross-repo handoff message
  originating from the `non-engineer-frontend-contribution` skill.
  Triggers specifically on messages containing the header
  "## Cross-repo change requested" or the block "## Paste-ready brief
  for Claude Code", or on pastes that start with "I need to add a
  backend change to support a frontend PR that's waiting." Also
  triggers when the user pastes any message that names a frontend PR
  URL from `tennis-miami-web`, `arenna-link`, or `tennis-miami` and
  asks for a matching backend change. This skill parses the message,
  identifies the target repo, creates a branch from `development`,
  implements the specified change, opens a PR back to `development`
  linked to the originating frontend PR, and — on merge — generates a
  notification message for the original requester (Asad or Kath).
  Installed on Yuval's and Yudi's Claude Code only. Do NOT trigger on
  general feature requests unconnected to a non-engineer handoff, on
  discussions about the handoff flow itself, or on frontend-only
  changes.
---

# Cross-repo handoff message — receiver side

## Purpose

The non-engineer contribution flow produces paste-ready handoff messages whenever Asad or Kath hit a scope boundary (their PR needs a backend change to work). This skill is the other half: it's what runs on Yuval's or Yudi's Claude Code when they paste that message in.

Goal: zero re-explanation. Paste the message, the work happens, the frontend PR gets unblocked.

> **Status: draft.** Written alongside skill #1 before week-2 dogfood data exists. Expect to edit this based on the first 2–3 real handoffs. If a handoff doesn't parse cleanly, the first instinct is to improve the producer (skill #1's template) rather than to add parsing cleverness here.

## When this fires

The user — Yuval or Yudi — pastes a message into Claude Code (desktop or mobile). The message follows the structure defined in `non-engineer-frontend-contribution/templates/cross-repo-handoff-message.md`.

Key markers that the skill should look for:

- Section header `## Cross-repo change requested` OR
- Section header `## Paste-ready brief for Claude Code` OR
- Opening line `I need to add a backend change to support a frontend PR that's waiting.` OR
- A message that names a frontend PR URL from `tennis-miami-web`, `arenna-link`, or `tennis-miami` and requests a corresponding backend change

If none of these match but the request is clearly a handoff, the receiver may not match — ask the user if they want to run it as a handoff anyway, or as a free-form request.

## Parse: what to extract

From the pasted message, extract:

| Field | Where it comes from |
|-------|----------------------|
| **Requester** | "From: <username>" line |
| **Requester's role** | Context or known team mapping (Asad → UX, Kath → CS) |
| **Frontend repo** | "working in `<repo>`" |
| **Frontend PR URL** | Branch line or PR URL in the message |
| **Target repo** | "Target repo: `<repo>`" |
| **Change needed** | "What needs to change:" bullets |
| **Must-not-break list** | "What should stay the same / not break:" bullets |
| **Tests to pass** | "Tests to pass:" or explicit test names |
| **Suggested PR title** | Optional — use "When done: Open a PR with title" line if present |

If any mandatory field is missing (target repo, change needed, or frontend PR URL) — STOP, tell the user what's missing, don't guess.

## Run

### 1. Verify target repo

Before creating a branch:

- `cd` into the target repo (user may already be in it; if not, note the path)
- Confirm `development` branch exists. If not, STOP and tell the user — this is a repo-level setup issue, not a per-PR issue
- Confirm working tree is clean or offer to stash existing work first

### 2. Create the branch

```bash
git fetch origin
git checkout -b {owner}/{type}/{slug} origin/development
```

Where:
- `{owner}` — lowercase first name of whoever is running this skill (from `git config user.name`). Usually `yuval` or `yudi`.
- `{type}` — inferred from the change: new endpoint → `feat`, bug fix to existing behavior → `fix`, migration → `chore`, schema change without behavior change → `chore`.
- `{slug}` — derived from the suggested PR title if present, otherwise from the change description. Max 4 hyphenated words.

Announce: *"Starting handoff work on `{branch-name}` in `{target-repo}`, branched from latest `development`. Frontend PR waiting: {frontend_pr_url}."*

### 3. Implement

Follow the instructions exactly as specified in the message. Do not creatively extend scope — the frontend PR is already built against a specific interface, so shipping more than what was asked breaks the coupling.

If the instructions are ambiguous on a specific point:
- If the ambiguity is small (naming, style), pick the convention used elsewhere in the repo and note the choice in the PR body
- If the ambiguity is material (security model, data shape, backwards compatibility), STOP and tell the user what needs clarification — don't guess on material questions

### 4. Verify

- Run the test suite named in the message (or `npm test` / `yarn test` / `pytest` / whatever the repo uses by default)
- If tests fail, do not open the PR — investigate, fix, or escalate
- If the change touches a migration, dry-run the migration locally before pushing

### 5. Open the PR

```bash
gh pr create \
  --base development \
  --title "<suggested title or inferred from change>" \
  --body "$(...)"
```

PR body should include:

```
## Context
Unblocks {frontend_pr_url} from {requester}.

## What changed
<description of the backend change>

## Link back
- Frontend PR: {frontend_pr_url}
- Requester: @{requester}

## Verification
- [ ] Tests passing: <list>
- [ ] Manual check: <specific manual verification if applicable>
- [ ] Must-not-break list reviewed

## Review
Tag @yuval and @yudi — first responder wins.

---
Generated by `cross-repo-handoff-message` skill on behalf of {requester}.
```

### 6. Notify the requester

Emit a one-liner the user can paste into the team chat to notify the original requester:

> @{requester} — backend change for your PR {frontend_pr_url} is open at {backend_pr_url}. Once it merges, run `done` again in your frontend branch.

If you know the team has a shared Slack / Asana / ClickUp channel for this, name it; otherwise default to "send this to {requester} directly."

### 7. After merge (best-effort)

If the user returns later after the backend PR is merged and says something like "done" / "merged" / "notify asad" — emit the notification message:

> @{requester}, the backend change your PR depended on is live on `development`. Go back to your frontend branch and say `done` and your PR will open.

Ideally this step is automated by a CI hook or a GitHub Action, but for now: manual, triggered by the engineer.

## Anti-patterns — never do these

- ❌ Creatively expand the scope beyond what the handoff asked for (the frontend PR assumes a specific interface; breaking it is worse than not shipping)
- ❌ Merge the backend PR yourself without a review — first-responder wins, still applies
- ❌ Start the work on `master` or any branch other than `development`-derived
- ❌ Skip the test suite because the change looks small
- ❌ Use a branch name that doesn't follow `{owner}/{type}/{slug}` — consistency makes the audit trail readable
- ❌ Rewrite the requester's message to sound better before storing it in the PR body — keep the original voice; it's part of the trail

## When the paste doesn't parse

If the pasted message doesn't match the expected template structure:

- It might be an old format, a manual handoff, or a message from outside the skill system — don't force-fit it
- Ask the user: *"This doesn't look like a standard handoff message. Do you want me to run it as a free-form backend change, or is the message incomplete?"*
- Never silently infer a frontend PR URL or requester — those have to be explicit

## Reference

- Producer side: `../non-engineer-frontend-contribution/SKILL.md`
- Template format: `../non-engineer-frontend-contribution/templates/cross-repo-handoff-message.md`
