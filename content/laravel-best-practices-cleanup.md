---
title: "Laravel: Best-Practices Review and Cleanup"
date: 2026-03-30T09:00:00
---

{{< callout type="info" >}}
   A full codebase review turned up security gaps, N+1 queries, missing indexes, data-integrity risks, and a pile of low-severity code smells. Every finding is now resolved.
{{< /callout >}}

## What Happened

After shipping the games admin section, we ran a structured best-practices review before adding more features. The findings were catalogued in a `BURNDOWN.md` and resolved one at a time. This post covers what was found and why each fix matters.

---

## Security Fixes

### Bypassed Validation in SecurityController

`SecurityController::update()` was reading `$request->password` directly instead of `$request->validated('password')`. The difference is not academic — `$request->password` returns the raw input regardless of whether it passed validation. Switching to `$request->validated('password')` ensures the controller only acts on data that has been through the validation rules.

### Admin Route Outside the Admin Middleware Group

The `admin/users/{user}` show route was sitting outside the `admin` middleware group in `routes/admin.php`. That meant any authenticated user could reach the admin user detail page by guessing their own profile URL. Moving the route inside the group gates it properly — the admin middleware now rejects the request before it reaches the controller.

---

## N+1 Query Fixes

Three separate N+1 problems were found and fixed.

### is_gm on Every Inertia Request

The `User` model had `is_gm` in its `$appends` array, backed by a method that queries the `game_user` pivot. Because it was appended, it ran on every serialisation — including the shared Inertia props that fire on every page load. For a logged-in GM, this meant a hidden pivot query on every request, regardless of what page they were viewing.

The fix removes `is_gm` from `$appends` and replaces the per-request call in `HandleInertiaRequests` with `loadExists()`:

```php
$user->loadExists(['games as is_gm' => fn ($q) => $q->wherePivot('is_gm', true)]);
```

`loadExists()` sets the attribute on the model from a single EXISTS subquery. The `boolean` cast on `is_gm` was kept — it ensures the `0`/`1` the EXISTS query returns serialises as a proper `true`/`false` in the Inertia payload rather than an integer.

### Admin Users List

`UserController` was loading the full user list and relying on the now-removed `$appends` accessor to populate `is_gm` per row. That meant one pivot query per user in the list. The controller now uses a `withExists()` subquery so the entire list is fetched with `is_gm` populated in a single pass. The controller also switched from `AuthorizesRequests` / `$this->authorize()` to the `Gate::authorize()` pattern used everywhere else in the application.

### preventLazyLoading

To catch future N+1 regressions automatically, `AppServiceProvider` now calls:

```php
Model::preventLazyLoading(! app()->isProduction());
```

In local and staging environments any unintended lazy load will throw an exception at the point it happens rather than silently burning extra queries in production.

---

## Data Integrity

### Transaction Around User Registration

`CreateNewUser` creates a `User` record and then calls `markAsRegistered()` on the invitation. Previously those two operations were independent — if `markAsRegistered()` threw, a user would exist in the database with a still-valid invitation token. Wrapping both in a `DB::transaction()` ensures they succeed or fail together. A partial registration is no longer possible.

---

## Performance Improvements

### SELECT * on Game Members Eager Load

`GameController::show()` called `$game->load('users')` with no column constraint, fetching every column on every member. The page only uses `id`, `name`, and `email`. The load is now `$game->load('users:id,name,email')`, which generates a constrained `SELECT` and reduces the data transferred from the database.

### Missing Index on game_user.user_id

The `game_user` pivot table has a composite primary key of `(game_id, user_id)`. That index covers forward lookups — finding all members of a game. Reverse lookups — finding all games a user belongs to — required a full table scan because there was no index starting with `user_id`. A new standalone index on `user_id` eliminates that scan. A migration adds it to existing deployments.

### Invitation Emails Were Blocking

`InvitationController::store()` and `resend()` were calling `Mail::queue()`, which implies the mailable should implement `ShouldQueue` — but `InvitationMail` didn't. Laravel falls back to synchronous delivery when the contract is missing, so every invitation was blocking the HTTP response on SMTP. Adding `ShouldQueue` to the mailable makes the queue dispatch actually queue.

### Unreliable Distinct Count

`DashboardController` was computing the count of distinct users across games with `->distinct('user_id')->count()`. That method does not reliably generate `COUNT(DISTINCT user_id)` across all databases — it can silently produce a plain `COUNT(*)` depending on the driver. The fix is explicit:

```php
->count(DB::raw('DISTINCT user_id'))
```

---

## Authorization and Data Leakage

### Admin Stats Exposed to All Users

Non-admin users were receiving `totalActiveUsers`, `loggedInUsersCount`, and `pendingInvitesCount` as `null` values in the Inertia props. The keys were present in the payload even when empty. Those props are now omitted entirely for non-admins — the keys simply don't appear, which matches the intent that only admins see admin statistics.

### Available Users List Exposed to Players

`GameController::show()` was sending the full list of non-member user names and emails to every visitor of the game detail page. Players can't add members, so they had no use for this data — but they were receiving it anyway. The list is now gated behind the `update` policy check:

```php
'availableUsers' => Gate::allows('update', $game)
    ? User::whereNotIn(...)->get(['id', 'name', 'email'])
    : [],
```

Players receive an empty array. Admins and GMs receive the full list.

### Inactive Users Passing Policy Checks

`GamePolicy::viewAny()` and `view()` were not checking `is_active` on the `game_user` pivot. An inactive GM could still pass `viewAny`, and an inactive member could still pass `view`. Both methods now require the matching pivot row to have `is_active = true` before granting access.

---

## Code Quality

### Form Requests for GameController and GameMemberController

Inline `$request->validate([...])` blocks in `GameController::store()`, `GameController::update()`, and `GameMemberController::store()` were replaced with dedicated Form Request classes: `StoreGameRequest`, `UpdateGameRequest`, and `StoreGameMemberRequest`. This matches the validation pattern used by every other controller in the application.

### Duplicated GM Pivot Query in GamePolicy

`GamePolicy::viewAny()` was doing a manual `whereHas('games', fn ($q) => $q->wherePivot('is_gm', true))` query instead of calling the `isGm()` method already on the `User` model. The duplicate logic is removed; the policy now calls `$user->isGm()` like everything else.

### Redundant Hash::make in CreateAdminUser

The `CreateAdminUser` Artisan command was calling `Hash::make()` on the password before passing it to `User::create()`. The `User` model already declares a `hashed` cast on the `password` attribute, so the model hashes the value automatically. The manual `Hash::make()` was double-hashing. It's removed.

### Wrong PHPDoc on UserFactory::admin()

The `admin()` factory state had a docblock that said "email should be unverified" — copy-pasted from the wrong method. It now correctly says the method returns a factory for a user with admin privileges.

---

## Pivot Table

### Timestamps on game_user

The `game_user` pivot table was missing `created_at` and `updated_at` columns. Without them there's no way to know when a user joined a game or when their membership last changed. A migration adds both columns (nullable for existing rows), and the `belongsToMany` definitions in `Game` and `User` now call `withTimestamps()` so new pivot rows are timestamped automatically.

---

## Developer Experience

### LazilyRefreshDatabase

All test classes were using `RefreshDatabase`, which wraps every test in a transaction and rolls it back regardless of whether the test actually touches the database. `LazilyRefreshDatabase` only migrates when the first database call is made. Tests that don't touch the database at all skip the overhead entirely. All twelve affected test files are updated.

`CreateAdminUserTest` also had two smaller issues fixed while the trait was being updated: test methods used the old `test_*` naming convention instead of `#[Test]` attributes, and `assertEquals` was replaced with the strict `assertSame`.

---

## Infrastructure

### Production Guard on AdminSeeder

`AdminSeeder` could previously be run in production with `db:seed`. It now checks `app()->isProduction()` at the top and exits early with a warning if the environment is production. Admin accounts in production should be created through the `CreateAdminUser` Artisan command, not seeded in bulk.

### Deployment Script

The deployment steps that were previously prose in `DEPLOY.md` have been extracted into a `deploy/server-scripts/deploy.sh` shell script. The markdown file now references the script rather than duplicating the steps.

---

## Tests

All 116 tests pass. The fixes above are covered by updates to `GameControllerTest`, `GameMemberControllerTest`, `DashboardTest`, and `Admin/UserShowTest`.
