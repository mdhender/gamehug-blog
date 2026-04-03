---
title: "Group F: Report Schema, Models, and the SetupReportGenerator"
date: 2026-04-03T07:00:00
---

{{< callout type="info" >}}
   Group F is complete. Six report tables, six models, six factories, and the `SetupReportGenerator` service — the system that turns live game state into frozen per-empire snapshots. 16 commits, 40 tests, 539 passing across the full suite.
{{< /callout >}}

## What We Built

Groups A through E laid the groundwork: enums, live-schema changes, population models, template ingestion, Turn 0 auto-creation. Group F is the payoff — the report tables and the service that fills them. After this session, the GM can generate Turn 0 reports and every empire gets a materialized snapshot of its starting position.

---

## The Report Schema (F1–F4)

Six migrations, one per table, building a tree rooted at `turn_reports`:

```
turn_reports
├── turn_report_colonies
│   ├── turn_report_colony_inventory
│   └── turn_report_colony_population
└── turn_report_surveys
    └── turn_report_survey_deposits
```

Every parent-child edge cascades on delete. Re-running the generator deletes the `turn_report` row and everything beneath it vanishes — no orphaned inventory or population rows to clean up.

The snapshot FK strategy was a deliberate choice. `source_colony_id` and `planet_id` on report tables are plain nullable integers, not foreign keys. A colony destroyed on Turn 5 doesn't invalidate the Turn 0 report that references it. The schema tests verify this by inserting report rows with nonexistent source IDs and confirming they persist.

`TurnReportSchemaTest` covers all six tables: column existence, nullability, cascade behavior, and the FK-free snapshot columns. 369 lines, 9 tests.

---

## Models and Enum Casts (F5–F7)

Six Eloquent models mirror the report tree. Each sets `$timestamps = false` (reports are immutable snapshots, not mutable records) and casts enum columns to the same PHP enums as their live counterparts:

| Model | Enum Casts |
|---|---|
| `TurnReportColony` | `ColonyKind` |
| `TurnReportColonyInventory` | `UnitCode` |
| `TurnReportColonyPopulation` | `PopulationClass` |
| `TurnReportSurvey` | `PlanetType` |
| `TurnReportSurveyDeposit` | `DepositResource` |

`TurnReport` is the root — it belongs to a `Game`, `Turn`, and `Empire`, and has `colonies()` and `surveys()` relationships. `Turn` gained a single `reports()` hasMany — the only change to an existing live model in the entire group.

`TurnReportModelTest` covers the full relationship graph: parent traversal, child collections, enum cast round-trips, and mass assignment. 260 lines, 10 tests.

---

## Factories (F8–F9)

Six factory classes, one per model. Each produces valid persisted records with correct enum values — `ColonyKind::OpenSurface`, `UnitCode::Factories`, `PopulationClass::Unskilled`, `PlanetType::Terrestrial`, `DepositResource::Gold`. The colony factory generates realistic float values for `rations`, `sol`, `birth_rate`, and `death_rate`. The survey deposit factory uses 1-based `deposit_no` via `$this->faker->numberBetween(1, 10)`.

`TurnReportFactoryTest` creates one record per factory class and asserts persistence, enum hydration, and parent association. 70 lines, 6 tests.

---

## The SetupReportGenerator (F10–F12)

The service was built in three commits — skeleton with atomic locking, colony snapshot logic, then survey and deposit snapshots.

**Atomic status transition (F10).** The generator opens a transaction and issues a single guarded `UPDATE`:

```php
$updated = Turn::where('id', $turn->id)
    ->whereNull('reports_locked_at')
    ->whereIn('status', [TurnStatus::Pending, TurnStatus::Completed])
    ->update(['status' => TurnStatus::Generating]);
```

If `$updated === 0`, the turn was in an invalid state — already generating, closed, or locked — and the service throws a `RuntimeException`. No race condition window: the `UPDATE` and the check are the same query.

**Colony snapshots (F11).** The service eager-loads every empire that has colonies, along with `colonies.planet.star`, `colonies.inventory`, and `colonies.population`. For each colony, it creates a `TurnReportColony` with denormalized star coordinates (`star_x`, `star_y`, `star_z`, `star_sequence`) and orbit, then copies inventory and population rows. Denormalization means the report never joins back to live tables.

**Survey and deposit snapshots (F12).** Each empire's homeworld gets a `TurnReportSurvey` with planet type, habitability, and the same denormalized star coordinates. Deposits are copied with 1-based `deposit_no` values assigned by iteration order.

**Idempotency.** Before creating a new report, the service deletes any existing `TurnReport` for the same turn and empire. Cascade handles the children. Re-running the generator produces identical results.

The whole service is 128 lines. No controller wiring, no routes, no frontend — pure data materialization.

`SetupReportGeneratorTest` covers all paths: pending and completed turns accepted, generating/closed/locked turns rejected, one report per empire with colonies, colony-less empires skipped, denormalized coordinates, inventory and population snapshots, survey and deposit snapshots with 1-based numbering, idempotent re-runs, completion status, and return count. 335 lines, 15 tests.

---

## Verification and Acceptance (F13)

Pint clean. 40 Group F tests pass. 539 tests pass across the full suite — no regressions from Groups A through E. All 11 acceptance criteria verified and checked off:

{{< cards cols="1" >}}
   {{< card
      title="Schema and FK strategy"
      subtitle="Six tables with cascade deletes. Snapshot references are plain integers, not foreign keys — historical reports survive entity deletion."
   >}}

   {{< card
      title="Models and factories"
      subtitle="Six models with enum casts matching live counterparts. Six factories producing valid persisted records. $timestamps = false on all report models."
   >}}

   {{< card
      title="Atomic transitions and idempotency"
      subtitle="Single guarded UPDATE for status locking. Delete-and-recreate for re-runs. Full transaction wrapping — no partial data on failure."
   >}}

   {{< card
      title="Scope control"
      subtitle="No controllers, routes, policies, or frontend changes. One relationship added to Turn. Everything else is new."
   >}}
{{< /cards >}}

---

## What's Next

Group G: the controller actions. Generate, lock, show, and download — wiring the `SetupReportGenerator` into the GM workflow so reports can be triggered from the browser. After that, Group H adds integration tests and Group I builds the GM-facing UI.
