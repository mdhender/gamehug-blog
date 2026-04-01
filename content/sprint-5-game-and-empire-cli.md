---
title: "Sprint 5: Empires on the Roster"
date: 2026-03-25T10:00:00
---

{{< callout type="info" >}}
   Sprint 5 is complete. The CLI can now create a game, register empires, and hand out magic link URLs — all from the command line.
{{< /callout >}}

## What We Built

Sprint 4 gave us a universe. Sprint 5 gives us **players in it**.

This sprint added four CLI commands for managing game setup files — the configs that track which empires are in a game and how they authenticate:

- **`cli create game --path DIR`** — initializes an empty `game.json` and `auth.json` in a directory. This is the starting point for a new game.
- **`cli create empire --path DIR [--empire N]`** — registers an empire, assigns a magic link UUID, and creates the empire's data directory. Pass `--empire 0` (or omit the flag) to auto-assign the next number.
- **`cli remove empire --path DIR --empire N`** — deactivates an empire. The entry stays in `game.json` (with `active: false`) and its magic link is revoked.
- **`cli show magic-link --path DIR --empire N --base-url URL`** — prints the full magic link URL for an empire. Pipe it, paste it, send it to a player.

These commands manage two simple JSON files — `game.json` (the empire roster) and `auth.json` (the magic link registry). They're not the full game state; they're the setup layer that says "these empires exist, and here's how they log in."

---

## Auto-Numbering and the Small Things

When you add an empire without specifying a number, the service finds the highest existing empire number and adds one. Empty game? You get empire 1. Game with empires 3 and 7? You get empire 8. It's a small convenience, but it means a referee can rapidly spin up a game:

```bash
cli create game --path data/beta
cli create empire --path data/beta    # → empire 1
cli create empire --path data/beta    # → empire 2
cli create empire --path data/beta    # → empire 3
```

Each call prints the assigned number and the magic link UUID. The referee sends the link to the player. Done.

---

## The Review Found Five Issues

After the five tasks were finished and passing, we ran a post-sprint review. It caught **five issues** — two SOUSA violations, one missed requirement, and two code smells. All fixed before closing.

{{< cards cols="1" >}}
   {{< card
      title="Missing empire directory"
      subtitle="AddEmpire wasn't creating the empire's data directory on disk. Fixed by adding CreateEmpireDir to the store port and calling it from the service."
   >}}

   {{< card
      title="Filesystem calls in the app layer"
      subtitle="CreateGame was calling os.Stat directly — a SOUSA violation. The fix moved existence checks (ValidateDir, GameConfigExists, AuthConfigExists) into the store port where they belong."
   >}}

   {{< card
      title="Hardcoded URL in the wrong layer"
      subtitle="ShowMagicLink was building the full URL in the app service. URL formatting is a presentation concern. Now the app returns just the UUID; the CLI command takes a --base-url flag (defaulting from EC_BASE_URL) and formats it."
   >}}
{{< /cards >}}

The other two were smaller: a `fmt.Sprintf` path instead of `filepath.Join` (resolved by the SOUSA fix), and `cmdShowVersion` landing in the wrong layer (resolved by promoting `version` to a top-level command).

---

## What's Not Here Yet

- **No empire placement** — empires are registered, but they don't have starting colonies on the map yet.
- **No order parsing** — orders are stored but not interpreted.
- **No turn processing** — the engine pipeline hasn't started.
- **No database** — still file-backed.

---

## What's Next

The pieces are accumulating. We have a cluster generator (Sprint 4) and an empire roster (Sprint 5). The next steps toward a playable game:

- **Empire placement** — dropping starting colonies onto habitable planets in the generated cluster
- **Order parsing** — reading the text orders players submit each turn
- **Turn execution** — the engine that processes orders against the game state
- **Report generation** — producing per-empire turn reports

That's the core loop. Each sprint gets closer.

---

## Version

The project is now at **v0.4.0-alpha**. The CLI builds, manages game setup, and passes all tests.
