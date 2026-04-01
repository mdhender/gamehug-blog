---
title: "Sprint 8: Templates and Types"
date: 2026-03-26T12:00:00
---

{{< callout type="info" >}}
   Sprint 8 is complete. Homeworld setup is now template-driven — same deposits, same habitability, every time — and the domain types for colonies, groups, and inventory are ready for Sprint 9.
{{< /callout >}}

## What We Built

Sprint 7 put empires on the map. Sprint 8 makes the map **reproducible**.

Before this sprint, `cli create homeworld` set a planet's habitability to 25 and left its deposits alone — whatever the cluster generator rolled, that's what you got. Every homeworld had different resources. That's fine for testing, but it's not fine for a fair game.

Now homeworld setup is driven by two template files that the gamemaster places in the data directory before running any setup commands:

```
data-path/
  homeworld-template.json   ← deposits and habitability for every homeworld
  colony-template.json      ← starting colony for every new empire (Sprint 9)
  cluster.json
  game.json
  auth.json
```

When `create homeworld` runs, it reads the homeworld template, deletes whatever random deposits were on the planet, and replaces them with the template's deposits — same resources, same yields, same quantities, for every homeworld in the game. Habitability comes from the template too. Reproducible starting conditions are now a config file, not a code constant.

---

## The Colony Got Real

The other half of this sprint was domain groundwork. The `Colony` struct went from a stub to something that can actually represent a colony:

```go
type Colony struct {
    ID            ColonyID
    Empire        EmpireID
    Planet        PlanetID        // replaces Location
    Kind          ColonyKind
    TechLevel     TechLevel
    Inventory     []Inventory
    MiningGroups  []MiningGroup
    FarmGroups    []FarmGroup
    FactoryGroups []FactoryGroup
}
```

`Location Coords` is gone. Colonies are tied to a `PlanetID` now — they don't move, so system coordinates are looked up from the planet when needed. The new fields — kind, tech level, inventory, groups — are all present but unpopulated. Sprint 9 fills them in.

---

## New Domain Types

Three kinds of groups were added, all following the same pattern — a typed ID, a slice of `GroupUnit` sub-groups organized by tech level:

- **`MiningGroup`** — assigned to a specific deposit
- **`FarmGroup`** — all farming units on a colony
- **`FactoryGroup`** — all factory units on a colony

`ColonyKind` distinguishes open air, orbital, and enclosed colonies. `Inventory` gained a `QuantityDisassembled` field to track unassembled units.

None of these are used by game logic yet. They exist so Sprint 9 can populate them without changing type definitions in the same sprint that adds behavior. Types first, logic second.

---

## The Template Plumbing

Templates follow the same layering as everything else in the project:

- **`domain/templates.go`** — pure types: `HomeworldTemplate`, `ColonyTemplate`, `DepositTemplate`
- **`app/template_ports.go`** — port interface: `TemplateStore` with `ReadHomeworldTemplate` and `ReadColonyTemplate`
- **`infra/filestore/templates.go`** — the adapter: reads JSON files from the data directory
- **`runtime/cli/cli.go`** — wires `Templates: store` into `GameService`

The pattern is identical to how `GameStore` and `ClusterStore` work. If you've read one adapter in this project, you've read them all.

---

## The Post-Sprint Review

After all seven tasks were done and passing, we ran a SOUSA compliance audit and code review. The audit was clean — no layering violations, no stale references to the removed `Colony.Location` field, `go vet` happy.

The review found **six code smells**, none of them blockers:

{{< cards cols="1" >}}
   {{< card
      title="Missing planetIdx guard"
      subtitle="If the planet loop in CreateHomeWorld doesn't find a match, planetIdx stays at -1 and the code panics. Currently unreachable — the auto-select path always finds a valid planet — but a defensive check should be added."
   >}}

   {{< card
      title="Colony ID generation uses len()+1"
      subtitle="If colonies are ever deleted, this produces duplicate IDs. No deletion path exists today, but it's a known limitation to track."
   >}}

   {{< card
      title="Template methods accept dataPath as a parameter"
      subtitle="Inconsistent with GameStore and ClusterStore, which receive the path at construction time. Acceptable for read-only one-shots, but worth aligning later."
   >}}

   {{< card
      title="No template content validation"
      subtitle="Invalid template files (negative habitability, empty deposits) would silently produce bad game state. Validation should be added at the app layer."
   >}}

   {{< card
      title="findSystemForPlanet partially orphaned"
      subtitle="Removed from AddEmpire but still used in CreateHomeWorld's auto-select distance check. Not dead code, but worth monitoring."
   >}}

   {{< card
      title="Unused candidate.location field after selection"
      subtitle="Needed during distance filtering but never read after the loop. Cosmetic — not worth changing."
   >}}
{{< /cards >}}

Items 1 and 2 are flagged for Sprint 9 remediation.

---

## What's Not Here Yet

- **No colony seeding** — the colony struct has the fields, but `create empire` doesn't populate them yet. That's Sprint 9.
- **No order parsing** — orders are stored but not interpreted.
- **No turn processing** — the engine pipeline hasn't started.
- **No database** — still file-backed.

---

## What's Next

Sprint 9 completes what Sprint 8 set up. When `create empire` runs, it will read the colony template and build a real starting colony — inventory, farm groups, mining groups split across deposits. After Sprint 9, an empire has a real colony with real production capacity. The game setup sequence will be complete.

---

## Version

The project is now at **v0.7.0-alpha**. All tests pass, both entry points build, and `go vet` is clean.
