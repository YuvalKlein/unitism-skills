# unitism-skills

Shared Claude skills for the UNITISM team and the businesses on it (ARENNA, Tennis Miami, future).

This repo is structured as a **Claude plugin marketplace**. Everyone on the team installs it once and Claude auto-loads the relevant skill whenever the task matches. One person's breakthrough becomes everyone's baseline.

## Install (Asad and Kath — Windows, no terminal)

You install this from inside the **Claude desktop app**, not the terminal. Steps:

1. Open the Claude desktop app.
2. Click the **Code** icon at the top-left (the one between Chat and Cowork). This is Claude Code embedded in the desktop app — same thing as the terminal version, just with a chat UI.
3. In the chat box at the bottom, type this and press enter:

   ```
   /plugin marketplace add YuvalKlein/unitism-skills
   ```

   Claude will fetch the marketplace and confirm. If it asks to trust the source, say yes.

4. Then type:

   ```
   /plugin install unitism@unitism-skills
   ```

   This installs the `unitism` plugin (which contains all three skills: the contribution flow, the handoff receiver, and the positioning check).

5. **Verify it worked.** Type a single `/` in the chat box. You should see autocomplete options including `non-engineer-frontend-contribution`, `cross-repo-handoff-message`, and `product-positioning-check`. If those three names appear, install succeeded. If `/` only shows the default options (like `/help`, `/clear`), the install didn't take — see Troubleshooting below.

You don't need to symlink anything, run any PowerShell commands, or touch a terminal. Steps 3–5 all happen inside the chat box.

### If the repo is private

If `YuvalKlein/unitism-skills` is a private GitHub repo, the desktop app may prompt you to authenticate with GitHub the first time you run step 3. Click through the GitHub auth dialog with your team account. If you don't have access to the repo, ask Yuval to add you as a collaborator on github.com/YuvalKlein/unitism-skills.

## Install (Yuval and Yudi — engineers)

Same as above. The same `/plugin install unitism@unitism-skills` gives you all three skills, including the engineer-side `cross-repo-handoff-message` skill (which only fires when you paste a handoff message from Asad or Kath, so it doesn't add noise the rest of the time).

If you prefer the CLI workflow:

```bash
claude
# inside Claude Code:
/plugin marketplace add YuvalKlein/unitism-skills
/plugin install unitism@unitism-skills
```

## Update

When new skills land in the repo, update with:

```
/plugin marketplace update unitism-skills
```

(Run this from the Code tab in the desktop app, or from Claude Code in the terminal — same command.)

## Verifying it's actually triggering

Installation is one thing; the skill triggering at the right time is another. Quick smoke test for each skill:

- **`non-engineer-frontend-contribution`** — open a Tennis Miami repo (`tennis-miami`, `tennis-miami-web`, or `arenna-link`) in Claude Code, then say *"I want to change the signup button color to navy."* If the skill is active, Claude announces the branch name (`<yourname>/feat/signup-button-color`) before doing anything else. If Claude just edits files without announcing a branch, the skill didn't trigger.
- **`product-positioning-check`** — in the same kind of repo, say *"add a Wallet tab to the bottom navigation."* If the skill is active, Claude pushes back with a positioning verdict referencing the locked principles doc, *before* writing code. If Claude just adds the tab silently, the skill didn't trigger.
- **`cross-repo-handoff-message`** — only fires when you paste a handoff message starting with `## Cross-repo change requested`. Not worth testing standalone; you'll see it the first time someone paastes one in.

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

## Troubleshooting

**`/plugin marketplace add` says it can't find the repo.** Either the repo is private and you're not authenticated (run the command again and look for a GitHub auth dialog), or the repo URL is wrong (should be `YuvalKlein/unitism-skills`, no `https://` prefix needed).

**`/` doesn't show the new skills after install.** Restart the Claude desktop app — close it completely from the system tray, then reopen. Plugin lists are cached on app start.

**The skill installed but doesn't trigger.** Run the smoke test for that skill above. If it doesn't fire, the trigger phrasing in your message may not match what the skill's `description` is looking for — try one of the example phrases from the skill's smoke test. If it still doesn't trigger, ping Yuval; the skill's `description` field may need tightening.

**You're on the desktop app, the Code icon isn't there.** Update the desktop app to the latest version. The Code-mode integration shipped in late 2025; older versions may not have it.

## Adding or editing a skill

For now: open a PR; Yuval reviews. Once we have 5+ active contributors to skills, delegate domain-expert review (Yudi on backend skills, Yuval on strategy skills).

When adding a new skill, place it under `skills/<skill-name>/` with a `SKILL.md` that follows the existing format (name + description in frontmatter, body sections like Purpose / Flow / Anti-patterns / Reference). Don't edit the marketplace.json or plugin.json unless you're adding a *new plugin* (not a new skill within the existing `unitism` plugin) — those files are stable.

## Reference

- [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) — format, trigger behavior, frontmatter fields.
- [Claude Code plugins documentation](https://docs.claude.com/en/docs/claude-code/plugins) — marketplace structure, install/update lifecycle.
- The SKILL.md `description` frontmatter is the single most important line in a skill — it decides when the skill auto-triggers. Spend real time on it.
