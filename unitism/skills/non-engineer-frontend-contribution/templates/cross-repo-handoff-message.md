# Cross-repo handoff message template

When the `non-engineer-frontend-contribution` skill detects that a change needs another repo to also change (or hits a merge conflict, or stumbles into infrastructure territory), it STOPS and produces a message using this template.

The person at the keyboard (Asad or Kath) pastes the message into their chat with Yuval or Yudi. The engineer then pastes the **"Paste-ready brief for Claude Code"** block — as-is — into their own Claude Code session (desktop or mobile). That session should be able to execute the change without any further clarification.

Design rule: **the paste-ready block must be self-contained.** No "see above," no assumptions about context the receiver has, no external links the receiver has to open to understand the task. If the block needs context from higher up in the message, the skill should duplicate that context into the block.

---

## Template (fill in and emit)

```
## Cross-repo change requested

**From:** {{username}} working in `{{frontend_repo}}`
**Branch:** `{{branch_name}}` ({{pr_url_or_none}})
**Reviewers:** Yuval or Yudi — first responder wins

### What {{username}} is trying to ship
{{one-sentence plain-English description of the frontend change}}

### Why this needs engineering help
{{one of: out-of-scope | merge conflict | infrastructure / CI | requires new dependency | requires new env var | requires secret rotation | other — with specifics}}

### The change needed in another repo
**Target repo:** `{{backend_repo_or_other}}`
**What needs to change:**
- {{specific item, not vague request — e.g., "Add `POST /api/v1/sessions` endpoint accepting { instructorId: string, startsAt: ISO8601, durationMinutes: number } and returning the created session with its id"}}
- {{specific item}}
- {{specific item}}

**What should stay the same / not break:**
- {{existing endpoints this must not change}}
- {{existing behavior this must preserve}}

### Risk and impact
- **What this touches:** {{tables / modules / services}}
- **What could break:** {{honest list — e.g., "existing mobile clients that post the old shape"}}
- **What to verify in dev before merging:** {{checklist — e.g., "old endpoint still works", "webhook still receives events", "session creation appears in admin dashboard"}}

### The frontend PR is waiting
Once the backend change lands on `{{backend_repo}}`'s `development`, {{username}} will re-run `done` on the frontend branch and the PR will open cleanly. **Do not merge the frontend PR before the backend change is in `development`.**

---

## Paste-ready brief for Claude Code

Paste this block as-is into Claude Code — desktop or mobile — in the `{{backend_repo}}` repo. It's self-contained.

```
I need to add a backend change to support a frontend PR that's waiting.

Context:
- {{username}} ({{role}}) opened a frontend PR in `{{frontend_repo}}` at {{pr_url_or_none}}
- Their PR depends on a backend change I'm about to make

The backend change:
{{repeat the "change needed" list from above, in imperative form — "Add endpoint X", "Update schema Y", "Add env var Z to the config"}}

Constraints:
- Branch off `development`, PR back to `development`. Do not touch `master`.
- Must not break: {{repeat the "stay the same" list}}
- Tests to pass: {{specific test names or "the existing test suite"}}

When done:
1. Open a PR to `development` with title "{{suggested title}}"
2. In the PR body, link back to the frontend PR: {{pr_url_or_none}}
3. Tag me (@yuval or @yudi) for review
4. After it's merged, tell {{username}} their frontend PR is unblocked

Stack context you'll need:
- {{backend stack — e.g., "NestJS + TypeORM + PostgreSQL on Railway"}}
- {{relevant conventions — e.g., "all endpoints under `/api/v1/`, never remove v1"}}
- {{any repo-specific things — e.g., "register in COMPONENTS.md before merging"}}
```
```

---

## Worked example

What the template looks like when the skill fires for real.

**Scenario:** Asad is adding a "cancel booking" button to the booking confirmation screen in `tennis-miami-web`. The button needs to hit an endpoint that doesn't exist yet.

```
## Cross-repo change requested

**From:** asad working in `tennis-miami-web`
**Branch:** `asad/feat/cancel-booking-button` (https://github.com/yuval/tennis-miami-web/pull/87)
**Reviewers:** Yuval or Yudi — first responder wins

### What asad is trying to ship
A "Cancel booking" button on the booking confirmation page that lets the user cancel a booking they just made.

### Why this needs engineering help
Out-of-scope — the frontend is calling `DELETE /api/v1/bookings/:id` which doesn't exist yet in `unitism-backend`.

### The change needed in another repo
**Target repo:** `unitism-backend`
**What needs to change:**
- Add `DELETE /api/v1/bookings/:id` endpoint, accepting no body, requiring the authenticated user to be either the booking's owner or the instructor
- On success: mark the booking `status: "cancelled"`, emit a `booking.cancelled` event, return 204
- On failure (not owner, not instructor, booking already cancelled, booking already started): return 403 / 409 with error body
- Add a DB migration if the bookings table doesn't yet have a `status` column that accepts "cancelled"

**What should stay the same / not break:**
- Existing `GET /api/v1/bookings/:id` endpoint
- Existing booking creation flow
- Existing webhook subscribers (they currently don't filter by status — make sure `cancelled` bookings don't break any of them)

### Risk and impact
- **What this touches:** `BookingsModule` in `unitism-backend`, the `bookings` table, any place that reads booking status
- **What could break:** clients that assume all bookings are "active" (grep the codebase for hardcoded status assumptions)
- **What to verify in dev before merging:** cancel a test booking from the new button, confirm status updates in admin, confirm the instructor's calendar view drops the cancelled slot, confirm no existing test fails

### The frontend PR is waiting
Once this lands on `unitism-backend` `development`, asad will re-run `done` and his PR will open cleanly. Do not merge the frontend PR before this change is in `development`.

---

## Paste-ready brief for Claude Code

Paste this block as-is into Claude Code — desktop or mobile — in the `unitism-backend` repo.

```
I need to add a backend change to support a frontend PR that's waiting.

Context:
- asad (UX designer) opened a frontend PR in `tennis-miami-web` at https://github.com/yuval/tennis-miami-web/pull/87
- His PR adds a "cancel booking" button that hits an endpoint that doesn't exist yet

The backend change:
- Add `DELETE /api/v1/bookings/:id` endpoint, no body, auth required
- Authorization: allow if authenticated user is the booking owner OR the instructor; otherwise 403
- On success: set booking.status = "cancelled", emit `booking.cancelled` event, return 204
- On conflict (already cancelled / already started): return 409 with { error: "booking_not_cancellable", reason: "..." }
- If `bookings.status` doesn't yet accept "cancelled", add a migration for it

Constraints:
- Branch off `development`, PR back to `development`. Do not touch `master`.
- Must not break: existing GET /api/v1/bookings/:id, existing booking creation, existing webhook subscribers
- Tests to pass: existing bookings suite must still pass; add tests for the four auth paths (owner, instructor, other user, unauthenticated)

When done:
1. Open PR to `development` with title "feat(bookings): add cancel endpoint"
2. PR body links to https://github.com/yuval/tennis-miami-web/pull/87
3. Tag @yuval or @yudi for review
4. After merge, message asad: "your tennis-miami-web PR is unblocked, run `done` again"

Stack context:
- NestJS + TypeORM + PostgreSQL on Railway
- All endpoints under `/api/v1/`, never remove v1
- Modular monolith — logic in `BookingsModule`, shared services via `/api/v1/projects/*` only if generalizing
- Register new endpoints in COMPONENTS.md before merging
```
```
