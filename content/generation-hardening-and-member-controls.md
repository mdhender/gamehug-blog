---
title: "Generation Hardening and Member Controls"
date: 2026-03-31T21:00:00
---

{{< callout type="info" >}}
   After shipping the game generation workflow, we ran a structured post-delivery code review, rewrote the planet and deposit algorithms to match the reference game rules, refactored the giant controller and page component into focused pieces, and added the GM member controls needed before turn report work can begin.
{{< /callout >}}

## What We Did

Yesterday's post described the full generation workflow — 25 tasks, 163 tests, every phase from star placement through empire assignment. That shipped. Today's work was about what you do *after* something ships: look hard at it, fix what doesn't hold up under scrutiny, and close the remaining gaps before moving on.

Two themes dominate the day. First, a post-delivery code review catalogued every finding in a burndown and resolved them one by one. Second, planet and deposit generation got a ground-up rewrite — the original algorithms were placeholders, and they needed to be replaced with something that actually reflects the game rules.

---

## Post-Delivery Code Review

After the generation workflow was complete, a structured review turned up 15 findings across correctness, performance, code quality, and maintainability. They were tracked in `BURNDOWN-REVIEW.md` and resolved in order.

### The Mega-Controller

The 667-line `GameGenerationController` was the most pressing problem. A single controller handling template uploads, star generation, planet generation, deposit generation, home system creation, empire creation, step deletion, inline editing, activation, and JSON download is not a controller — it's a namespace. It was broken into seven focused controllers in a `GameGeneration/` subdirectory:

- `GenerationStepController` — generate stars/planets/deposits, delete step
- `HomeSystemController` — create random, create manual
- `EmpireController` — create empire, reassign empire
- `TemplateController` — upload home system template, upload colony template
- `StarController` — inline star editing
- `PlanetController` — inline planet editing

The root `GameGenerationController` keeps only `show()`, `download()`, and `activate()`. Route names are unchanged, so all 163 existing tests passed without modification.

### The Mega-Component

`generate.tsx` had grown to 1,253 lines. The same extraction treatment: nine focused sub-components under `resources/js/pages/games/generate/`, each responsible for exactly one section of the page. Shared types moved to `types.ts`. The orchestrating `generate.tsx` is now around 160 lines — state management and breadcrumb layout, nothing more.

### Performance

Two batch-insert improvements landed in `HomeSystemCreator` and `EmpireCreator`. Previously `applyTemplate()` was creating planet and deposit rows one at a time in a loop. Now it collects all planet rows and calls `Planet::insert()`, queries back by `star_id` to resolve the inserted IDs, then batch-inserts all deposits via `Deposit::insert()`. The same treatment applies to colony inventory creation in `EmpireCreator`.

Additionally, `starList` and `planetList` on the generate page are now deferred props — they're fetched after the initial page load rather than blocking it. The table renders a skeleton while they arrive.

### Correctness Fixes

Several smaller issues turned up:

- `prng_state` was missing from `Game::$fillable`, which caused silent failures when generators tried to save the updated engine state.
- The `Star` model was missing a `hasOne` relationship to `HomeSystem`, breaking eager loads.
- The `GenerationStep` and `HomeSystem` models were using the deprecated `$dates` array instead of the `casts()` method.
- Template JSON validation was running in the controller before the Form Request had a chance to apply it; it was moved into Form Request `after()` hooks where it belongs.
- The test suite was hitting an out-of-memory error due to how the generation tests were bootstrapping game data. Fixed by restructuring the test setup; the property rename from `activePlayers` to `players` resolved a naming inconsistency at the same time.

---

## Planet and Deposit Algorithm Rewrite

The planet and deposit generators that shipped yesterday were functional but not faithful to the game rules. Today they were replaced.

### Planet Placement

The old approach rolled a random orbit count per star and distributed planet types with uniform probability. The new approach works slot by slot:

Each of a star's orbital slots gets an independent roll:
- 29% — terrestrial
- 5% — asteroid belt
- 7% — gas giant
- 59% — empty

Type caps prevent unrealistic distributions. Planets are sorted inner-to-outer after placement. Habitability is now orbit- and type-dependent: inner orbits favour higher habitability for terrestrials, outer orbits favour lower; asteroid belts and gas giants use their own ranges. The calculation runs through lookup tables rather than flat random ranges.

### Deposit Generation

Deposits are now generated from planet-type-specific tables. A terrestrial planet gets different resource distributions than an asteroid belt or a gas giant. Quantity and yield ranges come from the reference pseudo-code rather than ad hoc constants.

A new `GameRng::rollDice()` method handles the dice-table lookups. Both generator test suites were updated to cover the new algorithms. The full generation pipeline — PRNG initialisation, star placement, planet placement, deposit generation, and home system creation — is now documented in `docs/GENERATION.md`.

---

## Production Migration Fix

Production ran the original migration, which created a `game_user` table. That migration was later edited in-place to create `players` instead, so production never received the `players` table. A forward-fix migration was added that detects `game_user`, recreates it as `players` with the full schema (including the `id` surrogate key, `is_active`, and timestamps), migrates existing data, and drops the old table. The migration is a safe no-op when `game_user` is absent — fresh installs have always had `players` and are unaffected.

---

## Developer Tooling

### Seed-Users Command

A new `app:seed-users` Artisan command creates deterministic test users. It accepts a count from 1 to 250 (default 1) and generates users with predictable names (`User 1`, `User 2`) and emails (`user1@gamehub.test`). Blocked in production. A `UserSeeder` wraps it for `db:seed` use. The command is idempotent — re-running it skips any user whose email already exists.

### Deploy Script

The deployment script was failing in non-interactive SSH sessions because `bun` wasn't on `PATH`. The script now sources the correct profile before running any `bun` commands.

---

## Dashboard Fix

Non-admin users were seeing an empty Users card on the dashboard. The guard condition was checking `!== null`, but non-admins receive `undefined` rather than `null` for the admin-only props. The check was loosened to `!= null`, which catches both. Admins still see the card; everyone else doesn't.

---

## Members Tab: Promote and Remove

The Members tab gained two new actions that close the last gaps in GM-side member management.

**Promote to GM** promotes a player to the GM role. Restricted to admins. Guards: the member must not already be a GM and must not have an empire assigned (promoting an empire-holding player to GM would create an ambiguous role, since GMs don't play).

**Remove Member** permanently deletes the player record from the game — not a soft deactivation, but a hard delete of the pivot row. Available to admins and GMs. The same guards apply: the member must not be a GM and must not have an empire. A confirmation dialog fires before the delete is sent. This is the escape hatch for a member added by mistake before play begins; once a player has an empire, removal is no longer available.

Both actions are tested across happy paths, authorization failures, and the empire-guard edge cases — 10 tests in total.

---

## Bug Fixes

Two small UI bugs were resolved:

**Select dropdown clipping** — the game role select on the Members tab was rendering behind content when near the bottom of the viewport. Fixed with a `menuPlacement` adjustment.

**Breadcrumb link** — the generate page breadcrumb was pointing to the wrong URL. Fixed.

---

## Tests

All tests pass. The generation workflow now sits at 163 tests; the new member controls added 10 more; the seed-users command added its own suite. Running the full suite: green.

---

## What's Next

With GMs assigned, empires allocated, and the generation workflow solid, the groundwork for turn processing is in place. The next area of work is the initial **turn report** — reading the game state, computing what each empire sees based on their position and sensor range, and generating the per-player report they'll receive at the end of turn 0.
