---
title: "Sprint 10: Dashboard API"
date: 2026-03-26T19:00:00
---

{{< callout type="info" >}}
   Sprint 10 is complete. The API server now exposes a dashboard endpoint — colony counts, planet counts, and ship counts — ready for the Sprint 11 frontend to consume.
{{< /callout >}}

## What We Built

Sprint 9 gave empires a real starting colony. Sprint 10 gives the frontend something to **ask about it**.

Before this sprint, the API had no way to summarize an empire's holdings. You could fetch orders and reports, but nothing answered "how many colonies do I have, and what kind?" Sprint 10 adds one read-only endpoint that does exactly that:

```
GET /api/:empireNo/dashboard
Authorization: Bearer <token>

200 OK
{
  "colony_count":  1,
  "colony_kinds":  [{"kind": "Open Air", "count": 1}],
  "ship_count":    0,
  "planet_count":  1,
  "planet_kinds":  [{"kind": "Terrestrial", "count": 1}]
}
```

`colony_count` is the total number of colonies. `colony_kinds` breaks that down by kind — Open Air, Orbital, Enclosed — omitting any kind with a count of zero. `planet_count` is the number of **unique** planets the empire has colonies on, and `planet_kinds` groups those by planet type. `ship_count` is always zero for now; ships haven't been implemented yet.

---

## How It's Layered

The endpoint follows the same SOUSA pattern as orders and reports — one new file per layer, nothing cross-cutting:

- **`app/dashboard_ports.go`** — defines `KindCount`, `DashboardSummary`, and the `DashboardStore` interface. No imports — these are plain Go types with JSON tags.
- **`infra/filestore/dashboard.go`** — implements `GetDashboardSummary` on `*Store`. Reads `game.json` to find the empire, reads `cluster.json` to look up colonies and planets, counts and deduplicates, returns a sorted summary.
- **`delivery/http/handlers.go`** — `GetDashboard` handler: calls the store, maps `cerr.ErrNotFound` to 404, everything else to 500.
- **`delivery/http/routes.go`** — registers the route in the protected group, adds `dashboardStore app.DashboardStore` as a new parameter.
- **`runtime/server/server.go`** — passes `fileStore` as the `dashboardStore` argument, the same concrete value already used for orders and reports.

The same `*filestore.Store` satisfies `OrderStore`, `ReportStore`, and now `DashboardStore`. The runtime wires it three times; each layer only sees the interface it needs.

---

## The Counting Logic

Two details in the implementation are worth noting.

**Kind slices omit zeros.** If an empire has no Orbital colonies, `colony_kinds` has no Orbital entry. The frontend can render whatever entries arrive and skip the rest. Both slices are sorted by kind name ascending for deterministic output.

**Planets are deduplicated.** An empire could have two colonies on the same planet — an Open Air on the surface and an Orbital in orbit. `planet_count` counts that as one planet, not two. The implementation tracks seen planet IDs while walking the colony list.

---

## Tests

Five tests cover the implementation in `filestore/dashboard_test.go`. Each writes minimal `game.json` and `cluster.json` fixtures directly with `os.WriteFile` — no dependency on the store's own write path:

- **`TestGetDashboardSummary_OneColony`** — happy path: one colony, correct counts and kind strings
- **`TestGetDashboardSummary_MultipleKinds`** — two colonies of different kinds on different planet types; verifies both kind slices have two entries sorted correctly
- **`TestGetDashboardSummary_DeduplicatesPlanets`** — two colonies on the same planet; asserts `PlanetCount == 1`
- **`TestGetDashboardSummary_EmpireNotFound`** — empire ID not in `game.json`; asserts error wraps `cerr.ErrNotFound`
- **`TestGetDashboardSummary_NoColonies`** — empire exists but has no colonies; asserts all counts are zero and kind slices are empty

---

## What's Not Here Yet

- **No frontend** — the dashboard cards that consume this endpoint are Sprint 11.
- **Ships** — `ship_count` is always 0 until ships are implemented.
- **No order parsing** — orders are stored but not interpreted.
- **No turn processing** — the engine pipeline hasn't started.

---

## What's Next

Sprint 11 builds the frontend dashboard that calls this endpoint — the player-facing summary cards showing colony and planet counts. After that, the setup sequence is fully visible end-to-end: create an empire, open the dashboard, see your starting colony reflected back.

---

## Version

The project is now at **v0.10.0-alpha**. All tests pass, both entry points build, and `go vet` is clean.
