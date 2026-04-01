---
title: "Sprint 1: The API Server Lives"
date: 2026-03-24T14:43:00
---

{{< callout type="info" >}}
   Sprint 1 is complete. We have a working API server.
{{< /callout >}}

## What We Built

The first sprint delivered a **working API server** — the backbone that will eventually connect players to the game engine.

Here's what it can do right now:

- **Magic-link authentication** — no passwords, no account creation. You get a unique link, you log in.
- **JWT authorization** — once logged in, your empire identity is baked into a token. Every request proves who you are.
- **Order submission** — upload your turn orders via the API. They're stored on disk, ready for the game engine.
- **Report retrieval** — pull your turn reports (when they exist) as JSON.
- **Graceful shutdown** — the server shuts down cleanly on SIGINT, a timer, or a remote shutdown command.

It's not flashy. There's no UI yet. But the loop is there: authenticate, submit orders, get reports.

---

## How It Works

If you're curious about the architecture, the server follows a strict layered design called **SOUSA**. Dependencies flow inward only:

```
domain ← app ← infra / delivery ← runtime
```

In practice, this means:

- **Game rules** live in `domain` and know nothing about HTTP or databases.
- **Use cases** live in `app` and talk through interfaces, never concrete types.
- **Adapters** (file storage, JWT signing, magic links) live in `infra` and implement those interfaces.
- **HTTP handlers** live in `delivery` — they parse requests, call use cases, and format responses. That's it.
- **Wiring** happens in `runtime` — the only place that knows about all the concrete pieces.

This discipline matters because the game engine and the API server will eventually share the same core logic. If the HTTP layer leaks into the game rules, we're in trouble later.

---

## The Review That Caught 13 Issues

After finishing all nine tasks, we did a SOUSA compliance and code-smell review. It found **13 issues** — and we fixed every one before closing the sprint.

The highlights:

{{< cards cols="1" >}}
   {{< card
      title="Peer layers were crossing"
      subtitle="The HTTP handlers were importing the auth adapter directly. Fixed by injecting a function from the runtime layer instead."
   >}}

   {{< card
      title="Wiring was in the wrong place"
      subtitle="The main.go was constructing infrastructure adapters. That's runtime's job. Moved it where it belongs."
   >}}

   {{< card
      title="Unbounded request body"
      subtitle="Order uploads had no size limit. Now capped at 1 MiB — plenty for text orders, not enough for abuse."
   >}}

   {{< card
      title="Secrets in git"
      subtitle="A .env file with development secrets was tracked. Renamed and gitignored."
   >}}
{{< /cards >}}

The full list is in the sprint record for anyone who wants the details.

---

## What's Not Here Yet

To be clear about what this sprint did *not* build:

- No game engine — there's nothing to process orders yet.
- No web UI — it's API-only for now.
- No database — everything is file-backed. SQLite comes later.
- No turn processing — the CLI pipeline hasn't been started.

This is the foundation. The API server is one piece of the loop: **submit orders → process turn → generate reports → read reports**. Sprint 1 handles the first and last steps.

---

## What's Next

The next sprint will likely focus on one of:

- The **CLI game engine** — parsing orders, executing turns, generating reports
- The **web frontend** — a simple React UI for submitting orders and reading reports
- **SQLite migration** — moving from flat files to a real datastore

We'll pick based on what unblocks playtesting fastest.

---

## Version

The server is now at **v0.2.0-alpha**. It builds, it runs, it passes its tests. That's the bar, and it clears it.
