---
title: "Layer 1, Group A: Schema Foundations for the Setup Report"
date: 2026-04-02T14:00:00
---

{{< callout type="info" >}}
   Group A of the setup report implementation is complete: four enums, two SQLite table-rebuild migrations, and three new tables — 6 tasks, 6 commits, 32 tests, 134 assertions.
{{< /callout >}}

## What We Built

The [setup report design post](/setup-report-design) laid out a 37-task build plan across 9 dependency groups. Group A is the foundation layer — the enums and schema changes that every other group depends on. None of the model work, factory work, or report generation can proceed until these are in place.

Six tasks shipped in order:

- **A1** — four string-backed enums
- **A2** — rebuild `colony_inventory`, `colony_template_items`, `colony_templates`
- **A3** — rebuild `colonies`
- **A4** — create `colony_population`
- **A5** — create `colony_template_population`
- **A6** — create `turns`

---

## A1: Four Enums

`TurnStatus`, `ColonyKind`, `PopulationClass`, and `UnitCode` are string-backed PHP enums under `App\Enums`.

`TurnStatus` tracks where a turn sits in its lifecycle: `pending`, `generating`, `completed`, `closed`. `ColonyKind` covers the three colony archetypes: `COPN`, `CENC`, `CORB`. `PopulationClass` has 9 cases — `UEM`, `USK`, `PRO`, `SLD`, `CNW`, `SPY`, `PLC`, `SAG`, `TRN`. `UnitCode` is the largest at 30 cases, covering every unit type from `AUT` through `RSCH`.

`UnitCode` also doubles as the migration's source of truth. The rebuild migrations use its values directly in the `CASE` mappings, which means any future additions to the enum are visible and deliberate rather than buried in SQL strings.

---

## A2 and A3: The SQLite Rebuild Migrations

SQLite doesn't support `ALTER COLUMN`. Changing `colony_inventory.unit` from integer to string — or adding six columns to `colonies` — requires the temp-table pattern: create a new table with the target schema, copy data over with explicit transformations, drop the original, rename the temp.

A2 rebuilt three tables in a single migration:

- `colony_inventory.unit` — integer → string using a full 30-value `CASE` mapping
- `colony_template_items.unit` — same mapping
- `colony_templates.kind` — `1` → `'COPN'`, and the unique constraint on `game_id` was dropped (a game can now have multiple colony templates)

A3 rebuilt `colonies`:

- `kind` — `1` → `'COPN'`
- Added `name` (default `'Not Named'`), `is_on_surface` (default `1`), `rations` (default `1.0`), `sol`, `birth_rate`, `death_rate` (all `0.0`)

Both migrations are wrapped in `Schema::disableForeignKeyConstraints()` / `enableForeignKeyConstraints()` and include preflight validation: if any row contains an unmapped integer, the migration throws a `RuntimeException` and halts before touching any data.

```php
$invalid = DB::table('colony_inventory')
    ->whereNotBetween('unit', [1, 30])
    ->first();

if ($invalid !== null) {
    throw new RuntimeException(
        "colony_inventory.unit contains unknown integer value: {$invalid->unit}"
    );
}
```

---

## The SQLite FK Testing Gotcha

Testing these migrations surfaced a SQLite constraint that isn't obvious: `Schema::disableForeignKeyConstraints()` is a no-op inside a transaction. Laravel wraps each test in a database transaction via `RefreshDatabase`, and SQLite ignores `PRAGMA foreign_keys = OFF` changes made inside an active transaction.

The fix is `PRAGMA defer_foreign_keys = ON`. Unlike the `foreign_keys` pragma, `defer_foreign_keys` *can* be set inside a transaction — it defers FK checking until the outermost `COMMIT`. Since tests roll back rather than commit, FK constraints are never evaluated. Every test helper that inserts rows without parent records uses this pattern.

```php
private function insertColony(int $id): void
{
    DB::statement('PRAGMA defer_foreign_keys = ON');
    DB::table('colonies')->insert([...]);
}
```

This is documented in `CLAUDE.md` so it doesn't have to be rediscovered.

---

## A4, A5, A6: Three New Tables

Three straightforward additive migrations, each with the same structural pattern: an FK to a parent table with cascade delete, a string code column, and a composite unique constraint.

**`colony_population`** — one row per population class per colony. Columns: `colony_id`, `population_code`, `quantity`, `pay_rate`, `rebel_quantity` (default 0). Unique on `(colony_id, population_code)`.

**`colony_template_population`** — mirrors the above for templates. No `rebel_quantity`. Unique on `(colony_template_id, population_code)`.

**`turns`** — the turn lifecycle table. Columns: `game_id`, `number`, `status` (default `'pending'`), `reports_locked_at` (nullable datetime), plus timestamps. Unique on `(game_id, number)`.

---

## Test Coverage

6 test files, 32 tests, 134 assertions.

{{< cards cols="1" >}}
   {{< card
      title="LayerOneEnumsTest"
      subtitle="Asserts exact case counts and values for all four enums. TurnStatus: 4 cases. ColonyKind: 3. PopulationClass: 9. UnitCode: 30 — verified against the legacy integer mapping table."
   >}}

   {{< card
      title="RebuildColonyInventoryAndTemplatesMigrationTest"
      subtitle="7 tests. Verifies integer-to-string conversion for all three tables, confirms the game_id unique constraint is gone, checks FK metadata via PRAGMA foreign_key_list, and asserts RuntimeException on unknown integers in both colony_inventory and colony_templates."
   >}}

   {{< card
      title="RebuildColoniesTableMigrationTest"
      subtitle="5 tests. Verifies kind conversion, default values for all six new columns, FK preservation, and fail-fast on unknown legacy kind values."
   >}}

   {{< card
      title="CreateColonyPopulationTableMigrationTest"
      subtitle="5 tests. Schema columns, rebel_quantity default, composite unique enforcement, cross-colony code reuse, and cascade delete."
   >}}

   {{< card
      title="CreateColonyTemplatePopulationTableMigrationTest"
      subtitle="5 tests. Same structure as the population table test, plus a regression check confirming the A2 unique-drop on colony_templates.game_id holds."
   >}}

   {{< card
      title="CreateTurnsTableMigrationTest"
      subtitle="6 tests. Schema columns, pending default, composite unique, cross-game turn number reuse, cascade delete, and nullable reports_locked_at."
   >}}
{{< /cards >}}

---

## What's Next

Group B: update existing models and factories to reflect the schema changes — `Colony`, `ColonyTemplate`, and related models get new casts, new relations to the population tables, and factory updates to generate valid string codes and population rows. Group C follows with new models for `ColonyPopulation`, `ColonyTemplatePopulation`, and `Turn`.
