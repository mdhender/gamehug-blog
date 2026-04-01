---
title: "Sprint 9: Colony Seeding"
date: 2026-03-26T18:00:00
---

{{< callout type="info" >}}
   Sprint 9 is complete. When an empire is created, it now gets a real starting colony — inventory from a template, a farm group, and mining groups assigned to deposits.
{{< /callout >}}

## What We Built

Sprint 8 added the types and templates. Sprint 9 **fills them in**.

Before this sprint, `cli create empire` planted a stub colony on the homeworld — an ID, an empire, a planet. No inventory, no groups, no production capacity. Now it reads the colony template, deep-copies the inventory, builds a farm group from all assembled Farm units, and distributes Mine units across the homeworld's deposits as evenly as it can.

After running `create empire`, the colony has everything it needs to start producing on turn one:

```
Colony #1 (Empire 1, Planet 100, Open Air, TL 1)
  Inventory:  10 Farms TL1, 20 Mines TL1, 5 Factories TL1
  FarmGroup:  {ID:1, Units: [{TL:1, Qty:10}]}
  MiningGroups:
    {ID:1, Deposit:10, Units: [{TL:1, Qty:7}]}
    {ID:2, Deposit:11, Units: [{TL:1, Qty:7}]}
    {ID:3, Deposit:12, Units: [{TL:1, Qty:6}]}
  FactoryGroups: nil  ← player assigns via setup orders
```

---

## How Groups Work

Inventory records what exists on the colony. Groups record what's *assigned*.

The **farm group** is simple — one group per colony, collecting all assembled Farm units by tech level. If the template has 10 TL1 farms and 5 TL2 farms, the colony gets a single FarmGroup with two sub-groups.

**Mining groups** are one per deposit. Mine units are split as evenly as possible across all deposits on the homeworld. If you have 20 mines and 3 deposits, two groups get 7 and one gets 6. Within each group, sub-groups track tech level — if you're drawing from TL1 and TL2 mine pools, a group might end up with `[{TL:1, Qty:3}, {TL:2, Qty:4}]`.

**Factory groups** are deliberately left nil. Players assign factories to production orders — that's a game decision, not a setup default.

---

## The Post-Sprint Review

After all four tasks were done and passing, we ran a SOUSA compliance audit and code review. The audit was clean — no layering violations, `go vet` happy.

The review found **seven code smells**, ranging from a missing guard clause to test coverage gaps. All seven were fixed before closing:

{{< cards cols="1" >}}
   {{< card
      title="Missing planet-in-cluster guard"
      subtitle="AddEmpire looked up homeworld deposit IDs from cluster.Planets but silently accepted a missing planet. Added a guard that returns an error if the homeworld exists in game.Races but not in the cluster."
   >}}

   {{< card
      title="No colony template validation"
      subtitle="CreateHomeWorld validated its template; AddEmpire didn't. Added checks for valid ColonyKind, positive TechLevel, and non-negative inventory quantities — same pattern as the homeworld template validation."
   >}}

   {{< card
      title="Duplicate same-TL GroupUnit entries"
      subtitle="If the template had two Farm (or Mine) inventory rows at the same tech level, the builders created duplicate sub-groups instead of merging. Both the farm builder and buildMiningGroups now aggregate by TechLevel before building GroupUnits."
   >}}

   {{< card
      title="Template read before cheap validation"
      subtitle="ReadColonyTemplate ran before homeworld resolution, race lookup, and name scrubbing. Moved the I/O call after all the cheap checks so a bad empire name doesn't waste a file read."
   >}}

   {{< card
      title="Missing test coverage"
      subtitle="Added three tests: FactoryGroups is nil after AddEmpire, colony template read errors propagate, and homeworld planet missing from cluster returns an error."
   >}}

   {{< card
      title="TechLevel sort comparator used int subtraction"
      subtitle="Replaced int(a.TechLevel) - int(b.TechLevel) with cmp.Compare in both sort calls. Safe for realistic values, but the subtraction pattern is a known footgun."
   >}}

   {{< card
      title="Test fixture slice aliasing"
      subtitle="makeTestClusterWithDeposits stored the caller's depositIDs slice directly on the planet. Now copies the slice before assignment to prevent cross-test contamination."
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

Game setup is complete. An empire now starts with a real colony that has real production capacity. The next sprints shift from setup to gameplay — order parsing, turn execution, and report generation. That's the core loop. Once it runs, we have a playable game.

---

## Version

The project is now at **v0.8.0-alpha**. All tests pass, both entry points build, and `go vet` is clean.
