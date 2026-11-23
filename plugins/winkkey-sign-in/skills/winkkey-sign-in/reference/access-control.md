# Access control (optional)

The basic build lets anyone with a Wink ID sign in and keeps the session in a signed
cookie. Add these only when the developer asks for them.

## A. Only certain people: email allowlist

Setting `ALLOWED_EMAILS=ana@acme.com,ben@acme.com` (or `ALLOWED_DOMAINS=acme.com`). In the
complete handler, after the `consumed` check:

```ts
const allowed = (email?: string | null) => {
  const e = (email ?? "").trim().toLowerCase();
  const emails = (process.env.ALLOWED_EMAILS ?? "").toLowerCase().split(",").map((x) => x.trim()).filter(Boolean);
  const domains = (process.env.ALLOWED_DOMAINS ?? "").toLowerCase().split(",").map((x) => x.trim()).filter(Boolean);
  if (!emails.length && !domains.length) return true;
  return !!e && (emails.includes(e) || domains.some((d) => e.endsWith("@" + d)));
};
if (!allowed(consumedBy.email)) → redirect to /signin?signin_error=no_access
```

The email is the one on the person's Wink account. If someone's differs, use B.

## B. A user list in the database, with instant sign-out

Use when the app needs roles, an admin screen, or to cut someone off immediately (a signed
cookie stays valid until it expires).

Tables:

- `users`: id, email (unique, lower-case), name, role, active, `wink_tag` (set at first
  sign-in), last_sign_in_at.
- `sessions`: token_hash (SHA-256 of a random token; primary key), user_id, method,
  created_at, expires_at.
- `sign_in_attempts` (optional; replaces the pending cookie's contents): id (random,
  stored in the pending cookie), rid, return_to, expires_at. Delete the row when reading it
  so each attempt works once.

Complete handler, after `consumed`:

1. Find the user by `wink_tag`; if none, by email (lower-case).
2. Not found or not active → `no_access`.
3. Save `wink_tag` on first sign-in and update `last_sign_in_at`.
4. Make a random token (32 bytes), store its SHA-256 in `sessions` with an expiry, put the
   token itself in the session cookie.

Every request: hash the cookie token, look up an unexpired session joined to an active
user. Turning a user off or deleting their sessions signs them out at once. Sign-out
deletes the session row.

Pages only admins (or some roles) may open: check in the request filter as well as the
page. In Next.js a redirect from a layout still lets the page render and send its data in
the response, so the layout check alone leaks.

## C. Invite codes (people without a Wink ID yet)

For apps where the owner adds people by hand. The owner makes a one-time code for a person
on the user list and texts it; the person enters it on `/signin/invite`; whichever Wink ID
completes the invite is linked to that person.

- Table `invite_codes`: id, user_id, code_hash (SHA-256 of the normalised code, unique),
  created_at, expires_at (e.g. 14 days), used_at. Show the code once; making a new code
  deletes the person's unused one.
- Code: 12 characters from `ABCDEFGHJKMNPQRSTUVWXYZ23456789` (no 0/O/1/I/L), shown as
  `XXXX-XXXX-XXXX`; compare after removing dashes and spaces, upper-cased.
- Invite page: check the code with a server action that returns only the first name (or a
  generic "not valid" after a short pause). Then two buttons that POST the code to
  `/api/invite/start`: "Set up my Wink ID" (`kind: "enroll"`) and "I already have a Wink ID"
  (normal sign-in).
- `/api/invite/start`: re-check the code, create the auth-request (with `kind: "enroll"` when
  asked, and a longer `ttlSeconds`, e.g. 2 days, since enrolment takes time), remember the
  invite id with the `rid`, redirect to WinkKey.
- Complete handler for an invite: invite missing, used or expired → error; the `winkTag`
  already belongs to a different user → error; otherwise use the invited user, mark the code
  used, save the `winkTag`, open the session.
- Keep `/signin/invite` and `/api/invite/` public in the request filter.
