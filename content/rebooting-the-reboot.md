---
title: "Rebooting the Reboot"
date: 2026-03-23
---

{{< callout type="info" >}}
   Epimethean Challenge is back—again.
   This time, we’re building it to last.
{{< /callout >}}

## Epimethean Challenge, Attempt #2

The first attempt at rebooting **Epimethean Challenge** fizzled out.

Not because the idea was wrong.  
Not because the engine was impossible.

It failed because we tried to do *everything at once*:

- Build the web site
- Build the game engine
- Write the documentation
- Run a live game

…all while preparing for a move to Panama.

That was too much.

---

## What This Reboot Is

This is a **modernized PBBG engine** with a **faithful recreation of the 1978 Epimethean Challenge** as the starting point.

Not a redesign.  
Not a reimagining.

A reconstruction—with better tools.

---

## Back to 1978

A huge thanks to **James Columbo**, who has kindly granted permission to use the original **1978 rule book**.

You’ll find it in the **History** section.

{{< callout type="warning" >}}
Please do not repost the rulebook without his permission.
{{< /callout >}}

This rulebook is our **temporary baseline**.

The original game evolved quickly after 1978, and we already have the 1980s rules queued up.  
But first, we build something that *works*.

---

## What’s Different This Time

The philosophy is simple:

> **Always have a working product.**

{{< cards cols="1" >}}
   {{< card
      title="Smaller Steps"
      subtitle="No big leaps. Each piece gets built, tested, and stabilized before moving on."
   >}}

   {{< card
      title="Less Ambition (At First)"
      subtitle="We are deliberately limiting scope to avoid collapse."
   >}}

   {{< card
      title="Honest About Problems"
      subtitle="This will be rough in places. That’s expected—and visible."
   >}}

   {{< card
      title="Documentation Matters"
      subtitle="The system is defined, not guessed."
   >}}
{{< /cards >}}

---

## The Target Audience (For Now)

This isn’t ready for casual players.

Right now, this is for:

- Returning EC / Olympia-style PBEM players
- People comfortable with **alpha-quality systems**
- Folks who don’t mind sharp edges

There’s already a warning on the site—and it’s there for a reason.

---

## The Stack (Simple and Boring on Purpose)

### Frontend
- React + Vite + TailwindCSS

> I still like Ember—but it’s not the right tool for this job.

### Web Server
- Go
- SQLite3-backed

Features:
- Magic-link authentication (no passwords)
- Token-based access for uploads and reports

### Game Engine
- Go (CLI-driven)

Initial pipeline:

1. Generate cluster
2. Parse orders
3. Execute orders
4. Generate turn reports (JSON + text)

That’s it. No shortcuts.

---

## What “v1” Looks Like

Version 1 is not “feature complete.”  
It’s **functionally complete**.

- Order entry web page
- Turn report download page (text + JSON)
- Authentication working (magic links)
- Game engine runs via CLI
- Shared SQLite3 datastore between engine and web

If that loop works, we have a game.

---

## What Comes Next

- Deterministic cluster generation
- Strict order parsing
- First playable turns (internally)
- Report generation pipeline

Then we iterate.

---

## Contributors (Carefully)

We’re not ready for players yet.

But we *are* interested in:

- Testers
- Documentation reviewers

If you enjoy digging into rules, edge cases, and inconsistencies—this is your moment.

---

## Updates

No fixed schedule.

Progress will be posted when it’s real—primarily on Discord.

---

## Why Do This?

Because getting an old game like this running again is a challenge worth taking on.

---

## Final Thoughts

This isn’t just a reboot.

It’s a reset:

- Build smaller
- Ship earlier
- Keep it running

If you’ve been here before—welcome back.  
If you’re new—this is the ground floor.