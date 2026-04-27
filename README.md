# unitism-skills

Shared Claude skills for the UNITISM team and the businesses on it (ARENNA, Tennis Miami, future).

This repo is structured as a **Claude plugin marketplace**. The team admin syncs it once into the Claude organization; everyone on the team installs it once from inside the desktop app. Claude then auto-loads the relevant skill whenever the task matches. One person's breakthrough becomes everyone's baseline.

## Install — admin step (Yuval, one-time)

1. Open **claude.ai** in a browser and log in as the org owner.
2. Go to **Organization settings** → **Libraries** → **Plugins**.
3. Click **Add plugins** (top-right) → **Sync from GitHub**.
4. Enter `YuvalKlein/unitism-skills`. Authenticate with GitHub if prompted.
5. Set **User access** to whatever's right (open to all org members, restricted to specific people, etc.).

Claude will auto-detect the marketplace structure (`.claude-plugin/marketplace.json` at the repo root) and surface the `unitism` plugin to your org. Whenever the repo updates, the synced version follows automatically — no manual re-sync, no zip uploads.

**Prerequisite:** Asad and Kath must be members of your Claude organization (Identity and access tab in the same admin sidebar). If they're on individual Claude accounts, the plugin won't reach them — invite them to the org first.

## Install — member step (Asad, Kath, Yudi)

Once the admin step is done, every team member installs the plugin once from inside the **Claude desktop app**:

1. Open the Claude desktop app.
2. Click **Customize** in the left sidebar (briefcase icon).
3. Click **Browse plugins**.
4. Click the **Your organization** tab at the top.
5. Find **unitism** in the list, click the **`+`** button to install.

That's it. No terminal, no symlinks, no PowerShell, no GitHub auth dialog (the admin already handled the GitHub side).

### Verify it worked

Type a single `/` in any chat (Chat, Cowork, or Code tab — anywhere). You should see autocomplete options including:

- `non-engineer-frontend-contribution`
- `cross-repo-handoff-message`
- `product-positioning-check`

If those three names appear, the install succeeded. If only default options like `/help` show up, restart the desktop app (close from system tray, reopen) — plugin lists are cached at app start.

## Verifying the skills actually trigger

Installation is one thing; the skill triggering at the right moment is another. Quick smoke tests:

- **`non-engineer-frontend-contribution`** — open a Tennis Miami repo (`tennis-miami`, `tennis-miami-web`, or `arenna-link`) in Claude, then say *"I want to change the signup button color to navy."* If the skill is active, Claude announces the branch name (`<yourname>/feat/signup-button-color`) before doing anything else. If Claude just edits files without announcing a branch, the skill didn't trigger.
- **`product-positioning-check`** — in the same kind of repo, say *"add a Wallet tab to the bottom navigation."* If the skill is active, Claude pushes back with a positioning verdict referencing the locked principles doc, *before* writing code. If Claude just adds the tab silently, the skill didn't trigger.
- **`cross-repo-handoff-message`** — only fires when you paste a handoff message starting with `## Cross-repo change requested`. Not worth testing standalone; you'll see it the first time someone pastes one in.

## What's in the bundle today

- **`skills/non-engineer-frontend-contribution/`** — safe contribution flow for team members without engineering background. Branch-off-development naming, keyword-driven commit/push/PR (`push`, `done`, `revert`), scope + out-of-scope + secret checks, paste-ready cross-repo handoff message format, onboarding primer with flow diagram.
- **`skills/cross-repo-handoff-message/`** — receiver side for Yuval and Yudi. Triggers when a handoff message is pasted in; parses it, creates the branch in the target repo, implements the change, opens the PR, notifies the original requester. **Status: draft, pre-dogfood.** Expect to iterate after week-2 real usage.
- **`skills/product-positioning-check/`** — second pair of eyes on changes that touch a primary surface (bottom nav, onboarding, hero copy, app store listing, first push notification). Reads `_shared/strategy/product-positioning-principles.md`, identifies which product's positioning applies (Tennis Miami → community-first; ARENNA → instructor-first), runs the decision test, and emits PASS / FLAG / FAIL with a concrete non-primary alternative. Does not block — informs. Triggered by the April 2026 wallet-and-offers-in-nav incident.

## What's coming

Planned but not yet written (we write skills after feeling their absence, not before):

- `unitism-backend-feature` — modular monolith patterns, `/api/v1/projects/*` contract, attribute inheritance, COMPONENTS.md registration.
- `arenna-cross-repo-feature` — ARENNA's three-repo workflow (backend + sessionizer + agent-server).
- `strategy-doc` — voice/tone/terminology consistency across pitch, PRDs, investor docs.
- `new-business-on-platform` — launch pattern for business #2 and beyond.

See `_shared/strategy/ai-strategy-handoff.md` in `claude_files` for the full strategy and sequencing.

## Updates

When new skills land in this repo and get pushed to GitHub, the org-synced plugin updates automatically — no admin action needed. Members may need to restart the desktop app to pick up new skills, since the skill list is cached at startup.

If a critical update needs to land before someone's next restart, ask them to: **Customize → Skills → click the unitism plugin → Refresh**. (Skill names should re-appear in `/` autocomplete immediately after.)

## Troubleshooting

**The "Your organization" tab shows nothing.** Either you're not a member of the Claude organization that owns the synced plugin, or the admin hasn't synced it yet. Confirm with Yuval that you've been invited to the org and that the sync is complete on the admin side.

**`/` doesn't show the new skills after install.** Restart the Claude desktop app — close it completely from the system tray, then reopen. Plugin lists are cached on app start.

**The skill installed but doesn't trigger.** Run the smoke test for that skill above. If it doesn't fire, the trigger phrasing in your message may not match what the skill's `description` is looking for — try one of the example phrases from the smoke test. If it still doesn't trigger, ping Yuval; the skill's `description` field may need tightening.

**You're on the desktop app, the Customize tab isn't there.** Update the desktop app to the latest version. Plugin/customize support shipped in early 2026; older versions may not have it.

## Adding or editing a skill

For now: open a PR; Yuval reviews. Once we have 5+ active contributors to skills, delegate domain-expert review (Yudi on backend skills, Yuval on strategy skills).

When adding a new skill, place it under `skills/<skill-name>/` with a `SKILL.md` that follows the existing format (name + description in frontmatter, body sections like Purpose / Flow / Anti-patterns / Reference). Don't edit `marketplace.json` or `plugin.json` unless you're adding a *new plugin* (not a new skill within the existing `unitism` plugin) — those files are stable. After merge to `main`, the org-synced plugin picks up the change automatically.

## Reference

- [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) — format, trigger behavior, frontmatter fields.
- [Claude Code plugins documentation](https://docs.claude.com/en/docs/claude-code/plugins) — marketplace structure, install/update lifecycle.
- The SKILL.md `description` frontmatter is the single most important line in a skill — it decides when the skill auto-triggers. Spend real time on it.
