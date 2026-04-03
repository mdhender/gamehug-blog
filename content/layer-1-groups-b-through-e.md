---
title: "Layer 1, Groups B–E: Building the Infrastructure for Turn Reports"
date: 2026-04-02T23:00:00
---

{{< callout type="info" >}}
   Groups B through E of the setup report Layer 1 are complete. Four dependency groups, 30+ tasks, 9 new models and factories, a rewritten template ingestion pipeline, and Turn 0 auto-creation on game activation — 499 tests passing.
{{< /callout >}}

## What We Built

[Group A](/layer-1-group-a-schema-foundations) laid the schema: enums, migrations, three new tables. Groups B through E are the model layer built on top of it — casts, relationships, population support, and the business logic that ties everything together. The session also cleaned up a few things that became visible once all the new infrastructure was in place.

---

## Before Group B: Template Format and Deposit Codes

Two preparatory changes landed before the model work began.

The colony template JSON format was redesigned to support multiple colonies per file. The old format described a single colony; the new format is an array of colony definitions. Each template entry carries an `id` (the `ColonyKind` code — `COPN`, `CORB`, etc.), an `items` array, and a `population` array. Unit codes switched to an embedded tech-level format (`ASW-1` instead of a separate `tech_level` field). Consumable units like `FUEL` and `STU` have no tech level and are written as bare codes.

`DepositResource` enum values were unified with `UnitCode` — `gold`, `fuel`, `metallics`, `non_metallics` became `GOLD`, `FUEL`, `METS`, `NMTS`. A data migration updated existing rows in `deposits` and `home_system_template_deposits`. This is a prerequisite for report generation, which needs to join deposit codes against inventory codes without a translation step.

---

## Admin Side Work

Three admin features shipped between Group A and the model work:

- **Admin password reset** — a new `sendPasswordResetLink` action on `Admin\UserController`, a confirmation dialog on the user show page, and a flash banner via `HandleInertiaRequests`.
- **Game visibility fix** — `GamePolicy::viewAny` was restricted to GMs only; players with active memberships couldn't reach the games index. Fixed by checking `is_active` membership instead of role.
- **Sidebar Games link** — the sidebar only showed the Games link to admins. `HandleInertiaRequests` now sets `has_active_games` on the shared user object; the sidebar uses that flag.

---

## Group B: Enum Casts and Fillable Columns

Group B updated four existing models and three factories to use the enums from Group A.

`Colony` got the most attention: `ColonyKind` cast on `kind`, six new fillable columns (`name`, `is_on_surface`, `rations`, `sol`, `birth_rate`, `death_rate`), and float casts for the numeric fields. `ColonyInventory` and `ColonyTemplateItem` each got a `UnitCode` cast on `unit`. `ColonyTemplate` got a `ColonyKind` cast on `kind`.

The factories followed: `ColonyFactory`, `ColonyTemplateFactory`, and `ColonyTemplateItemFactory` all use enum values in their `definition()` methods so factory output is valid string-code data rather than stale integers.

Each model and factory has its own test file covering the cast, the raw DB value round-trip, mass assignment, and the relationship chain. Seven test files, all new.

---

## Group C: New Models

Group C added three models and three factories for the tables created in Group A.

**`ColonyPopulation`** and **`ColonyTemplatePopulation`** are paired models for the population tables. Both cast `population_code` to `PopulationClass`. `Colony::population()` and `ColonyTemplate::population()` are `hasMany` relationships using the new models. `ColonyPopulation` includes `rebel_quantity`; `ColonyTemplatePopulation` does not.

**`Turn`** is the turn lifecycle model. It casts `status` to `TurnStatus`, belongs to a `Game`, and has `reportsLocked` scoped by `reports_locked_at`. The test covers each lifecycle status, the nullable lock timestamp, and the composite unique constraint.

The three factories — `ColonyPopulationFactory`, `ColonyTemplatePopulationFactory`, `TurnFactory` — generate valid enum-backed data and include relationship-aware states for test setup.

---

## Group D: Template Ingestion

The template upload pipeline was rewritten to handle the new multi-colony array format.

**`Game::colonyTemplates()`** — a `hasMany` added to the `Game` model. It's the primary association from `EmpireCreator`'s perspective: one game, many colony templates, each with items and population.

**`UploadColonyTemplateRequest`** was fully rewritten. The old validation handled a single-colony flat structure. The new validation handles an array of colonies, validates each `id` against `ColonyKind`, validates each item's unit code against `UnitCode` (with a special path for consumable units that have no tech level), and validates population entries against `PopulationClass`. `STU` was added to the `isConsumable()` check — it's used as a raw structural material without a tech level in the sample data.

**`TemplateController::uploadColony()`** was refactored to delete all existing templates for the game before inserting the new set, then iterate the validated array to create one `ColonyTemplate` with its items and population per entry.

The regression test uses the real `colony-template.json` sample file and asserts the full parsed output: 2 templates (COPN with 17 items and 4 population rows, CORB with 1 item and 4 population rows), spot-checking tech levels (`ASW-1` → `tech_level=1`, `FUEL` → `tech_level=0`, `STU` → `tech_level=0`) and population quantities.

---

## Group E: EmpireCreator and Turn 0

Group E wired the new infrastructure into the game activation path.

**`EmpireCreator`** was extended in two ways. `createColony()` was renamed to `createColonies()` (void return) and now uses `colonyTemplates()` with an eager-loaded `items` and `population` to create one colony per template. For each template, it inserts the colony, copies inventory rows (using `->value` for enum-backed columns), and inserts `colony_population` rows with `rebel_quantity = 0`. A game that has a COPN template and a CORB template gets two colonies per empire.

**`GameGenerationController::activate()`** was extended to create Turn 0 within the same transaction as empire creation. After all empires are created, it inserts a `Turn` with `number = 0` and `status = pending`. If empire creation fails, no turn is created; if turn creation fails, empires are rolled back.

`GameGenerationControllerActivateTest` gained 47 tests covering the Turn 0 path: that the turn exists after activation, that it has the correct number and status, and that it's associated with the right game.

---

## Test Coverage

499 tests pass across the full suite after Group E.

{{< cards cols="1" >}}
   {{< card
      title="Group B: 7 test files"
      subtitle="ColonyModelTest, ColonyInventoryModelTest, ColonyTemplateModelTest, ColonyTemplateItemModelTest, ColonyFactoryTest, ColonyInventoryFactoryTest, ColonyTemplateItemFactoryTest. Enum casts, mass assignment, raw DB round-trips, and relationship traversals."
   >}}

   {{< card
      title="Group C: 5 test files"
      subtitle="ColonyPopulationModelTest (130 tests), ColonyTemplatePopulationModelTest (115 tests), TurnModelTest (113 tests), TurnFactoryTest (67 tests), PopulationFactoriesTest (97 tests)."
   >}}

   {{< card
      title="Group D: UploadColonyTemplateValidationTest (266 tests)"
      subtitle="Validates every rule in the rewritten FormRequest: array structure, ColonyKind, UnitCode with tech-level parsing, consumable units (FUEL, STU), PopulationClass. Plus the real-file regression test."
   >}}

   {{< card
      title="Group E: GameGenerationControllerActivateTest (47 new tests)"
      subtitle="Turn 0 existence, number, status, and game association after activation. Rolls back on failure."
   >}}
{{< /cards >}}

---

## What's Next

Group F: the report schema and `SetupReportGenerator` service — the tables that store materialized snapshots (`turn_reports`, `turn_report_colonies`, `turn_report_colony_inventory`, `turn_report_colony_population`, `turn_report_surveys`, `turn_report_survey_deposits`) and the service that populates them. After that, Group G wires up the controller actions (generate, lock, show, download), Group H adds integration tests, and Group I builds the GM-facing UI.
