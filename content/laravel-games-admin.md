---
title: "Laravel: Games Admin, Member Management, and Dashboard"
date: 2026-03-29T23:00:00
---

{{< callout type="info" >}}
   The Laravel web app now has a full games admin section. Admins and GMs can create, browse, and manage games. Members can be added, deactivated, and reactivated without losing their game history.
{{< /callout >}}

## What We Built

Today's session covered four areas: app branding, a games list page, a game detail page with member management, and a dashboard card showing active game status.

---

## Branding

The app shed its "Laravel Starter Kit" identity. The sidebar now shows the EC hexagonal icon, the favicon and apple-touch-icon are updated, and the welcome page has been rewritten to describe Epimethean Challenge rather than a generic starter template.

Small change, but it matters — every screenshot from here forward will look like the product, not the scaffolding.

---

## Games List

A new **Games** section appears in the sidebar for admins and GMs. Clicking it opens `/games`, which shows a table of games with columns for name, status, GM count, and player count.

Admins see every game. GMs only see games where they hold an active GM role. That scoping happens at the query level — no post-filtering.

The table has a client-side toggle between active and inactive games. Inactive games are dimmed and marked with a badge but stay in the same table structure so the layout doesn't shift.

Admins get a create form above the table. The form suppresses password manager overlays (`data-1p-ignore`) since the game name field was attracting 1Password's inline icon. Admins also get a delete button per row with a confirmation dialog before any destructive action fires.

---

## Game Detail Page

Each game name in the list links to a detail page at `/games/{id}`. The page has three sections.

**Edit form** — name and active/inactive status. Available to admins and the game's own GMs. The status toggle is a checkbox that controls whether the game appears in the default active view. A GM who makes a game inactive doesn't lose access to it; they can reactivate it from the same form.

**Active members table** — lists current GMs and players with their role and a Deactivate button. Admins can deactivate anyone, including GMs. GMs can only deactivate players on their own games — they can't remove a fellow GM.

**Add member form** — a two-field form: user select and role select. The user dropdown only shows people who aren't already in the game (active or inactive). Admin accounts are excluded entirely — they have implicit GM-level access to everything and shouldn't occupy a roster slot. The GM role option in the dropdown is hidden for non-admins.

---

## Soft-Delete for Members

Removing a member doesn't delete the pivot row. It sets `is_active = false` on the `game_user` record. The user stays in the database with their role intact; they just stop appearing in the active member list.

This matters because a departing player's ships and colonies don't vanish when they leave — those assets become independent and drift for a few turns. The game can continue without them. If the player changes their mind, an admin or GM can reactivate them from the **Inactive members** section that appears below the active table whenever inactive records exist.

A deactivated member can't be added back through the add form — they're excluded from the available users dropdown regardless of status. Reactivation goes through the dedicated Reactivate button, which restores their original role.

The same rules apply to GMs: a GM who steps down becomes inactive. They can be reinstated by an admin. A GM who is deactivated can't rejoin as a player — their role is preserved and they'd need explicit reactivation.

---

## Dashboard Card

The dashboard gained its first real content: a card showing the count of active games and a link to the current game — meaning the active game most recently updated.

For admins the count and current game span all games in the system. For other users they're scoped to games the user belongs to. The card links directly to the game detail page so jumping into the current game is one click from the dashboard.

---

## Authorization

The `GamePolicy` covers the full CRUD surface. Admins pass every check. GMs can view and update their own games but can't create new ones or manage GM membership. The `GameMemberController` enforces an additional layer: even a GM who passes the `update` policy check is blocked from deactivating or reactivating other GMs. That's an admin-only action.

All policy checks use `Gate::authorize()` — consistent with the rest of the application.

---

## Tests

116 tests, all green.

- `GameControllerTest` — 14 tests covering index scoping (admin vs. GM views), store, update, and destroy authorization
- `GameMemberControllerTest` — 16 tests covering show data shape, add member validation, deactivation, reactivation, and the GM-management restriction
- `DashboardTest` — 6 tests covering the active game count and current game link for both admin and non-admin users

---

## What's Next

The games admin is feature-complete for the current scope. The next area to build out is the empire section — giving players a view of their own empire, starting with the empire detail page that member rows on the game page will eventually link to.
