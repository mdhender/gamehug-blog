---
title: "Sprint 2: The Frontend Arrives"
date: 2026-03-24T16:59:00
---

{{< callout type="info" >}}
   Sprint 2 is complete. Players can now log in, submit orders, and read reports through a web UI.
{{< /callout >}}

## What We Built

Sprint 2 delivered the **web frontend** — the React UI that connects players to the API server built in Sprint 1.

Here's what it can do right now:

- **Authenticated dashboard** — if you have a valid session, you see your empire name and navigation to your orders and reports.
- **"Cluster Under Construction" gate** — if you're not authenticated, you get a clear holding page instead of a broken or empty app.
- **Magic-link sign-in** — the frontend handles the `?magic=` token in the URL, exchanges it for a JWT, and strips the parameter from the address bar cleanly.
- **Orders editor** — a full-page monospace text area for writing and submitting your turn orders. If no orders exist yet, it starts blank. If they do, they load in ready to edit.
- **Reports list** — a list of your available turn reports by year and quarter, each linking to the full report.
- **Report viewer** — each report renders in a monospace pre block, exactly as the server sent it.

The loop is now closed end-to-end for a human player: log in, write orders, submit, come back later and read your report.

---

## Backend Work That Made It Possible

Before the frontend could launch, the API needed two small additions.

**`GET /api/me`** was added first. This endpoint tells the frontend whether the current token is valid and which empire it belongs to. It returns `authenticated: true/false`, the empire number, and a display name. Critically, it never returns an error — unauthenticated requests get `empire: 0, authenticated: false, name: "guest"`. The frontend uses this as its single source of truth on load.

**Report links in `GET /api/:empireNo/reports`** were added next. Originally the reports list returned just the year and quarter. The frontend would have had to construct the URL itself — which means the frontend would need to know the URL structure. Instead, each report item now includes a pre-built `link` field. The frontend follows the link; it doesn't need to know how it's formed. This is a small thing that avoids a coupling that tends to cause headaches later.

---

## How the Frontend Is Structured

The React app (`apps/web/`) follows the same principle of keeping things where they belong:

- **`lib/auth.ts`** — JWT storage and the `Authorization` header. One place, used everywhere.
- **`lib/api.ts`** — all fetch calls to the backend. Pages don't call `fetch` directly; they call typed functions here.
- **`lib/types.ts`** — shared TypeScript types for API responses.
- **`pages/`** — one file per page: `DashboardPage`, `OrdersPage`, `ReportsPage`, `ReportPage`.
- **`components/AppShell.tsx`** — the nav sidebar and header wrapper used by all authenticated pages.

The `App.tsx` root component handles the auth lifecycle: check for a magic link in the URL on load, call `/api/me`, then decide what to render. Everything else is passed down as props or handled in the relevant page component.

---

## A Few Details Worth Noting

**Orders are raw text.** The `POST /api/:empireNo/orders` endpoint reads the request body as plain bytes — no JSON wrapping. The frontend sends a `text/plain` body to match. This keeps orders human-readable and simple to edit outside the UI if needed.

**404 on orders isn't an error.** When a player has never submitted orders, the server returns 404. The frontend treats this as "no orders yet" and opens a blank editor rather than showing an error message.

**The sidebar shows text.** The original AppShell design used a narrow icon-only sidebar on desktop — a pattern that works well when there are many nav items with universally recognizable icons. With only three items (Dashboard, Orders, Reports) and icons that don't obviously represent those concepts, we widened the sidebar and added labels.

**Build artifacts are committed.** The `dist/` folder is checked in so the API server can serve the frontend directly without a separate build step in production. The `tsconfig.tsbuildinfo` incremental build cache is now correctly gitignored.

---

## What's Not Here Yet

- **No game engine** — orders are stored, but nothing processes them yet.
- **No database** — still file-backed. SQLite is coming.
- **No turn processing** — the CLI pipeline hasn't started.
- **No admin UI** — user management exists in the codebase from earlier work but isn't wired into the new navigation.

---

## What's Next

With a working frontend and a working API server, the next sprint will likely focus on:

- The **CLI game engine** — parsing orders, running the turn, generating reports
- **SQLite migration** — replacing flat-file storage with a real datastore
- **Turn processing pipeline** — closing the full loop so submitted orders actually become reports

Playtesting is the goal. Every sprint gets us closer.

---

## Version

The server is at **v0.2.3-alpha**. The frontend builds clean, serves from `dist/`, and talks to the API on the same origin.
