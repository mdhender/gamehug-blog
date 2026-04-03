---
title: "User Handles"
date: 2026-04-03T11:00:00
---

{{< callout type="info" >}}
   Users now have handles — short, unique identifiers assigned at registration. Users can see theirs but can't change it. Admins can. 2 commits, 435 lines changed, 7 new test files and updates across 5 existing ones.
{{< /callout >}}

## What We Built

Every user account now carries a `handle` — a lowercase string, max 16 characters, unique across the system. Handles accept letters, numbers, underscores, hyphens, and single quotes. The migration backfills existing users by extracting the first word of their name.

The admin seed account gets the handle `penguin`.

---

## The Handle Field

The `add_handle_to_users_table` migration adds a `handle` column with a unique index and backfills it from existing names. On the model side, a mutator lowercases on save. Validation enforces required, unique (case-insensitive), and the character rules.

Handle shows up in three places on the frontend: the registration form (where the user picks it), the profile settings page (read-only display), and the admin users list and detail views.

---

## Read-Only for Users, Editable by Admins

The second commit tightened the permissions. `ProfileUpdateRequest` no longer accepts `handle` — the profile settings page renders it as plain text instead of an input. Users see their handle but can't change it.

Admins get a dedicated `PATCH` route backed by `HandleUpdateRequest`, with the same validation rules (unique, case-insensitive, character whitelist). The admin user detail page gained inline editing — click the handle, change it, save.

The split keeps the profile form simple and puts handle management where it belongs: with the people running the game.

---

## Tests

`HandleValidationTest` covers the validation rules: required, uniqueness, character constraints, max length. `HandleUpdateTest` covers the admin path — successful updates, validation failures, authorization checks, and guest rejection. Registration and profile update tests were updated to include the handle field.

7 new handle-specific tests across the two test files, plus updates to `RegistrationTest`, `ProfileUpdateTest`, `UserIndexTest`, and `UserShowTest`.

---

## What's Next

Back to Layer 1. Group G wires up the `SetupReportGenerator` into controller actions — generate, lock, show, and download — so the GM can trigger reports from the browser.
