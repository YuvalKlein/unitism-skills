---
name: non-engineer-frontend-contribution
description: >
  Use this skill whenever the user is contributing to the frontend of
  `arenna-link`, `tennis-miami`, or `tennis-miami-web`. Triggers on any
  UI change (buttons, forms, layout, colors, spacing, icons), copy or
  text changes, design tweaks, or bug fixes visible to users. Also
  triggers on requests phrased as "I want to change...", "fix the...",
  "update the...", "the landing page needs...", or similar, when the
  session is in one of those three repos or mentions them by name.
  This skill keeps the person on a safe branch off `development`,
  auto-writes commits, and runs scope / secret / out-of-scope checks
  before opening a PR. It also generates paste-ready handoff messages
  when a change requires a different repo (backend, agent-server).
  Do NOT trigger on backend repos (`unitism-backend`, `arenna-backend`,
  `arenna-agent-server`), on Flutter/Dart business-logic changes that
  are not UI-visible, on CI / deploy / infrastructure config, or on
  documentation-only changes. Intended for team members whose primary
  role is not engineering (e.g., UX design, customer success) —
  engineers working in these repos can say "I'll handle this manually,
  skip the rails" to opt out mid-session.
---

# Non-engineer frontend contribution

## Purpose

The UNITISM team has contributors whose primary role is not engineering (Asad — UX Design, Kath — Customer Success). They now ship frontend code directly via Claude rather than waiting on the CTO. This skill runs the full git flow underneath a few plain-language keywords, so they don't need to think about branches, commits, PRs, or conflict resolution. It also catches the cases where a change really does need an engineer — and hands off cleanly instead of letting it get merged by accident.

## Who this applies to

Team members whose Claude Code install includes this skill. By default:

- Asad, Kath → **yes**
- Yuval (CEO, full-stack), Yudi (CTO, full-stack) → **no** (they don't install it)

If one of the engineers has it installed because they're in a shared session, they can opt out by saying "skip the rails — I'll handle this manually" and this skill steps aside for the rest of the session.

## The three keywords

The whole flow runs on three words the person types in chat:

| Keyword | What it does |
|---------|--------------|
| (describe the change) | Claude creates a branch off `development` and starts working |
| `push` | Claude commits the current changes (auto-written message) and pushes the branch to GitHub. Preview URL auto-deploys. Safe to use liberally — the branch is private. |
| `done` or `PR` | Run all checks, rebase, open a PR to `development`, post the preview URL |
| `revert` | Undo a merge the person themself shipped (own branches only) |

**Mental model:** a user's personal branch (e.g., `asad/feat/new-button`) is their private workspace. Pushing to it is safe — it only deploys to a private preview URL, never to a shared environment. `development` and `master` are the deploy-to-shared branches, which are never pushed to directly — only merged into via PR. The net effect: iterate freely on your branch with `push`, ask for review with `done`.

## One-time per-repo setup (runs automatically if missing)

Before doing any work, verify:

1. The repo has a `development` branch. If missing, STOP and emit the cross-repo handoff message (`templates/cross-repo-handoff-message.md`) addressed to Yuval or Yudi saying "please create a `development` branch in this repo."
2. `git config user.name` returns a value. Use the lowercase first name as the branch prefix (e.g., "Asad Rahman" → `asad`). If blank, ask once and persist.
3. Working tree is clean. If not, offer to `push` the pending work onto the existing branch first.

## Flow, step by step

### 1. Person describes the change

Example: *"I want to change the signup button color to navy."*

Ask at most **one** clarifying question, only if the intent is genuinely ambiguous. Good question: *"Which signup button — the one on the landing page, or the one inside the app?"* Bad question: *"What exact hex code?"* — non-engineers find over-questioning discouraging.

### 2. Create a branch

```bash
git fetch origin
git checkout -b {username}/{type}/{slug} origin/development
```

Branch naming:

- `{username}` — lowercase first name from `git config user.name`
- `{type}` — one of `feat`, `fix`, `copy`, `style`, `chore`. Infer from intent: *"change the X"* → `feat`; *"fix the Y"* → `fix`; *"reword"* / *"rewrite text"* → `copy`; *"adjust spacing"* → `style`.
- `{slug}` — hyphenated short summary, max 4 words. e.g., `signup-button-color`, `homepage-copy-v2`, `fix-dropdown-overflow`.

Announce what happened in one line: *"Working on `asad/feat/signup-button-color`, branched from latest `development`."*

### 3. Do the work

Make the change. If there's a cheap local-verify step (dev server, storybook, visual check), do it. **Do not refactor code that isn't part of the stated task** — that's a direct cause of scope creep and the reason step 5b exists.

### 4. `push` — commit and publish

One keyword, two atomic git actions. Claude does both.

**(a)** Auto-write a conventional-commit message from the diff. Format:

```
<type>(<optional scope>): <imperative, sentence-case, ≤72 chars>
```

Examples:
- `feat(signup): change button color to navy blue`
- `fix(form): correct email validation on signup page`
- `copy(landing): rewrite hero subheadline`

Commit with that message.

**(b)** Push the branch:

```bash
git push -u origin <branch>
```

Confirm: *"Pushed to GitHub. Preview URL will appear in ~1 minute at <url>."* Vercel/Firebase Hosting auto-deploy a preview per branch — see `reference/onboarding-for-non-engineers.md` for address patterns.

There is no local-only "save" concept. The branch is the safe space; pushing to it is just syncing and generating a fresh preview. Iterate freely — `push` as many times as needed.

### 5. `done` or `PR` — prepare and open the PR

Run these checks **in order**. Do not skip any. If any fails, STOP and follow the failure path — do not open the PR.

#### 5a. Rebase onto latest `development`

```bash
git fetch origin
git rebase origin/development
```

If there are merge conflicts: STOP. Do not attempt to resolve them — a wrong auto-resolve is worse than a stuck PR. Emit the cross-repo handoff message addressed to Yuval or Yudi, with the failing file paths and the recent `development` commits that caused the conflict. Wait for the engineer to resolve before resuming.

#### 5b. Scope-creep check

Look at the diff. Does the set of changed files still match what the branch name said it was about?

Triggers a nudge if:
- Slug is about one thing (e.g., `signup-button-color`) but diff touches 8+ unrelated components
- Diff crosses 3+ feature directories without a clear reason
- Commit history on the branch has changes with different apparent intents (e.g., a copy change *and* a new feature *and* a bug fix)

Response if triggered: *"This branch started as `{slug}` but the changes now span `{list of areas}`. Want to split this into separate branches, or rename this one to something broader like `{suggested-new-slug}`? Narrow PRs are much faster to get reviewed."* Wait for direction — don't force a split, just make it visible.

#### 5c. Out-of-scope check (most important)

Scan the diff for signals that this change requires a different repo to also change for it to actually work. Specifically look for:

- New API calls to endpoints that don't currently exist in the API client / types
- Request bodies with fields the backend TypeScript / Dart types don't include
- References to response fields the types don't include
- Env vars used but not declared in `.env.example` or config schemas
- New external services / SDKs added as dependencies
- New data shapes that assume backend support that isn't there

If **any** signal fires: STOP. Do not open the PR. Fill in `templates/cross-repo-handoff-message.md` with the specifics and show it to the user. Tell them:

> This change needs a backend change first. Here's a message to send to Yuval or Yudi — copy it as-is into your chat with them. When they say it's done, come back here and say `done` again and I'll continue.

#### 5d. Secret scan

Grep the diff for common leak patterns:

- API key prefixes: `sk-`, `pk_live_`, `pk_test_`, `AIza`, `xoxb-`, `ghp_`, `github_pat_`
- `.env` files added to the commit
- Plausible base64/hex blobs `[A-Za-z0-9]{32,}` inside files that aren't lockfiles, snapshots, or build outputs
- Literal assignments: `password=`, `secret=`, `token=`, `api_key=` followed by non-placeholder values

If any match: STOP. Do not push. Offer to amend the commit to remove the secret and rotate it via Yuval or Yudi. Say clearly what was found and where.

#### 5e. Open the PR

```bash
gh pr create \
  --base development \
  --title "<slug rewritten as sentence>" \
  --body "<auto-written body>"
```

PR body template:

```
## What changed
<one paragraph — what the user asked for, what ended up in the diff>

## Preview
<preview URL from Vercel or Firebase preview channel>

## Visual check
- [ ] Opened the preview URL
- [ ] Looks right on desktop
- [ ] Looks right on mobile

## Review
First responder wins (Yuval or Yudi).

---
Requested by {username} via `non-engineer-frontend-contribution` skill.
```

#### 5f. Notify

Print the PR URL. Emit a one-liner the person can paste into the team chat:

> `{username}` opened a PR in `{repo}`: `{title}` — {PR URL}

### 6. `revert` — undo a merge

Only the branch author can revert their own branch's merge. If the user asks to revert something someone else merged, decline and emit a handoff message to Yuval or Yudi.

For the author's own merges:

```bash
git checkout development
git pull origin development
git revert -m 1 <merge-commit-sha>
git push origin development
```

Then notify: *"Reverted. `development` is back to how it was before your PR. Anything you wanted to keep from it is still on your original branch — tell me and we'll try again."*

## Anti-patterns — never do these

- ❌ Start work directly on `master` or `development` (always branch off)
- ❌ Push directly to `development` or `master` (only PRs reach those branches)
- ❌ Open a PR without running rebase + scope + out-of-scope + secret checks
- ❌ Auto-resolve merge conflicts for a non-engineer
- ❌ Merge to `development` yourself — always PR, always human review
- ❌ Touch a second repo in the same flow — out-of-scope check must catch it
- ❌ Skip secret scan because "it's probably fine"
- ❌ Over-question the person with three clarifying questions before starting
- ❌ Refactor surrounding code that wasn't part of the task
- ❌ Use any word other than "units" or "revenue shares" when committing strategy-adjacent copy (see `_shared/strategy/ai-strategy-handoff.md` — terminology rule)

## When a request doesn't fit this skill

If the user asks for something this skill isn't built for — a config change, a CI tweak, a deploy script, a new dependency, a backend change — STOP and emit the cross-repo handoff message addressed to Yuval or Yudi. Non-engineers should not touch CI, config, infrastructure, or backend directly; the handoff flow is how those changes get made safely.

## Reference

- `templates/cross-repo-handoff-message.md` — the paste-ready format emitted on out-of-scope, conflict, or infra-adjacent requests.
- `reference/onboarding-for-non-engineers.md` — the plain-language primer Asad and Kath read before their first use, including the flow diagram.
