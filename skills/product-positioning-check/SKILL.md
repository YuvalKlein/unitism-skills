---
name: product-positioning-check
description: >
  Use this skill whenever a change is being made to a primary surface
  in a UNITISM consumer-facing repo (`tennis-miami`, `tennis-miami-web`,
  `arenna-link`, or future consumer apps on the platform). Primary
  surfaces include: bottom navigation / tab bar, the first screen
  shown after login or signup, the onboarding or signup flow's
  headline / primary CTA, the app store listing copy, the marketing
  site hero or top-level navigation, the home tab's empty state, and
  the first push notification copy sent to a new user. Triggers on
  diffs that add, remove, rename, reorder, or re-label items on these
  surfaces; on copy changes to hero headlines, CTAs, or onboarding
  text; on icon or label changes for nav tabs; and on any request
  phrased as "add X tab", "show Y on home", "rename the Z tab",
  "change the hero", or similar primary-surface language. Also
  triggers when a backend or features-only change introduces a new
  user-facing surface that will need primary-surface placement
  (e.g., a new entitlement that implies a new tab). The skill checks
  the change against `_shared/strategy/product-positioning-principles.md`,
  identifies which product's positioning applies (Tennis Miami →
  community-first; ARENNA → instructor-first), runs the
  primary-surface decision test, and outputs a flag-with-reasoning
  + a suggested non-primary alternative when the change risks shifting
  the product's perceived category. Do NOT trigger on backend-only
  changes with no user-visible surface, on internal tooling, on test
  files, on lockfiles, on changes confined to non-primary surfaces
  (account/settings/profile sub-views), or on changes to internal
  product or strategy docs. Engineers can opt out for a single change
  by saying "skip the positioning check — I've thought about it."
---

# Product positioning check

## Purpose

A non-engineer recently shipped a change that didn't break the build but did break the product's positioning: wallet and offers were added as bottom-nav tabs in Tennis Miami, shifting the app's first-glance answer from *"community"* to *"marketplace."* The build passed. CI passed. The PR review caught nothing because reviewers checked the code, not the positioning.

This skill exists to catch that class of regression. It runs alongside (not instead of) the normal review skills. It does one job: when a primary surface is being touched, evaluate the change against the product's stated positioning principle and either bless it, suggest a non-primary alternative, or flag it for explicit Yuval sign-off.

It is not a gate. It is a second pair of eyes that knows what the product is supposed to be.

## Source of truth

This skill reads `_shared/strategy/product-positioning-principles.md` (in `claude_files`, sibling to the strategy handoff). That doc names:

- The core principle (primary surfaces reflect the cold-start side)
- What counts as a primary surface
- Per-product positioning (Tennis Miami → community-first; ARENNA → instructor-first)
- The decision test
- Worked examples and anti-patterns

If that doc is missing or out of date, **STOP** and tell the user before proceeding — running the check without a current source of truth produces false confidence. The doc must exist and must have a `Status: locked` line; if it says "draft" or has no status, treat it as advisory and surface that caveat in the output.

## When this fires

The skill activates when a diff touches a primary surface in a consumer-facing repo. Concretely:

**Files / patterns that always trigger** (Tennis Miami / Flutter naming, adapt for the actual repo):

- Anything named `bottom_nav`, `tab_bar`, `main_navigation`, `app_shell`, `root_navigator`, `home_scaffold`
- `onboarding/`, `signup/`, `welcome/`, `splash/` directories — first-flow surfaces
- `marketing/`, `landing/`, the website's hero component or top-level layout
- App store metadata files: `app_store_metadata.json`, `fastlane/metadata/`, `play_store/`, anything with screenshots / descriptions
- Push notification template files, especially the welcome / first-notification template

**Diff patterns that always trigger** (regardless of file):

- A new route, screen, or tab added to a navigator's primary tab list
- A change to a list/array literal that holds nav items, tab labels, or onboarding step labels
- Copy changes to strings named `hero_*`, `headline_*`, `welcome_*`, `tagline`, `app_description`, `cta_primary`
- An icon or label change on an existing nav tab
- A reorder of tabs (the leftmost / first tab carries disproportionate weight)

**Phrasing in the user's request that always triggers** (even if no file pattern matched yet):

- "add a `<word>` tab"
- "show `<thing>` on the home screen"
- "rename the `<word>` tab"
- "change the hero / headline / tagline"
- "update the app store description"
- "I want users to land on `<screen>` after login"
- "promote `<feature>` to the nav"

**Things that explicitly DO NOT trigger:**

- Changes inside `account/`, `settings/`, `profile/`, `wallet/` sub-screens (non-primary by definition)
- A new modal sheet opened from a deliberate user action (non-primary)
- Backend-only changes — no user-visible surface
- Internal admin tools
- Test files, generated files, lockfiles
- Strategy docs / PRDs (different skill: `strategy-doc`, planned)

## Run

### 1. Identify the product

Determine which positioning applies based on the repo:

| Repo | Product | Positioning |
|------|---------|-------------|
| `tennis-miami`, `tennis-miami-web` | Tennis Miami (consumer) | community-first |
| `arenna-link` | ARENNA (instructor side of the consumer surface) | instructor-first |
| Future consumer repos | Look up in the principles doc; if missing, STOP |

If the repo isn't in the principles doc's per-product applications section, do not guess. Tell the user the principles doc needs an entry for this product before the check can run.

### 2. Read the source of truth

Read `_shared/strategy/product-positioning-principles.md`. Confirm:

- Status line says "locked"
- The relevant per-product application section exists
- The "what counts as a primary surface" section is intact

If any of these are missing, surface that and stop.

### 3. Classify the surface

For each changed file or diff hunk, classify:

- **Primary surface** — matches the file patterns or diff patterns in *When this fires*. Run the full check.
- **Non-primary surface** — account/settings/profile, modal sheet, backend, test. Skip the check; emit a one-line "no positioning impact" so it's visible the skill considered it.
- **Ambiguous** — looks like a screen but unclear if it's reached from the nav or from a sub-action. Ask the user one question: *"Is `<screen>` reachable from the bottom nav, or only from inside another flow?"*

### 4. Run the decision test

For each primary-surface change, mentally run the test from the principles doc:

> Show the surface to someone who has never used the product. Ask them: *"in one sentence, what is this app?"* Does the answer match the product's stated positioning?

Concretely, for a nav change:
- Read out the resulting tab labels in order: e.g., *"Home, Players, Matches, Wallet, Offers"*
- Ask: if those five words were the app's tagline, what would a stranger think this app is for?
- Compare to the principle: Tennis Miami's primary surfaces should answer *"this is the place where Miami tennis players find each other and play."*
- If the gap is wide (e.g., "Wallet, Offers" forces a marketplace reading) — fail.
- If the gap is narrow or arguable (e.g., "Pay" tab vs "Pay" button — both lean transactional but degree differs) — flag for Yuval, don't auto-block.

### 5. Cross-check the anti-patterns

Compare the change to the explicit anti-pattern list in the principles doc. Examples:

- Bottom-nav tab named after a transaction (`Wallet`, `Offers`, `Buy`, `Shop`, `Subscribe`, `Deals`)
- First screen after login leading with a price or discount
- App store first sentence naming the wrong cold-start side
- "Browse" / "Discover" tab on the cold-start side's product

If any anti-pattern matches, the result is a hard fail (not a flag).

### 6. Produce the output

The output is structured. Three possible verdicts:

**PASS** — change does not affect a primary surface, or affects one and the decision test resolves cleanly. One-line emission:

> Positioning check: ✅ no impact on primary-surface positioning. (Reason: `<surface>` is non-primary / change preserves the community-first answer / etc.)

**FLAG** — change affects a primary surface and the decision test is arguable. Emit:

> Positioning check: ⚠️ this change touches a primary surface. The principle at risk: **`<short principle>`**. The decision test reading: *"`<what a stranger would say>`"* — compare to the stated positioning *"`<what the doc says>`"*. This may be intentional, but it should have explicit sign-off from Yuval before merge. Suggested non-primary alternative: `<concrete alternative>`.

**FAIL** — change matches an anti-pattern or the decision test reads clearly wrong. Emit:

> Positioning check: ❌ this change conflicts with the locked product positioning. Principle: **`<short principle>`**. What this surface would now answer to a first-time user: *"`<wrong reading>`"*. What it should answer: *"`<right reading>`"*. Concrete fix: move `<feature>` from `<primary surface>` to `<non-primary surface>` — e.g., `<example>`. The feature is fine; the placement is the issue. See `_shared/strategy/product-positioning-principles.md` worked example #1 for the canonical case.

For FLAG and FAIL, always include:
- The exact principle being risked (quote the line from the doc)
- A concrete suggested alternative (placement, not removal — features are usually fine, surfaces are the issue)
- A pointer to the relevant worked example in the principles doc

Never just say "this is wrong." Say *what* is wrong, *why* per the doc, and *where it should live instead*.

### 7. Don't gate, don't lecture

This skill emits a verdict. It does not block the PR or the commit. The non-engineer skill (`non-engineer-frontend-contribution`) handles git flow; this one informs the diff. Two reasons:

- Sometimes the positioning has changed and the principles doc is the thing that's stale, not the code. The skill must not block valid evolution.
- Lecturing erodes trust. One paragraph, structured, then quiet.

If the skill keeps producing FAIL on the same kind of change across multiple sessions, that's evidence the principles doc needs an update — surface this once at the end of the verdict, don't repeat it.

## Anti-patterns — never do these

- ❌ Block the PR or refuse to commit. This skill informs; the engineer decides.
- ❌ Run the check without reading the principles doc fresh. The doc is the source of truth, not the skill's prior memory.
- ❌ Apply Tennis Miami's positioning to ARENNA or vice versa. The two products take opposite cold-start sides.
- ❌ Flag a non-primary surface change. Wallet inside `Account` is fine. Wallet in the nav is not. Know the difference.
- ❌ Conflate "feature exists" with "feature is in the nav." The wallet feature is locked business model; the wallet *tab* is a positioning question. These are independent.
- ❌ Lecture. One paragraph, structured, then quiet.
- ❌ Treat a draft principles doc as locked. If status isn't "locked," surface that and run advisory only.
- ❌ Guess a product's positioning. If a new repo isn't in the doc, stop and tell the user the doc needs an entry.
- ❌ Repeat the same FAIL verdict across multiple changes in one session without escalating once: "I've flagged this pattern N times — is the principle stale?"

## When the check is wrong

The principles doc encodes a snapshot of the strategy. Strategy evolves. If the engineer pushes back with a coherent reason — *"we're explicitly trying to shift Tennis Miami toward marketplace because the cold-start solved itself"* — the right move is:

1. Acknowledge the verdict was based on the current doc.
2. Suggest the conversation happen against the doc, not the diff: *"if Tennis Miami's positioning is changing, update `product-positioning-principles.md` first, then this change becomes a clean PASS."*
3. Don't argue the merits in the PR. The skill's job is to surface drift, not adjudicate strategy.

## Reference

- `_shared/strategy/product-positioning-principles.md` — the source of truth this skill reads. Keep it short and current.
- `_shared/strategy/ai-strategy-handoff.md` — terminology rules and platform context.
- `claude_files/projects/tennis-miami/current-direction-apr2026.md` — Tennis Miami's locked business model decisions (separate from primary-surface positioning).
- `non-engineer-frontend-contribution/SKILL.md` — the git-flow skill this runs alongside.
