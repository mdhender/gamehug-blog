---
title: "Sprint 7: Homeworlds"
date: 2026-03-25T14:00:00
---

{{< callout type="info" >}}
   Sprint 7 is complete. The CLI can now designate homeworlds, assign empires to races, and drop starting colonies onto the map — the last setup step before a game can begin.
{{< /callout >}}

## What We Built

Sprint 5 gave us an empire roster. Sprint 6 cleaned up the plumbing. Sprint 7 puts **empires on the map**.

This sprint added two new capabilities and reworked the file layout that ties them together:

- **`cli create homeworld --data-path /dir`** — selects a terrestrial planet from the cluster and designates it as a homeworld. Sets the planet's habitability to 25, creates a Race record tied to that planet, and marks it as the active homeworld for the next `create empire` call. You can override the selection with `--planet N` or control spacing between homeworlds with `--min-distance N`.

- **`cli create empire --data-path /dir --name "The Hegemony"`** — now does far more than register a number. It assigns the empire to the active homeworld's race, creates a starting colony at the homeworld's star system, scrubs the empire name (stripping HTML and shell-special characters), and generates the magic link. Up to 25 empires can share a homeworld.

The full setup sequence is now five commands:

```bash
cli create game      --data-path /dir
cli create cluster   --data-path /dir
cli create homeworld --data-path /dir
cli create empire    --data-path /dir --name "First Empire"
cli show magic-link  --data-path /dir --empire 1
```

Each step produces exactly the files the next step needs.

---

## The File Layout Changed

The old layout had `create cluster` writing to an arbitrary `--path` and `create game-state` merging the cluster into the game file. That was awkward — two different commands, two different notions of where data lived.

Now everything lives under one `--data-path` directory:

```
data-path/
  game.json      ← empires, races, homeworlds
  cluster.json   ← stars, planets, deposits
  auth.json      ← magic links
  1/             ← empire 1 (orders, reports)
  2/
```

`game.json` and `cluster.json` are separate files read together when the full game state is needed. The `create game-state` command is gone — it was a merge step that no longer has a reason to exist.

---

## New Domain Types

The domain layer grew three new concepts this sprint:

- **`Race`** — tied to a homeworld planet. Each race can host up to 25 empires. A race's ID equals its homeworld's planet ID.
- **`Colony`** — an empire's presence at a location. `create empire` now plants a starting colony at the homeworld's star system with tech level 1.
- **`Coords.Distance`** — Euclidean distance between two points in the cluster. Used to enforce minimum spacing between homeworlds so players don't start on top of each other.

---

## Name Scrubbing

Empire names are now sanitized before storage. The scrubber strips HTML-special characters (`<`, `>`, `&`, `"`, `'`), shell-special characters (`` ` ``, `$`, `;`, `|`, and friends), compresses runs of whitespace, and trims. If what's left is empty, the service rejects the name.

This is a game that will eventually accept player input over HTTP. Cleaning names at the service layer — not the CLI, not the handler — means the rule applies everywhere.

---

## The Post-Sprint Review

After all twelve tasks were done and passing, we ran a code review. It found **seven issues** — all low-to-medium severity, no blockers. Every one was fixed before closing.

{{< cards cols="1" >}}
   {{< card
      title="Redundant ClusterWriter interface"
      subtitle="ClusterWriter was now a strict subset of ClusterStore. Removed it; ClusterService uses ClusterStore directly."
   >}}

   {{< card
      title="Dead pointer variable in AddEmpire"
      subtitle="A *domain.Planet pointer was assigned but never dereferenced — only nil-checked. Replaced with a simpler boolean."
   >}}

   {{< card
      title="CLI printed raw name, not scrubbed name"
      subtitle="The success message showed the name the user typed, not the one that was stored. AddEmpire now returns the scrubbed name and the CLI prints it."
   >}}

   {{< card
      title="Missing Coords.Distance test"
      subtitle="The domain package had zero test files. Added a table-driven TestCoordsDistance covering axis-aligned, diagonal, and zero-distance cases."
   >}}

   {{< card
      title="Stale filename: game_config.go → auth.go"
      subtitle="domain/game_config.go no longer contained GameConfig — just AuthConfig and AuthLink. Renamed to match its contents."
   >}}

   {{< card
      title="Stale type names: GameConfigStore → GameStore"
      subtitle="The interface operated on domain.Game, not GameConfig. Renamed GameConfigStore → GameStore and GameConfigService → GameService across all layers."
   >}}

   {{< card
      title="End-to-end smoke test verified"
      subtitle="The full five-command CLI sequence was run against a temp directory and confirmed working."
   >}}
{{< /cards >}}

---

## What's Not Here Yet

- **No order parsing** — orders are stored but not interpreted.
- **No turn processing** — the engine pipeline hasn't started.
- **No report generation** — no per-empire turn reports yet.
- **No database** — still file-backed.

---

## What's Next

With homeworlds placed and empires on the map, the game setup workflow is complete. What remains is the game itself:

- **Order parsing** — reading the text orders players submit each turn
- **Turn execution** — the engine that processes orders against the game state
- **Report generation** — producing per-empire turn reports

That's the core loop. Once it runs, we have a playable game.

---

## Version

The project is now at **v0.6.0-alpha**. All tests pass, both entry points build, and the full setup sequence runs end to end.
