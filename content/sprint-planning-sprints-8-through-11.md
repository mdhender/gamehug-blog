---
title: "The Road to v0.7: Sprints 8–11 Planned"
date: 2026-03-25T16:00:00
---

{{< callout type="info" >}}
   Sprints 8 through 11 are planned and ready to execute. Four sprints in one session — homeworld templates, colony seeding, a dashboard API, and the frontend to display it all.
{{< /callout >}}

## What Just Happened

Sprint 7 put empires on the map. Before writing a single line of new code, we spent a session planning the next four sprints in full.

That might sound like overhead, but it's the opposite. Planning four sprints at once let us make design decisions that span all four — catching contradictions early, agreeing on type shapes before anything gets built, and splitting the work so each sprint has a clear deliverable that the next one builds on. The result is four sprint documents with task-level detail: exact files, exact function signatures, exact test names.

The planning session itself was productive enough that it's worth writing up.

---

## The Theme: Prove the Model First

All four sprints share a common thread: **get the setup sequence right before tackling the game loop**.

A play-by-mail game has two distinct phases. Setup is when the gamemaster configures the universe — clusters, homeworlds, empires, starting conditions. Play is when players submit orders each turn and the engine processes them. We've been building setup. We're almost done.

The discipline we've been practicing is to make setup *data-driven* wherever possible. Hard-coded starting conditions become a liability the moment someone wants to run a different scenario. Templates solve that.

---

## Sprint 8: Homeworld Templates

Right now, `cli create homeworld` sets a planet's habitability to 25 and leaves its deposits as-is. That's fine for early development, but it means every gamemaster gets the same habitability and different (random) resources depending on which planet gets chosen.

Sprint 8 changes that. Two template files move into the data directory:

```
data-path/
  homeworld-template.json   ← deposits and habitability for every homeworld
  colony-template.json      ← starting colony for every new empire
  cluster.json
  game.json
  auth.json
```

When `create homeworld` runs, it reads the template, deletes whatever random deposits were on the planet, and replaces them with the template's deposits — same resources, same yields, same quantities, for every homeworld in the game. Habitability comes from the template too. Reproducible starting conditions are now a config file, not a code constant.

Sprint 8 also does the domain groundwork for Sprint 9: it adds `ColonyKind` (open air, orbital, enclosed), group types (`MiningGroup`, `FarmGroup`, `FactoryGroup`), and expands the `Colony` struct to carry inventory and groups. None of that is populated yet — that comes in Sprint 9 — but the types need to exist and compile before we can write the logic.

---

## Sprint 9: Colony Seeding

Sprint 9 completes what Sprint 8 sets up.

When `create empire` runs, it now reads the colony template and builds a real starting colony — not just a placeholder. The colony gets:

- **Kind and tech level** from the template (open air, tech level 1 to start)
- **Inventory** copied from the template — farms, mines, factories, and population units
- **A farm group** — all farming units collected into a single group, organized by tech level
- **Mining groups** — one per deposit on the homeworld planet, mine units split as evenly as possible across them

The mining group algorithm is intentionally minimal. We divide the total mine count across N deposits, distribute any remainder round-robin, and assign sub-groups by tech level. Future sprints will revisit this when we have a better sense of what players actually want. Right now, "something reasonable" is good enough.

Factory units are in the inventory but deliberately unassigned. That's the player's first decision — which factories to activate and for what. The setup orders will handle it.

After Sprint 9, an empire has a real colony with a real inventory and real production capacity. The game setup sequence is complete.

---

## Sprint 10: Dashboard API

With the game state rich enough to be interesting, the frontend needs something to show.

Sprint 10 adds a single endpoint: `GET /api/:empireNo/dashboard`. It returns summary counts — how many colonies by kind, how many ships, how many planets by kind. No individual records yet, just the numbers the dashboard cards need:

```json
{
  "colony_count": 1,
  "colony_kinds": [{ "kind": "Open Air", "count": 1 }],
  "ship_count": 0,
  "planet_count": 1,
  "planet_kinds": [{ "kind": "Terrestrial", "count": 1 }]
}
```

The implementation follows the same pattern as the existing order and report stores: `filestore.Store` already knows the data path, already implements `OrderStore` and `ReportStore`, and now implements `DashboardStore` too. Adding it to `AddRoutes` is one new parameter.

Sprint 10 is backend only. The frontend waits for Sprint 11.

---

## Sprint 11: The Dashboard

The current dashboard is an empire name and two buttons. Sprint 11 replaces it with something worth looking at.

Three cards appear in a grid — colonies, ships, planets — each showing a count and a breakdown by kind. Two of the three link out to summary pages. The sidebar gains three new entries: Colonies, Ships, and Star List.

Two of those sidebar pages have real (if limited) data. The third two are honest placeholders:

- **Ships:** "No ships. (The assemble ship order has not been implemented.)"
- **Star List:** "No stars. (The probe order has not been implemented.)"

This is deliberate. The game has a probe order that will eventually let empires survey distant star systems. It doesn't exist yet. Showing a placeholder with a clear explanation is better than hiding the link — it tells players what's coming and sets expectations correctly.

The Colonies page shows real data from the dashboard endpoint: a table of colony counts by kind. It's not a detailed per-colony view — that comes when we have an individual colony endpoint — but it's accurate and useful.

---

## Four Sprints, One Pattern

Looking at Sprints 8–11 together, the same principle runs through all four:

**Get the data right, then display it.**

Sprint 8 gets the domain types right. Sprint 9 populates them. Sprint 10 exposes them. Sprint 11 shows them. None of these can be reordered without losing something — the domain shape has to exist before the logic that uses it, the logic has to run before the API returns anything real, the API has to exist before the frontend can fetch.

This is what "proving the model" looks like in practice. Each sprint is small enough to finish cleanly, but they compound. By the end of Sprint 11, someone can create a game, place homeworlds, add empires, log in, and see their starting colony reflected in a real dashboard.

That's not the full game. But it's the foundation the full game gets built on.

---

## What's Still Ahead

After Sprint 11, the setup workflow is complete and the UI reflects it. The roadmap items that remain:

- **Order parsing** — interpreting the orders players submit each turn
- **Turn execution** — the engine that runs phases against game state
- **Report generation** — per-empire turn reports
- **SQLite persistence** — replacing the file-backed store

The core game loop. That's the next mountain.

---

## Version

The project is at **v0.6.0-alpha**. The post-sprint review findings from Sprint 7 have been addressed. Sprints 8–11 are planned and ready.
