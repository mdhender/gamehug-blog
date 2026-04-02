---
title: "Designing the Setup Report"
date: 2026-04-01T21:00:00
---

{{< callout type="info" >}}
   The design for the setup report — the Turn 0 report every empire receives when the GM activates a game — is complete. Implementation is next.
{{< /callout >}}

## What's a Setup Report?

When the GM activates a game, every empire needs to know: here is your colony, here is what you have, here is what your homeworld looks like. That's the setup report. It's Turn 0 — the state of the universe before any orders are processed.

In the old Epimethean Challenge engine, this was a text file mailed to each player. In the new version, it will be a materialized snapshot stored in the database, viewable in the browser and downloadable as JSON.

---

## The Data Warehouse Pattern

Reports are **not** computed on the fly. They're snapshots — frozen copies of game state at a specific point in time, stored in dedicated report tables.

This means:

- Reports are immutable once the turn is locked.
- The GM can re-run the report generator before locking. Re-running deletes and recreates all report data for that turn — fully idempotent.
- Frontend queries hit report tables directly. No joins against live game state.
- Historical reports survive even as the game moves forward. If a colony is destroyed on Turn 5, the Turn 0 report still shows it.

The alternative — querying live tables and trying to reconstruct what the game looked like three turns ago — is a problem we never want to have.

---

## Turn Lifecycle

The `turns` table tracks where a turn sits in its lifecycle:

| Status | Meaning |
|---|---|
| `pending` | Turn exists but reports haven't been generated |
| `generating` | Report generator is running (concurrency guard) |
| `completed` | Reports are generated; GM can re-run or lock |
| `closed` | Locked. Reports are immutable. Next turn can begin. |

Turn 0 is auto-created when the game is activated. The GM assigns empires over time, runs "Generate Reports" when ready, optionally adds more players and re-runs, then locks the turn when satisfied. After locking, no more report generation for that turn.

---

## What the Setup Report Contains

For Turn 0, each empire's report includes:

{{< cards cols="1" >}}
   {{< card
      title="Colony Snapshot"
      subtitle="Name, kind (open surface / enclosed / orbiting), tech level, location coordinates, rations, standard of living, birth rate, death rate. Denormalized — the report carries all the data it needs without joining back to live tables."
   >}}

   {{< card
      title="Inventory"
      subtitle="Everything the colony has: factories, farms, mines, fuel, metallics, non-metallics — each with tech level and assembled/disassembled quantities. Straight from the colony template."
   >}}

   {{< card
      title="Population"
      subtitle="A new concept in the data model. Starting population from the original game: 3.5M unemployable, 3.7M unskilled, 1M professional, 1.5M soldiers. Each class tracks quantity, pay rate, and rebel count."
   >}}

   {{< card
      title="Homeworld Survey"
      subtitle="Planet type, habitability, and all mineral deposits — resource type, yield percentage, quantity remaining. Full visibility on the homeworld without requiring a survey action."
   >}}
{{< /cards >}}

Ships, mining groups, factory groups, farm groups, production, probes, and espionage are all deferred to Layer 2. None of them exist at Turn 0.

---

## Schema Changes to Existing Tables

The setup report work requires changes to existing tables before any report tables are created.

**String codes replace integers.** The `colony_inventory.unit` and `colony_template_items.unit` columns change from integer IDs to string codes (`FCT`, `FRM`, `MIN`, `FUEL`, `METS`, etc.). The `colonies.kind` and `colony_templates.kind` columns change from `1` to `COPN`. String codes make JSON extraction straightforward and align with the original engine's code system.

**New columns on colonies.** The colony gains `name`, `is_on_surface`, `rations`, `sol`, `birth_rate`, and `death_rate` — all needed for report snapshots.

**Population tables.** Two new live tables — `colony_population` and `colony_template_population` — track population classes per colony and per template respectively. Population is driven by the same template system as inventory: the colony template JSON gains a `population` section, and `EmpireCreator` copies it to the colony on creation.

**SQLite constraint.** SQLite doesn't support `ALTER COLUMN`, so the column-type changes use a table-rebuild pattern: create a temp table with the desired schema, copy data with explicit `CASE` mapping, drop the original, rename. Not glamorous, but it's the only safe path on SQLite.

---

## New Enums

Four enums codify the string codes used across the schema:

| Enum | Values |
|---|---|
| `TurnStatus` | `pending`, `generating`, `completed`, `closed` |
| `ColonyKind` | `COPN`, `CENC`, `CORB` |
| `UnitCode` | `FCT`, `FRM`, `MIN`, `FUEL`, `METS`, `NMTS`, `GOLD`, `CONS`, `SPY`, `ANM`, `ASP`, `ATP`, `HDR`, `SEN`, `SHD`, `MSL`, `MSS`, `ENG`, `LFS` |
| `PopulationClass` | `UEM`, `USK`, `PRO`, `SLD`, `CNW`, `SPY`, `PLC`, `SAG`, `TRN` |

---

## Report Snapshot Tables

The report schema is a tree rooted at `turn_reports` (one per empire per turn):

```
turn_reports
├── turn_report_colonies
│   ├── turn_report_colony_inventory
│   └── turn_report_colony_population
└── turn_report_surveys
    └── turn_report_survey_deposits
```

Every parent-child relationship cascades on delete. Re-running the generator deletes the `turn_report` row — and everything beneath it disappears with it.

References back to live entities (colony ID, planet ID) are plain integers, not foreign keys. A colony destroyed on Turn 3 shouldn't invalidate the Turn 0 report that mentions it.

Colony and ship report children use separate tables rather than polymorphic nullable FKs. `turn_report_colony_inventory` and `turn_report_ship_inventory` (Layer 2) are separate tables with clean Eloquent relations and proper FK cascading. No nullable column ambiguity.

---

## The Report Generator

`SetupReportGenerator` is the service that materializes reports. Given a game and its current turn:

1. Acquires a status lock — transitions the turn from `pending` or `completed` to `generating`. Rejects if the turn is already generating or closed.
2. Deletes all existing report data for this turn (idempotent regeneration).
3. Iterates every empire that has a colony. For each:
   - Creates the `turn_report` header.
   - Snapshots each colony with its inventory and population.
   - Snapshots the homeworld survey with all deposits.
4. Sets the turn status to `completed` and records the count.

The whole operation runs in a transaction. If anything fails, the turn reverts to its prior status and no partial data is left behind.

---

## GM Workflow

Four new controller actions on `TurnReportController`:

- **Generate** — runs `SetupReportGenerator` for the current turn. GM-only. Redirects back with a count of reports generated.
- **Lock** — sets `reports_locked_at` and moves the turn to `closed`. No more report generation after this.
- **Show** — renders a text-style report in the browser, following the section structure of the original `turn-report.txt`. GM can view any empire; players can view only their own.
- **Download** — returns the report as a structured JSON file.

---

## Build Plan

The work is organized as 37 tasks across 9 dependency groups:

| Group | Scope | Tasks |
|---|---|---|
| A | Enums and live-schema migrations | 6 |
| B | Update existing models and factories | 7 |
| C | New models and factories | 4 |
| D | Template ingestion updates | 3 |
| E | Business logic extensions | 2 |
| F | Report schema and service | 8 |
| G | Routes, authorization, controller | 3 |
| H | Tests | 2 |
| I | Frontend | 2 |

Groups must be completed in order. Tasks within a group can be done in any sequence.

---

## What's Next

Implementation starts with Group A — the four enums and the SQLite table-rebuild migrations. From there it's a steady march through the dependency chain: update existing models, add population and turns, extend the template system, wire up the report generator, add controller actions, test everything, and build the GM-facing UI.

When this is done, a GM will be able to activate a game and hand every player a report that says: *here is your colony, here is what you have, here is where you are.* That's the starting line.
