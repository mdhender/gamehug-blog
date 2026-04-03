---
title: "Group G: Turn Report Controller and Authorization"
date: 2026-04-03T12:00:00
---

{{< callout type="info" >}}
   GMs can now generate, lock, view, and download turn reports from the browser. Group G wired the `SetupReportGenerator` from Group F into four controller actions behind a dedicated policy. 9 commits, 42 tests, 1,200 lines across controller, policy, and test files.
{{< /callout >}}

## What Players and GMs See

Players can view and download their own empire's turn report. GMs and admins can do that for every empire, plus trigger report generation and lock turns. Reports render as an HTML page or download as a structured JSON file — one file per empire per turn, with colonies, inventory, population, surveys, and deposits all included.

Separately, every user now has a **handle** — a short unique identifier assigned at registration and visible on profiles, the admin panel, and (eventually) in-game. Users can see theirs but can't change it. Admins can.

---

## The TurnReportPolicy (G1)

Authorization lives in a dedicated `TurnReportPolicy` rather than being bolted onto an existing model policy. The split keeps the rules clear:

- **generate** and **lock** — GM or admin only.
- **show** and **download** — GM/admin for any empire, players for their own empire only. Non-members denied.

The shared logic for show/download lives in a private `canViewEmpireReport` method that checks `$empire->player?->user_id === $user->id`. 39 lines, 10 tests covering every role/ability combination.

---

## The Controller (G2–G5)

`TurnReportController` has four actions, each behind Gate authorization:

**generate** (G2) — `POST /games/{game}/turns/{turn}/reports/generate`. Validates the game is active and the turn is Turn 0, then delegates to `SetupReportGenerator`. RuntimeExceptions from the service (wrong turn status, race conditions) are caught and converted to validation errors. Returns a redirect with the report count.

**lock** (G3) — `POST .../lock`. Same active-game and Turn 0 guards. Issues a single guarded `UPDATE` — the same atomic pattern the generator uses — setting `reports_locked_at` and flipping status to `Closed`. If the update touches zero rows, the turn was already locked, closed, or mid-generation.

**show** (G4) — `GET .../empires/{empire}`. Loads the report with eager-loaded colonies, inventory, population, surveys, and deposits (all ordered by ID). Renders a Blade view. Manually checks `$empire->game_id` matches `$game->id` before authorization since the empire binding is unscoped.

**download** (G5) — `GET .../empires/{empire}/download`. Same load logic, but serializes the full report tree to JSON — game metadata, turn status, empire name, colonies with nested inventory and population, surveys with nested deposits. Returns it as an attachment: `report-{game}-turn-{turn}-empire-{empire}.json`.

The shared `loadReport` method handles eager loading for both show and download. The controller is 192 lines.

---

## Routes and Scoped Bindings

The four routes live under `games/{game}/turns/{turn}/reports` with `scopeBindings()` so the turn must belong to the game. The empire routes (`empires/{empire}`) use `withoutScopedBindings()` because empire belongs to the game, not the turn — the controller validates that relationship manually with `abort_unless`.

---

## User Handles

Two commits before the Group G burndown added a `handle` field to users. The migration adds a unique-indexed column and backfills from the first word of each user's name. Handles are lowercase, max 16 characters, accepting letters, numbers, underscores, hyphens, and single quotes.

Users see their handle on the profile settings page as read-only text. Admins get a dedicated `PATCH` route with inline editing on the user detail page. The split keeps the profile form simple and puts handle management where it belongs.

7 new handle tests plus updates to registration, profile, user index, and user show tests.

---

## Tests

5 test files, 42 tests, 969 lines:

| Test File | Tests | Lines |
|---|---|---|
| `TurnReportPolicyTest` | 10 | 153 |
| `TurnReportControllerGenerateTest` | 7 | 149 |
| `TurnReportControllerLockTest` | 10 | 191 |
| `TurnReportControllerShowTest` | 7 | 222 |
| `TurnReportControllerDownloadTest` | 8 | 254 |

Every action is tested for authorization (admin, GM, player, non-member, guest), validation (inactive game, wrong turn number, invalid state transitions), and happy paths. The download test verifies the full JSON structure including nested collections. The lock test covers the atomic update edge cases — already locked, currently generating, and closed turns all rejected.

---

## What's Next

Group H: integration tests that exercise the full pipeline — generate reports, lock the turn, view and download — end to end. After that, Group I builds the GM-facing UI.
