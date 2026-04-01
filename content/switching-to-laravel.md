---
title: "Switching the Web App to Laravel"
date: 2026-03-29T12:00:00
---

{{< callout type="info" >}}
   The player-facing web application is moving to Laravel. Documentation stays in Hugo. Player docs are getting reorganized to be easier to find.
{{< /callout >}}

## Why Change?

The React + Go stack worked. It got us from zero to a working dashboard with auth, orders, and reports. But as the game grows, the web application needs to do more — and the custom Go HTTP layer was turning into a framework whether we wanted it or not.

Rather than keep building our own, we're switching to one that already exists.

---

## Why Laravel

Laravel is a batteries-included PHP framework, and that's exactly the point. We were spending sprint time on plumbing — session handling, form validation, middleware wiring, CSRF protection, request lifecycle management — all problems that Laravel solved years ago. Every hour we spent reinventing that infrastructure was an hour we weren't spending on the game.

Laravel's ecosystem is mature in a way that matters for a project like this. Authentication, job queues, database migrations, mail notifications — these aren't plugins you hope are maintained. They're first-party, well-documented, and tested against each other. For a small team building a PBBG, that's the difference between shipping features and debugging glue code.

The templating story is also a win. Blade gives us server-rendered HTML with just enough dynamism. The React dashboard was fine, but it added a build step, a node dependency tree, and a client-server contract that had to stay in sync. Blade templates talk directly to the same models the backend uses. One language, one process, one deployment artifact. That's less surface area for bugs and less infrastructure to babysit.

And honestly? Laravel is *pleasant*. The documentation is excellent. The conventions are obvious. When you follow them, things just work. That's a luxury we haven't had with the bespoke Go delivery layer, where every new endpoint meant re-reading our own framework code to remember how we wired the last one.

---

## What Stays the Same

The game engine is still Go. The CLI is still Go. Turn processing, order parsing, cluster generation — none of that changes. The engine doesn't care what serves the web pages.

SQLite remains the shared datastore. Laravel talks to the same database the CLI does. The operational model — stop the server, run the CLI, restart — is unchanged.

Documentation stays in Hugo with the Hextra theme. It's working well, it's fast, and it publishes cleanly. No reason to move it.

---

## Player Docs Are Moving

The player-facing documentation has grown enough that it's getting hard to find things. We're going to spend some time reorganizing the player docs — better landing pages, clearer navigation, and a structure that makes it obvious where to look for tutorials vs. reference material vs. how-to guides.

The content isn't changing. The addresses are.

If you've bookmarked specific pages, expect some links to break during the transition. We'll do our best to keep it short.

---

## What's Next

Get the Laravel application standing up with auth and the dashboard. Once it's at parity with what the React app could do, we retire the old frontend and move forward.

---

## Version

The project is now at **v0.12.1-alpha**. The Go backend and CLI are unaffected by this change.
