---
name: unitism-backend-feature
description: >
  Use this skill whenever the user is working on `unitism-backend` —
  adding, modifying, or removing endpoints under `/api/v1/*`, creating
  or editing modules, writing TypeORM migrations or entities, changing
  DTOs or response shapes, modifying database triggers, or touching
  authentication, units, projects/holons, payments, or any cross-cutting
  platform module. Triggers on requests phrased as "add an endpoint
  for...", "the backend needs to support...", "modify the X module",
  "update the X entity", "create a migration for...", "fix the
  payment / auth / units bug", or any session where the working
  directory is `unitism-backend` or where files under its `src/` are
  being edited. Also triggers when a `cross-repo-handoff-message`
  paste asks for a backend change in `unitism-backend`. The skill
  enforces: COMPONENTS.md / API_GUIDE.md / PROTECTED.md as the canon,
  the `/api/v1/*` versioning contract, attribute inheritance via DB
  triggers, the no-breaking-changes rule, TDD red→green→refactor,
  event-table logging for every mutation, and — critically —
  cross-client verification before declaring done (web AND mobile,
  not just one). Do NOT trigger on frontend-only changes (those go
  to `non-engineer-frontend-contribution`), on infrastructure or CI
  changes, or on documentation-only edits. For ARENNA cross-repo
  features that touch `unitism-backend` AND `arenna-agent-server`
  AND `sessionizer` in one flow, this skill runs for the
  `unitism-backend` portion only.
---

# UNITISM backend feature

## Purpose

Every cross-cutting capability in the UNITISM ecosystem — units, projects/holons, auth, payments, transparency — lives in `unitism-backend`. ARENNA, Tennis Miami, Kairos, `unitism-web`, and `arenna-agent-server` all consume it as a single source of truth via versioned REST. A drift in the backend contract silently breaks every client at once.

This skill keeps backend work on the rails. It enforces the canonical contract files, the `/api/v1/*` versioning rule, the attribute-inheritance pattern, and TDD discipline. Most importantly, it enforces **cross-client verification before declaring a feature done** — because backend changes that pass on web routinely fail on Android, and the inverse, and neither developer notices until a user reports it.

## Source of truth — read these BEFORE writing code

Three files in `unitism-backend` define the contract for the entire ecosystem. Read them at the start of every backend task; do not work from memory.

1. **`COMPONENTS.md`** — the module catalog. Before adding a new `@Controller` or module, check whether the capability already exists. Extend, don't duplicate. Every new shared capability is registered here in the same PR as the first commit.
2. **`API_GUIDE.md`** — the HTTP contract. Base URLs, auth flow, canonical field names (including the `avatar_url` trap), response envelopes, CORS, validation pipe behavior, versioning. Frontend apps read this directly from GitHub; leaving it stale silently breaks every client.
3. **`PROTECTED.md`** — behaviors that must never regress. Each entry has `Files involved` and `How to verify` sections. Run `protect` Phase 1 (intake) at the start of the task and Phase 2 (verification) before commit.
4. **`CLAUDE.md`** at repo root — repo-specific conventions and the in-flight Project System spec.
5. **`PRIORITY_TODO.md`** — current platform-wide priorities; check whether the work being requested overlaps with something already planned or in progress.

If any of these files are missing or out of date for the area being touched, STOP and surface it before writing code. The canon is the contract.

## Core rules — never violate

These come from `unitism-backend`'s CLAUDE.md and the project handoff prompt. They are non-negotiable.

1. **All endpoints versioned `/api/v1/*`.** Never remove `v1`. Deprecation is allowed; removal is not.
2. **No breaking changes to existing endpoints.** Add fields; never rename or remove. If a rename is unavoidable, ship as a new endpoint under a new path and let clients migrate.
3. **TypeORM migrations only.** Never modify the database manually. Every schema change is a migration; migrations are reviewed and committed.
4. **Every mutation logs to an event table.** Radical transparency is the design principle — `units_event`, `projects_event`, etc. The audit trail is non-optional.
5. **Inherited attributes are enforced at the DB trigger level, not in application code.** `business_id` and `tier` on `projects` are inherited and locked from the parent. The trigger is the source of truth; the application layer must not be relied on.
6. **Bottom-up unit pricing.** Units are assigned only to leaf tasks; container holons aggregate. Adding units to a container is a hard error.
7. **Single DRI per holon.** One person, one outcome, no hiding. Diffuse ownership defeats the closed-loop improvement cycle. The `assignee_id` field is mandatory on leaf tasks.
8. **No hardcoded service URLs.** Env vars only. Cross-service URLs come from `process.env`.
9. **TDD: red → green → refactor.** Write the failing test first. All tests pass before merge.
10. **Register in `COMPONENTS.md` in the same PR** as the first commit of any new shared capability.
11. **Update `API_GUIDE.md` in the same PR** as any change to an endpoint path, DTO field name, or response shape. Stale API_GUIDE.md silently breaks every client that reads it from GitHub.

## The cross-client trap — the most important section in this skill

> *"I thought it worked. I tested on web. Yehuda tested on Android. Only one of us was right."*

This has happened more than once and it is the single most expensive failure mode in backend work. It produces false confidence — the change looks shipped, but half the ecosystem is broken, and the developer who tested doesn't know.

### Why it happens

The backend serves multiple clients with different stacks, different deserialization behavior, and different ways of handling missing or unexpected fields:

| Client | Stack | Tested by | Common gotchas |
|---|---|---|---|
| `unitism-web` | Next.js 16 / TS strict | Yuval | Strict types catch field mismatches at build time. Tolerant of extra fields. |
| `sessionizer` (ARENNA) | Flutter / Dart | Yehuda | Strict deserialization; missing or renamed fields throw or silently drop at runtime depending on how the model is written. |
| `arenna-link`, `tennis-miami` | Flutter | Asad / Kath via `non-engineer-frontend-contribution` | Same class as sessionizer. |
| `arenna-agent-server` | LLM tool calls | Yuval / Yehuda | Errors surface as agent confusion, not stack traces. |
| `tennis-miami-web` | Next.js | Yuval / Asad | Same class as `unitism-web`. |

The trap: **the same backend change behaves differently across these.** Four concrete patterns show up repeatedly:

**Pattern 1 — silent field drop.** A field is renamed (`photo_url` → `avatar_url`). The web client's TypeScript catches it at build time and forces the rename. The Flutter client's Dart deserializer either throws at runtime or silently drops the field, depending on how the model is written. Web works; mobile silently shows missing avatars.

**Pattern 2 — ValidationPipe whitelisting.** NestJS's `ValidationPipe` with `whitelist: true` strips fields not on the DTO. If a client sends a field the DTO doesn't list, the field is silently dropped, the endpoint returns 200, the mutation appears to work, and the data isn't actually persisted. This is the documented `avatar_url` trap (see PROTECTED.md). It can present as "web works, mobile doesn't" purely because the two clients send different field shapes.

**Pattern 3 — response shape change.** The web client's TypeScript adapts to a new envelope shape on next build. The Flutter client, frozen at the prior API client version, parses the response by exact key match and breaks at runtime.

**Pattern 4 — auth flow asymmetry.** Web and mobile use different storage for JWTs (localStorage vs native secure storage), different refresh patterns, different timeouts. A change to auth middleware can pass on web (where the token gets refreshed silently on 401) and fail on mobile (where the same 401 logs the user out). Payment flows are especially sensitive to this — most "payment is broken on Android" reports trace back to auth, not to payment logic.

### What to do — every backend change, no exceptions

Before declaring a backend feature done:

1. **List the clients this endpoint touches.** Open `COMPONENTS.md` and the consuming repos. Concretely: which of `{unitism-web, sessionizer, arenna-link, tennis-miami, tennis-miami-web, arenna-agent-server}` calls this code path? If the answer is "I don't know," that itself is a finding — find out first by grepping the endpoint URL or controller method name across each consuming repo.

2. **For each affected client, identify the call site.** A grep is the minimum. If the endpoint is consumed via a generated client (OpenAPI codegen), regenerate the client and check the diff. If a field was renamed and the generated client compiles unchanged, that's a red flag — the rename probably didn't propagate.

3. **Add a contract test, not just a unit test.** Unit tests on the controller don't catch client-mismatch bugs. Minimum bar:
   - An integration test that hits the endpoint with the *exact* request body each consuming client sends — not a hand-rolled "ideal" body.
   - Fixtures stored under `test/client-fixtures/` named per client: `web.json`, `flutter-sessionizer.json`, `agent-server.json`. Each fixture is the actual JSON the client sends, copied from a real network capture, not invented.
   - The test asserts the response shape matches what the client expects to *receive*, not just that the endpoint returns 200.

4. **Manually smoke test from at least two clients before merge.** Not "I'll do it later." Before the PR is opened. Web alone is not enough; Flutter alone is not enough. The first time mobile is the slow one to test, that's the moment to write a small Flutter test harness that hits the endpoint, so the next change is automatable.

5. **If the change touches authentication, payments, or units, smoke test from THREE clients.** Web, Flutter, and (for payments / units) the agent-server tool calls. These three subsystems are the ones where client-mismatch bugs are most expensive — payment failures, lost unit attributions, and auth lockouts each cost more than the smoke test does.

6. **If a regression is found cross-client, capture it in PROTECTED.md.** The entry must include the *why*, not just the what. Example: *"Last broke: `72ea3ca` — assumed `photo_url` was the snake_case of `photoURL`, but the backend field is `avatar_url`; ValidationPipe silently stripped the unknown field so updates returned 200 but never applied."* Future-you will not remember the why; the entry is for them.

### When in doubt, ask the other person to smoke-test before merge

If Yuval is writing the change, Yehuda is the cross-check, and vice versa. The cost of a 5-minute *"can you check this on Android before I merge"* message is far less than the cost of a payment flow that's broken in production for two days. The point is **two clients, two humans, before merge** — not after.

## Workflow

### 1. Read the canon

Before writing any code: open `COMPONENTS.md`, `API_GUIDE.md`, `PROTECTED.md`, `CLAUDE.md`, `PRIORITY_TODO.md`. Skim, don't memorize — know what's there so you can find it when the question comes up.

### 2. PROTECTED.md Phase 1 — intake

Run the `protect` intake. Output every entry's title, grouped by section. Identify which entries overlap with the planned task — files involved, same subsystem, or similar risk class. Be generous; false positives are cheap, misses are expensive. Confirm to the user:

> "PROTECTED.md loaded. N entries. At risk for this task: [list]. Proceeding — will verify all of these before commit."

### 3. Check COMPONENTS.md — does this already exist?

If extending an existing module, name it. If adding new, justify why none of the existing modules fit. Resist creating a new module just because the work feels like it doesn't fit anywhere — usually that means a module needs to grow, not multiply.

### 4. Identify affected clients (the cross-client trap, applied)

Before writing the contract, list every client of the code path being touched. Update the task's plan with the list. If you can't list them, do the grep first.

### 5. Write the API contract first

DTOs, controller signature, response envelope. If the feature adds an endpoint, add the OpenAPI annotations now, not later. The contract is the integration point.

### 6. Write the failing test (RED)

One test that describes the expected behavior. If it's a mutation, the test must include the event-table assertion. If it's a holon endpoint, the test must include the inheritance assertion (correct `business_id` and `tier` flowed from parent).

For cross-client surfaces, the failing test loads the per-client fixture from `test/client-fixtures/` and asserts the response matches what the client expects.

### 7. Implement (GREEN)

Minimum code to pass. Nothing speculative. If a database trigger is needed, the migration goes in this commit, not later. Run the migration locally and confirm it applies cleanly.

### 8. Refactor

With all tests green, clean up duplication. Don't refactor surrounding code that wasn't part of the task — that's a direct cause of regressions in PROTECTED.md entries.

### 9. Run the full test suite

Not just the new tests. Catch regressions early.

### 10. PROTECTED.md Phase 2 — verify

Walk every entry from Phase 1. For each: state the entry name, run `How to verify` if it's a runnable command, mark `OK` / `MANUAL` / `STALE` / `BROKEN`. **Do not commit if any entry is BROKEN.**

### 11. Cross-client smoke test

Web AND Flutter for routine changes. Web AND Flutter AND agent-server for auth, payments, or units. If you can't run one of the clients yourself, message the other person and wait.

### 12. Update COMPONENTS.md and API_GUIDE.md

In the same PR as the first commit. Stale canon silently breaks every client.

### 13. PROTECTED.md Phase 3 — capture

If the task fixed a bug or shipped a new critical path, add or update the PROTECTED.md entry with the commit SHA in `Last broke:` and the *why*, not just the what.

### 14. Commit, PR, link the PROTECTED report

Conventional commit format. PR body includes the `protect` verification report and the list of clients that were smoke-tested.

## Anti-patterns — never do these

- ❌ Hardcoded URLs between services. Env vars only.
- ❌ Adding units to a container holon. Leaf tasks only.
- ❌ Putting business logic in a frontend client. The backend is the single source of truth.
- ❌ Breaking v1 endpoints. Deprecate, don't remove.
- ❌ Modifying the database without a migration.
- ❌ Renaming a DTO field without checking every client. The `avatar_url` trap exists for a reason.
- ❌ Declaring "done" after testing on one client only.
- ❌ Skipping the event-table write because "it's a small mutation."
- ❌ Enforcing inheritance in TypeScript instead of in the DB trigger.
- ❌ Refactoring surrounding code that wasn't part of the task.
- ❌ Creating a new module when the capability fits an existing one.
- ❌ Updating an endpoint without updating API_GUIDE.md in the same PR.
- ❌ Using "tokens" or "coins" anywhere in the code, comments, or commit messages — the unit of account is **units** (see `_shared/strategy/ai-strategy-handoff.md`).

## When this skill doesn't apply

- Frontend-only changes — go to `non-engineer-frontend-contribution`.
- Infrastructure, CI, deploy scripts, or env var changes — those are out-of-scope for this skill and should be flagged for explicit human review.
- Documentation-only changes — no code, no test required.
- Strategy or pitch document edits — go to `strategy-doc` (planned).

If a request looks like it might be backend but the working directory or affected files are unclear, ask once before assuming.

## Reference

- `unitism-backend/COMPONENTS.md` — module catalog (the canon).
- `unitism-backend/API_GUIDE.md` — HTTP contract (the canon).
- `unitism-backend/PROTECTED.md` — never-regress list (the canon).
- `unitism-backend/CLAUDE.md` — repo-specific conventions and Project System spec.
- `unitism-backend/PRIORITY_TODO.md` — current platform priorities.
- `_shared/strategy/ai-strategy-handoff.md` — terminology rules (units, partners, no tokens) and platform context.
- `non-engineer-frontend-contribution/SKILL.md` — frontend counterpart; produces the cross-repo handoff messages this skill receives.
- `cross-repo-handoff-message/SKILL.md` — receiver-side flow when a frontend PR is waiting on a backend change.
