# unitism-skills

Shared Claude skills for the UNITISM team and the businesses on it (ARENNA, Tennis Miami, future).

This repo is structured as a **Claude plugin marketplace**. The team admin syncs it once into the Claude organization with "Installed by default" access; everyone in the org gets the plugin automatically on their next app launch. Claude then auto-loads the relevant skill whenever the task matches. One person's breakthrough becomes everyone's baseline.

## Install — admin step (Yuval, one-time)

1. Open **claude.ai** in a browser and log in as the org owner.
2. Go to **Organization settings** → **Libraries** → **Plugins**.
3. Click **Add plugins** → **Sync from GitHub**.
4. In the dialog: pick `YuvalKlein/unitism-skills` from the dropdown. Leave **Marketplace name** as default (or set to something cleaner like `Unitism`). Keep **Sync automatically** ON. Set **Default access** to **Installed by default** — this is what gives every org member the plugin without them needing to click anything.
5. Click **Create**.

Claude will sync the marketplace from the repo. Within a few seconds the entry should show **Synced** with the **Unitism** plugin listed under it. From this moment on, every push to `main` propagates to every org member's desktop app on next restart.

**Prerequisite:** Asad and Kath must be members of your Claude organization (Identity and access tab in the same admin sidebar). If they're on individual Claude accounts, the plugin won't reach them — invite them to the org first.

## Install — member step (Asad, Kath, Yudi)

There is no install step. With **Installed by default** set on the admin side, the plugin appears automatically.

1. **Restart the Claude desktop app** (close it completely from the system tray, then reopen). Plugin lists are cached at startup, so a fresh launch is needed to see new plugins.
2. Type a single `/` in any chat. You should see autocomplete options including:
   - `non-engineer-frontend-contribution`
   - `cross-repo-handoff-message`
   - `product-positioning-check`

If those three names appear, the plugin is active and the skills are ready to fire on the right triggers.

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

## Troubleshooting

**`/` doesn't show the new skills.** Restart the Claude desktop app — close it completely from the system tray, then reopen. Plugin lists are cached on app start.

**The skill installed but doesn't trigger.** Run the smoke test for that skill above. If it doesn't fire, the trigger phrasing in your message may not match what the skill's `description` is looking for — try one of the example phrases from the smoke test. If it still doesn't trigger, ping Yuval; the skill's `description` field may need tightening.

**You're a new team member and don't see the plugin.** Confirm with Yuval that you've been invited to his Claude organization (claude.ai → Organization → Identity and access). Without org membership, "Installed by default" doesn't reach you.

**You're on the desktop app, the Customize sidebar item isn't there.** Update the desktop app to the latest version. Plugin/customize support shipped in early 2026; older versions may not have it.

## Adding or editing a skill

For now: open a PR; Yuval reviews. Once we have 5+ active contributors to skills, delegate domain-expert review (Yudi on backend skills, Yuval on strategy skills).

When adding a new skill, place it under `skills/<skill-name>/` with a `SKILL.md` that follows the existing format (name + description in frontmatter, body sections like Purpose / Flow / Anti-patterns / Reference). Don't edit `marketplace.json` or `plugin.json` unless you're adding a *new plugin* (not a new skill within the existing `unitism` plugin) — those files are stable. After merge to `main`, the org-synced plugin picks up the change automatically.

## Repo structure

```
unitism-skills/
├── .claude-plugin/
│   ├── marketplace.json    # registers this repo as a marketplace, points to the unitism plugin
│   └── plugin.json         # describes the unitism plugin (name, version, description)
├── skills/
│   ├── non-engineer-frontend-contribution/
│   │   ├── SKILL.md
│   │   ├── reference/
│   │   └── templates/
│   ├── cross-repo-handoff-message/
│   │   └── SKILL.md
│   └── product-positioning-check/
│       └── SKILL.md
├── README.md
└── .gitignore
```

The plugin (`.claude-plugin/plugin.json`) and the marketplace (`.claude-plugin/marketplace.json`) coexist at the repo root by design. The marketplace.json uses `source: { source: "github", repo: "YuvalKlein/unitism-skills" }` — the same-repo reference works because the validator clones the repo and finds the plugin metadata at the root.

## Reference

- [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) — format, trigger behavior, frontmatter fields.
- [Claude Code plugins documentation](https://docs.claude.com/en/docs/claude-code/plugins) — marketplace structure, install/update lifecycle.
- The SKILL.md `description` frontmatter is the single most important line in a skill — it decides when the skill auto-triggers. Spend real time on it.
