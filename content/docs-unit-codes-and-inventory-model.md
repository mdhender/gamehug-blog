---
title: "Docs: Unit Codes, Inventory Model, and Site Redesign"
date: 2026-04-03T23:59:00
---

{{< callout type="info" >}}
   The documentation site gained a full unit-code reference, a quick glossary, and a redesigned landing page. Behind the scenes, four developer-facing pages formalize the inventory model — assembly states, installed vs. cargo, and the invariants the engine must enforce. 4 commits, 516 new lines across 15 files.
{{< /callout >}}

## What Players See

The docs site now has a **Unit Codes** reference page listing every item in the game — production units, ship systems, weapons, vehicles, consumables, population classes, and colony types — with mass, fuel cost, output, and crew requirements in sortable tables. Players can look up `FCT-3` or `FOOD` without digging through the 1978 manual.

A new **Quick Glossary** defines the terms that appear in turn reports and order syntax: assembled, unassembled, installed inventory, cargo inventory, SC, and the rest. Short definitions, no fluff.

Both pages are linked from a new top-level **Reference** section in the navigation bar, sitting alongside Players, Referees, Developers, and History.

The landing page and Players index were rebuilt using Hextra's `hero-headline`, `hero-subtitle`, `hero-button`, and `feature-grid` shortcodes, replacing the plain card grids. The Players index also picked up a direct link to the unit-codes page.

---

## Developer Documentation

Four new pages under `developers/` formalize the inventory model that the engine implements.

### Assembly-Required Units

Explains why Gamehub prefers `assembly_required` over the 1978 manual's phrase "operational unit." The manual uses "operational" and "assembled" interchangeably — the new docs pin down the mapping: `assembly_required = true` means the unit can exist in either an assembled (usable) or unassembled (stored, needs labor) state. Units with `assembly_required = false` are always considered assembled.

### Ship and Colony Inventory

Documents the rule that units only exist inside a ship or colony inventory — there is no free-floating unit pool. Separates two concepts that are easy to conflate: **installed inventory** (units that constitute the SC itself — hull structure, drives, working factories) vs. **cargo inventory** (material being stored or transported). A space drive can appear as either, and the distinction affects how the engine accounts for it.

### Invariants

Three inventory-state invariants for implementation:

1. Non-assembly units are always assembled — they never appear in an unassembled bucket.
2. Factory output enters inventory differently depending on whether the produced unit has `assembly_required = true` (enters as unassembled) or `false` (enters as a non-assembly item).
3. Inventory role (installed vs. cargo) is orthogonal to assembly state.

### Terminology

A canonical term list mapping Gamehub's developer vocabulary to the 1978 manual's phrasing. Covers assembly terms (`assembled`, `unassembled`, `stored non-assembly`), inventory roles (`installed inventory`, `cargo inventory`), and quantity fields (`quantity_assembled`, `quantity_disassembled`).

---

## Site Structure

The Hugo config gained a `defaultContentLanguage` block, reweighted the main menu to put Players first, and added the Reference section and a GitHub link to the nav bar. The players section picked up `cascade: type: docs` so child pages inherit the docs layout, and a new `players/reference/` folder was scaffolded for future player-facing reference material.

---

## What's Next

The unit-codes page and inventory model docs give the engine implementation a single source of truth to code against. Next up is wiring the inventory model into the turn-processing pipeline — factories producing units, assembly orders consuming labor, and the report generator reflecting the correct assembly states.
