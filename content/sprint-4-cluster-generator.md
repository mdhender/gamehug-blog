---
title: "Sprint 4: Stars in the Machine"
date: 2026-03-24T21:30:00
---

{{< callout type="info" >}}
   Sprint 4 is complete. The CLI can now generate a star cluster — the map every game starts from.
{{< /callout >}}

## What We Built

Sprint 4 delivered the **cluster generator** — the part of the game engine that creates the universe players will explore, colonize, and fight over.

Here's what it does:

- **Generates 100 star systems** inside a 31×31×31 cube, with deterministic placement driven by a seeded PRNG. Same seeds, same galaxy — every time.
- **Populates each star with orbits** — terrestrial planets, asteroid belts, and gas giants, distributed by weighted random tables drawn from the original 1978 rules.
- **Assigns habitability and natural resource deposits** to every planet. Each deposit has a resource type (gold, fuel, metallics, non-metallics), a quantity, and a yield percentage.
- **Outputs a normalized cluster file** — a flat, ID-referenced JSON structure that becomes the starting map for a game.

There's also a **distribution tester** that runs the generator thousands of times and reports statistics: average planet counts, deposit quantities, yield percentages, and habitability breakdowns by planet type. This is how we'll tune the tables as we move from the 1978 baseline toward the later rule sets.

---

## Two Commands, Separate Jobs

The original implementation had a single `create cluster` command that changed behavior based on an `--iterations` flag. The review flagged this as a smell — one command doing two unrelated things — so we split it:

- **`cli create cluster`** — generates one cluster, writes the JSON file, prints a summary report.
- **`cli test cluster`** — runs N iterations (default 100), aggregates the stats, prints the distribution report. No files written.

Different intents, different side effects, separate commands.

---

## The Architecture Review

This sprint started with a code review before the feature was even committed. The review found **six SOUSA violations and six code smells**.

The big one: all the game logic, file I/O, and CLI wiring were sitting together in `cmd/cli/main.go`. The generator types lived in orphan packages (`generators/`, `adapters/`, `fsck/`) that didn't fit any SOUSA layer. The fix was a full decomposition:

{{< cards cols="1" >}}
   {{< card
      title="Generation rules → domain/clustergen"
      subtitle="Orbit tables, deposit probabilities, habitability curves — these are game rules. They belong in the domain layer, and now that's where they live."
   >}}

   {{< card
      title="Use cases → app"
      subtitle="CreateCluster, TestCluster, and CreateGame are now proper use cases with port interfaces. The app layer doesn't know about files or the CLI."
   >}}

   {{< card
      title="File I/O → infra/filestore"
      subtitle="Reading and writing cluster JSON and game JSON moved to the filestore adapter. Overwrite-safety is handled here, not in the CLI."
   >}}

   {{< card
      title="CLI handlers → delivery/cli"
      subtitle="The cobra commands are thin: parse flags, call a use case, format output. The report formatting (the big table of stats) lives here too — presentation, not domain logic."
   >}}

   {{< card
      title="Wiring → runtime/cli"
      subtitle="Concrete infra gets instantiated and injected into app services, which get handed to delivery commands. Same pattern as the API server."
   >}}
{{< /cards >}}

The orphan packages (`generators/`, `adapters/`, `fsck/`) are gone. Every file now lives in a SOUSA layer.

---

## Generator Internals Stay Internal

One detail worth calling out: the generator uses its own tree-shaped types internally — pointer-based structs with nested slices that make the generation code natural to write. These types are **unexported**. The only public function is `GenerateCluster`, and it returns a `domain.Cluster` directly.

This means the generator can change its internal representation freely without affecting any other layer. The rest of the system only ever sees the normalized domain model.

---

## What's Not Here Yet

- **No order parser** — orders are stored but not executed.
- **No turn processing** — the full engine pipeline hasn't started.
- **No empires** — `create game` wraps a cluster in a Game shell, but there are no players in it yet.
- **No database** — still file-backed.

---

## What's Next

With a cluster generator in hand, the next pieces are:

- **Empire placement** — dropping starting colonies onto habitable planets
- **Order parsing** — reading the text orders players submit
- **Turn execution** — the engine that processes orders against the game state
- **Report generation** — turning the new game state into per-empire reports

That's the core loop. Once it runs, we have a playable game.

---

## Version

The project is now at **v0.3.0-alpha**. The CLI builds, generates clusters, and passes all tests.
