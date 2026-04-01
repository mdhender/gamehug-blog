---
title: "Cleaner Invitation Registration"
date: 2026-03-31T21:56:00
---

{{< callout type="info" >}}
   The invitation flow now makes the email constraint obvious — both in the invite itself and on the registration form.
{{< /callout >}}

## What We Shipped

Invitations in GameHub are scoped to an email address. That's a deliberate security choice — the invite token is issued for a specific recipient, and accepting it with a different address shouldn't be possible. The problem was that nowhere in the flow did we actually say that. Players received an email, clicked the link, and landed on a blank registration form with no indication that the email field mattered or was locked.

Two things changed.

---

## The Invitation Email

The invitation email now includes an explicit notice:

> Please note: this invitation is tied to this email address. You must register using **your@email.com** to accept it.

Before this, the email said "you've been invited" and showed a button. That was it. Players had no way to know — without trying and failing — that the address mattered. The notice sets the expectation before they click.

---

## The Registration Form

When a player follows a valid invitation link, the backend now looks up the invitation by token and passes the associated email to the registration page as `invitationEmail`. The `FortifyServiceProvider` resolves the token against the `Invitation` model using the existing `valid()` scope before rendering the page:

```php
Fortify::registerView(function (Request $request) {
    $token = $request->query('token', '');
    $invitation = Invitation::query()->valid()->where('token', $token)->first();

    return Inertia::render('auth/register', [
        'invitationToken' => $token,
        'invitationEmail' => $invitation?->email,
    ]);
});
```

On the frontend, if `invitationEmail` is present, the email field is pre-filled, marked `readOnly`, and styled with `bg-muted` to signal it isn't editable. A short message underneath confirms why:

> This email is linked to your invitation and cannot be changed.

If there's no invitation email — someone navigating directly to `/register` without a token, or with an expired one — the field behaves exactly as before. The `invitationEmail` prop is optional and the form degrades cleanly.

---

## Tests

4 new test cases cover the additions:

- The invitation email includes the recipient address in the notice text.
- The registration page receives `invitationEmail` when the token matches a valid invitation.
- The registration page receives `null` for `invitationEmail` when the token is missing or invalid.
- Registration still works end-to-end with a locked email field.

---

## What's Next

Member management controls — promoting players to GM and removing members from a hub — are the next area in progress.
