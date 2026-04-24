# unitism-skills

Shared Claude Code skill library for UNITISM and the businesses on it (ARENNA, Tennis Miami, future).

Every contributor installs this repo once. From then on, Claude Code auto-loads the relevant skill whenever the task matches. One person's breakthrough becomes everyone's baseline.

## Install

```bash
git clone git@github.com:yuval/unitism-skills.git ~/unitism-skills
```

Then symlink the skills that apply to you.

### For Asad and Kath (UX + Customer Success — frontend contributors)

```bash
ln -s ~/unitism-skills/non-engineer-frontend-contribution ~/.claude/skills/non-eng-frontend
```

That's the only one you need right now. More will be added as we learn where the rails help most.

### For Yuval and Yudi (full-stack)

Symlink the engineer-facing skills individually:

```bash
ln -s ~/unitism-skills/cross-repo-handoff-message ~/.claude/skills/handoff
# as more skills land:
# ln -s ~/unitism-skills/unitism-backend-feature ~/.claude/skills/backend
```

Keep your symlinks explicit — don't symlink the whole repo. That way you only pull in skills that apply to your role, and swapping in new skills later stays tidy.

### Updating

```bash
cd ~/unitism-skills && git pull
```

Restart Claude Code to pick up changes.

## What's here today

- **`non-engineer-frontend-contribution/`** — safe contribution flow for team members without engineering background (Asad, Kath). Branch-off-development naming, keyword-driven commit/push/PR, scope + out-of-scope + secret checks, paste-ready cross-repo handoff message format, onboarding primer with flow diagram.
- **`cross-repo-handoff-message/`** — receiver side for Yuval + Yudi. Triggers when a handoff message is pasted into Claude Code; parses it, creates the branch in the target repo, implements the change, opens the PR, notifies the original requester. **Status: draft, pre-dogfood.** Expect to iterate after week-2 real usage.

## What's coming

Planned but not yet written (on purpose — we write skills after feeling their absence, not before):

- `unitism-backend-feature` — modular monolith patterns, `/api/v1/projects/*` contract, attribute inheritance, COMPONENTS.md registration.
- `arenna-cross-repo-feature` — ARENNA's three-repo workflow (backend + sessionizer + agent-server).
- `strategy-doc` — voice/tone/terminology consistency across pitch, PRDs, investor docs.
- `new-business-on-platform` — launch pattern for business #2 and beyond.

See `_shared/strategy/ai-strategy-handoff.md` in the claude_files folder for the full strategy and sequencing.

## Adding or editing a skill

For now: open a PR; Yuval reviews. Once we have 5+ active contributors to skills, delegate domain-expert review (Yudi on backend skills, Yuval on strategy skills).

## Reference

- [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) — format, trigger behavior, frontmatter fields.
- The SKILL.md `description` frontmatter is the single most important line in a skill — it decides when the skill auto-triggers. Spend real time on it.
