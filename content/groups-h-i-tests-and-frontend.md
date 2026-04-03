---
title: "Groups H & I: Tests and Frontend"
date: 2026-04-03T14:00:00
---

{{< callout type="info" >}}
   Turn reports are now visible to both GMs and players. GMs generate and lock reports from the game management page; players view and download their own setup report from the game page. 13 commits, 889 lines across 13 files, 210 PHPUnit tests green.
{{< /callout >}}

## What Players and GMs See Now

**Players** visiting their game page now see a Setup Report card once reports have been generated. Two links — "View setup report" (opens the Blade text report in a new tab) and "Download JSON" (saves the structured file). If reports haven't been generated yet, the card says so. Players without an empire don't see the card at all.

**GMs** get a new Turn Reports section on the game management page, slotted after the Empires section. It has two action buttons — Generate Reports and Lock Reports — plus a per-empire table showing report status (Generated / Pending) with view and download links for each empire that has one. Locking is styled as a destructive action since it's irreversible; both buttons disable when inapplicable.

Both sets of links use plain `<a>` tags rather than Inertia `<Link>` components because the targets are non-Inertia responses — Blade HTML and JSON attachments.

---

## Group H: Test Hardening

Group H started with a full audit of the 13 pre-existing test suites — 137 tests, 468 assertions — to check for regressions from the Groups A–E schema changes. Everything passed clean. The enum migrations, factory updates, and model casts from earlier groups had already been absorbed by the test fixtures.

H.2 through H.4 confirmed that model tests, upload/generation controller tests, and the EmpireCreator and activation tests all still worked. No fixes were needed in any of them — the earlier groups had done their job.

The real new work was H.5 and H.6.

### Snapshot Lifecycle Tests (H5)

Four new tests in `SetupReportGeneratorTest` covering behaviors that weren't previously exercised:

- **Snapshot immutability** — generate a report, mutate live colony names, inventory quantities, and population quantities, then assert the report still contains the original values.
- **Historical survival** — generate a report, delete the live colony (cascading to inventory and population), assert the report and its snapshot data survive.
- **Rerun refresh** — generate, change live data, regenerate on the same turn. Assert snapshot values update to the new live values and stale child rows are replaced, not accumulated.
- **Multi-colony snapshot** — two colony templates (different kinds), two colonies per empire, one report. Assert two `turn_report_colonies` rows with correct kinds and distinct inventory and population.

The generator test file is now 493 lines with 19 tests.

### Controller Test Gaps (H6)

Four small additions across the existing controller test files:

- Admin happy paths for generate and lock (admin users without a game role can still trigger these actions).
- Turn route scoping for show and download — a turn belonging to a different game returns 404, not someone else's report.

---

## Group I: Frontend Wiring

### Wayfinder Generation (I1)

`php artisan wayfinder:generate` produced typed helpers for `TurnReportController` — `generate`, `lock`, `show`, and `download`. These power all the URL construction on the frontend.

### Report Props on the Generate Page (I2)

`GameGenerationController::show()` now returns a `reportTurn` prop with the current turn's id, number, status, lock timestamp, and two computed booleans: `can_generate` and `can_lock`. The `members` payload gained `has_report` on each empire object, driven by a lookup against `turn_reports` for the current turn.

Six focused prop tests cover the matrix: no turn, pending turn, completed turn, closed turn, empire with report, empire without.

### The TurnReportsSection Component (I3)

`TurnReportsSection.tsx` (146 lines) follows the same patterns as `EmpiresSection` — accepts `game`, `reportTurn`, and `members` props, renders nothing when `reportTurn` is null. Two `useForm` instances drive the Generate and Lock buttons, posting to the Wayfinder-generated URLs. The empire table filters to members with empires and shows status badges and action links.

### Player Setup Report Card (I4)

`GameController::show()` gained a `setupReportPayload` method that finds the authenticated user's empire, checks for a turn report, and returns the availability flag. The game show page renders a Setup Report section when the payload is present — either the two action links or a "not yet generated" message.

Five prop tests: player with report, player without report, player without empire, GM without empire, non-active game.

---

## Final Verification (I5)

210 PHPUnit tests, 720 assertions. TypeScript compiles. ESLint passes. Pint clean. Both GM and player flows verified end-to-end in the browser.

---

## What's Next

Layer 1 for turn reports is complete — schema, service, controller, authorization, tests, and frontend all shipped. The next layer will extend the report system to cover actual game turns beyond Turn 0, with movement orders, combat resolution, and production cycles feeding into the report generator.
