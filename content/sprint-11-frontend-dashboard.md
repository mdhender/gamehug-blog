---
title: "Sprint 11: Frontend Dashboard Enhancement"
date: 2026-03-26T20:00:00
---

{{< callout type="info" >}}
   Sprint 11 is complete. The dashboard now shows live colony, ship, and planet summary cards, and the sidebar has three new destinations: Colonies, Ships, and Star List.
{{< /callout >}}

## What We Built

Sprint 10 gave us a dashboard API endpoint that knows how many colonies and planets an empire has. Sprint 11 makes that visible — replacing the two-button placeholder dashboard with real data cards and connecting the sidebar to three new pages.

When you log in now, the dashboard loads `GET /api/:empireNo/dashboard` on mount and renders a three-card grid:

- **Colonies** — total colony count with a kind breakdown (Open Air, Orbital, Enclosed) and a "View details →" link to the Colonies page
- **Ships** — always 0 for now, with a "View details →" link to the Ships page
- **Planets** — unique planet count with a planet-type breakdown (no detail page yet — that comes later)

While the fetch is in flight you get animated skeleton placeholders. If it fails you get a brief inline error message. Either way, the Orders and Reports buttons below the cards stay visible and functional.

---

## Three New Pages

The sidebar grew three new items.

**Colonies** (BuildingOffice2Icon) navigates to a page that fetches the same dashboard endpoint and renders a table — kind in the left column, count in the right. If the empire has no colonies yet it shows "No colonies." If the fetch fails it shows the error inline.

**Ships** (RocketLaunchIcon) is a placeholder for now:

> No ships. (The assemble ship order has not been implemented.)

**Star List** (MapIcon) is also a placeholder:

> No stars. (The probe order has not been implemented.)

Both placeholders are intentional. The sidebar shape is now correct; the pages will fill in as those features land.

---

## How It Fits Together

The app uses no router — page state is a `Page` union type in `App.tsx` managed with `useState`. Adding three new values (`"colonies"`, `"ships"`, `"star-list"`) to the union was the only structural change. New nav items set page state on click; `renderPage()` maps each value to its component. Nothing novel — exactly the pattern already used for Orders, Reports, and the admin page.

Data fetching follows the same `useEffect` + `useState` pattern as `OrdersPage`: one state variable each for `loading`, `error`, and `data`. No third-party data-fetching library.

On the type side, `lib/types.ts` gained `KindCount` and `DashboardSummary`, and `lib/api.ts` gained `fetchDashboard`. Both follow existing patterns — the same `apiFetch<T>` wrapper and interface style already used everywhere else in the client.

---

## What's Not Here Yet

- **No ships** — `ship_count` is still 0 until the assemble ship order is implemented.
- **No star probing** — the probe order hasn't been built yet.
- **No planets page** — the Planets card on the dashboard has no detail page; that's a future sprint.
- **No turn processing** — orders are stored but not yet interpreted.

---

## What's Next

The scaffolding is in place. The next passes will start hooking up real game actions — order parsing, turn processing, and the first visible effects of player decisions.

---

## Version

The project is now at **v0.11.0-alpha**. The build is green, all backend tests pass, and `go vet` is clean.
