---
title: "Laravel: Game Generation Workflow"
date: 2026-03-30T18:00:00
---

{{< callout type="info" >}}
   The game generation workflow is complete. A GM can now take a freshly created game through star generation, planet generation, deposit generation, home system placement, empire assignment, and activation — all from a single page, all driven by a deterministic seeded PRNG.
{{< /callout >}}

## What We Built

This session delivered the full **game generation workflow** — the process by which a GM transforms an empty game record into a playable universe. It touches every layer of the application: migrations, models, generator services, controller actions, a React frontend, and an extensive test suite.

The work was organized as a 25-task burndown. All 25 tasks are complete.

---

## State Machine

The `games` table gained a `status` column that gates every action in the workflow. Status progresses forward only:

```
setup → stars_generated → planets_generated → deposits_generated → home_system_generated → active
```

Each step must be completed before the next is available. Before the game is `active`, the GM can delete any step — which cascades to all downstream data and reverts the status. Once `active`, nothing can be deleted or modified.

The `Game` model exposes readable helpers (`isSetup()`, `isStarsGenerated()`, etc.) and capability helpers (`canGenerateStars()`, `canDeleteStep()`, `canActivate()`) that the controller and frontend use to enforce these rules consistently.

---

## Data Model

The cluster is stored relationally — not as a JSON blob — because the game engine will rely on this schema for turn adjudication.

{{< cards cols="1" >}}
   {{< card
      title="Stars"
      subtitle="100 stars placed in a 31×31×31 coordinate cube. Stars sharing coordinates form a system group, distinguished by a sequence number. No separate systems table."
   >}}

   {{< card
      title="Planets"
      subtitle="Each star has up to 11 orbital slots. Planets carry type (terrestrial, asteroid, gas_giant), habitability (0–25), and an is_homeworld flag."
   >}}

   {{< card
      title="Deposits"
      subtitle="Each planet has up to 40 deposits. A deposit has a resource type (gold, fuel, metallics, non-metallics), a yield percentage, and a quantity remaining."
   >}}

   {{< card
      title="Home Systems"
      subtitle="An ordered queue of stars designated for starting colonies. Each home system links a star to its homeworld planet and tracks its position in the queue."
   >}}

   {{< card
      title="Empires and Colonies"
      subtitle="An empire links a game_user pivot record to a home system. Each empire gets a starting colony on the homeworld planet, seeded from the colony template."
   >}}

   {{< card
      title="Generation Steps"
      subtitle="Each PRNG-consuming event writes a generation_steps record capturing the input and output PRNG state. Records are deleted — not retained — when a step is rolled back."
   >}}
{{< /cards >}}

Each entity carries `game_id` directly so bulk deletion is a single scoped query rather than a cascade through join tables.

---

## Templates

The GM uploads two JSON templates before generating anything.

The **home system template** defines the complete planetary layout for a home system star: orbits, planet types, habitability, deposits, and which planet is the homeworld. When the template is applied to a star it replaces all existing planetary data — kill-and-fill, not merge.

The **colony template** defines the starting colony type, tech level, and unit inventory for each new empire.

Templates are validated on upload, stored relationally, and locked once the game becomes `active`.

---

## Generator Services

Three services share the same basic contract: accept a `Game`, acquire a database-level lock on the game row, consume the PRNG from where the prior step left off, write the results, save the new PRNG state, record a generation step, and advance the status.

**`StarGenerator`** initializes `GameRng` from the game's `prng_seed` (or a GM-supplied override), places 100 stars, and saves the resulting state. The seed override doesn't permanently change `game.prng_seed` — if the GM discards the stars and regenerates with no override, they get the same galaxy they would have gotten originally.

**`PlanetGenerator`** picks up the PRNG state from star generation and distributes planets across all 100 stars. Planet count, type, and habitability are all driven by the seeded RNG, so the same upstream state always produces the same planetary layout.

**`DepositGenerator`** picks up from planet generation and populates deposits for every planet. Same determinism guarantee.

All three generators are tested with fixed seeds: the same seed always produces the same output, and the tests assert that directly.

---

## HomeSystemCreator

Home system creation works in two modes.

**Random selection** picks up the current PRNG state, finds a star that is at least `game.min_home_system_distance` Euclidean units from every existing home system star, applies the home system template to it (killing all prior planetary data), creates the `HomeSystem` record, and saves the new PRNG state. The minimum distance defaults to 9 and is configurable per game.

**Manual selection** accepts a specific star from the GM. No distance constraint is enforced. No PRNG is consumed. No step record is written — manual placements are transparent to the PRNG chain.

Both modes reject a star that has already been designated as a home system.

---

## EmpireCreator

Empire creation runs after the game is `active`. The service accepts a game, a `game_user` pivot record, and an optional target home system.

Without a target, it finds the first home system in the queue (ordered by creation order) that still has capacity. "Full" is a fixed constant: 25 empires on a home system's homeworld planet. If every home system is full, the service throws — the GM must create a new home system before retrying. The system never silently creates a home system on the GM's behalf.

After creating the empire record, the service seeds a starting colony on the homeworld planet from the game's colony template.

Guards: the game caps at 250 empires total, a `game_user` can only have one empire per game, and a deactivated member cannot be assigned a new empire (though their existing empire persists).

The GM can also **reassign** an existing empire to a different home system. The colony moves to the new homeworld planet.

---

## Delete Step Cascade

Before activation, the GM can roll back any step. The cascade rules are strict:

| Deleting       | Also deletes                                    | Status reverts to    |
|----------------|-------------------------------------------------|----------------------|
| `home_systems` | Home systems, empires, colonies                 | `deposits_generated` |
| `deposits`     | Deposits + everything above                     | `planets_generated`  |
| `planets`      | Planets + everything above                      | `stars_generated`    |
| `stars`        | Everything                                      | `setup`              |

When a step is deleted, `game.prng_state` is restored to that step's `input_state` so the next run starts from the same point the deleted run did. Every delete action requires a confirmation dialog on the frontend describing what will be destroyed.

---

## Generate Page

The frontend is a single React page at `/games/{game}/generate`. It is state-driven: each section (Templates, Stars, Planets, Deposits, Home Systems, Activate, Empires) is enabled or disabled based on `game.status`. Sections after the current step are shown but inactive until their prerequisites are met.

The Stars section shows an inline table of all 100 stars at `stars_generated`, where the GM can edit coordinates before planets are generated. The Planets section does the same for planet attributes before deposits are generated. Both sections include a "Delete Step" button with a confirmation dialog.

The Home Systems section shows the current queue — star location, queue position, and empire count against the 25-slot capacity — and provides buttons for both random and manual home system creation.

The Empires section shows each player member alongside their assigned empire (or an "Assign Empire" button if they don't have one). The GM can pick a specific home system or use first-available. When all home systems are at capacity, the buttons are disabled and a message directs the GM to create a new home system first.

A **Download JSON** link appears in the Stars section whenever cluster data exists. It calls `GET /games/{game}/generate/download` and returns the full cluster — stars, planets, and deposits — as a formatted JSON attachment. Useful for offline analysis and archiving.

---

## Concurrency

Only one generation process can run for a given game at a time. Each generator acquires a `lockForUpdate()` on the game row before writing anything. A concurrent request will block on the lock and pick up after the first writer finishes — it will then see the updated status and reject the request as a duplicate.

---

## Tests

163 new tests cover the generation workflow across 13 test files.

**Generator services** (65 tests):
- `StarGeneratorTest` — 10 tests including fixed-seed determinism assertions
- `PlanetGeneratorTest` — 11 tests
- `DepositGeneratorTest` — 10 tests
- `HomeSystemCreatorTest` — 20 tests covering both modes, distance enforcement, duplicate star rejection
- `EmpireCreatorTest` — 14 tests covering happy paths, capacity limits, 250-empire cap, deactivated member guard, and colony template application

**Controller actions** (98 tests):
- `GameGenerationControllerTest` — 35 tests for the show action, template uploads, and all generate actions
- `GameGenerationControllerDeleteStepTest` — 18 tests covering all four cascade paths
- `GameGenerationControllerCreateHomeSystemTest` — 12 tests
- `GameGenerationControllerEmpireTest` — 10 tests
- `GameGenerationControllerUpdateStarTest` — 8 tests
- `GameGenerationControllerUpdatePlanetTest` — 7 tests
- `GameGenerationControllerActivateTest` — 4 tests
- `GameGenerationControllerDownloadTest` — 4 tests

---

## What's Next

The generation workflow is complete and the game can be activated. The logical next area is the **player-facing game view** — letting players log in and see their empire, their home system, and eventually submit orders. That work will build on the empire and colony records created here.
