---
name: winkkey-sign-in
description: >-
  Add "Sign in with Wink" (WinkKey biometric sign-in: selfie, Face ID, Touch ID or passkey) to a web app. Use when the user asks to add Wink or WinkKey sign-in or login, has a WinkKey API key (starts with wkey_) to set up, wants to protect pages behind Wink sign-in, or is debugging a WinkKey sign-in that fails, loops or signs in the wrong person.
---

# Sign in with Wink

WinkKey is Wink's hosted biometric sign-in. Your server asks WinkKey for a sign-in link,
the person confirms on winkkey.net with a selfie, Face ID, Touch ID or a passkey, WinkKey
sends them back to your app, and your server asks WinkKey who it was. There is no SDK to
install: it is two HTTPS calls with the developer's WinkKey API key. The app then keeps
its own session.

```
browser ─ GET /api/sign-in/start ─► app server ─ POST /api/merchant/auth-request ─► WinkKey
        ◄─ redirect to WinkKey link (pending cookie) ◄────────── { rid, url } ───────────┘
person confirms on winkkey.net
WinkKey ─ redirect ─► GET /api/sign-in/complete ─ GET /api/merchant/auth-request/{rid} ─► WinkKey
                      ◄────── { status: "consumed", consumedBy: { winkTag, email } } ───┘
browser ◄─ redirect to the page they wanted, session cookie set

browser ─ POST /api/sign-out ─► app server ─ POST /api/merchant/signout ─► WinkKey  (optional)
        ◄─ session cookie cleared, redirect to /signin ◄──────── 204 ────────────────┘
```

## Reference files

- `reference/api.md`: the WinkKey API (always read this).
- `reference/nextjs.md`: Next.js (App Router) code.
- `reference/express.md`: Node with Express (or any Node server; adapt the handlers).
- `reference/access-control.md`: only some people allowed, instant sign-out, invite codes.

Any other stack (Python, Ruby, PHP, Go, .NET): follow `api.md` and port the Express
handlers; the logic is the same two calls and two cookies.

## Steps

Talk to the developer in plain words; they may not know auth terms.

### 1. Look at the app

Find the framework and version (`package.json`, etc.), where server routes live, whether
there is already a sign-in or session system, and which port it runs on. If sign-in
already exists, ask whether Wink replaces it or sits next to it before changing anything.

### 2. Set up the key and settings

Settings go in the app's local env file (`.env.local` for Next.js, `.env` otherwise).
Make sure that file is in `.gitignore` before writing to it.

- `WINKKEY_API_KEY`: ask the developer for their key from the WinkKey portal. Tell them
  they can paste it into the env file themselves (`WINKKEY_API_KEY=wkey_…`) or paste it
  in the chat for you to write. Never repeat the key back, print it, log it or put it in
  code or commits. If they don't have one yet: get it on the WinkKey portal first.
- `SESSION_SECRET`: generate it yourself (`node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`)
  and write it to the env file without showing it.
- `PUBLIC_BASE_URL`: the https address people reach the app at (`http://localhost:<port>` locally, see step 4).
- `WINKKEY_BASE_URL`: leave unset unless told otherwise; defaults to `https://winkkey.net/v2`.

### 3. Build it

Follow the framework reference. The default build is:

- **Who can sign in:** anyone with a Wink ID. If the developer wants only certain people,
  use the email allowlist in `access-control.md` (ask: "Should anyone with a Wink ID get
  in, or only people you list?").
- **Sessions:** a signed, HttpOnly cookie that lasts 8 hours; no database needed. If they
  need to sign someone out remotely or keep a user list, use `access-control.md`.
- **Pages:** routes `/api/sign-in/start`, `/api/sign-in/complete`, `/api/sign-out` (POST),
  a `/signin` page with a "Sign in with Wink" button, and protection on every page that
  needs a signed-in person (a request filter sending others to `/signin?return_to=…`, plus
  a session check in each protected page or handler).
- **Sign-out:** the default clears the local session AND calls `/api/merchant/signout` so
  the next sign-in shows the face camera again. If the app is one of several the person
  signs into with Wink (so signing out of one shouldn't sign them out everywhere), remove
  the WinkKey call — the inline comment points at the three lines to delete.
- **Identity:** use `winkTag` as the person's id (stable for that person); `email` is for
  display and allowlists.

Keep the developer's existing styles: make the sign-in page look like the rest of their app.

### 4. Run it and sign in once

No tunnel is needed locally: WinkKey sends people back to `http://localhost` addresses.
Set `PUBLIC_BASE_URL` to the dev server's address (for example `http://localhost:3000`),
matching exactly what the developer opens in the browser, and restart the app. The
session cookie must not be `Secure` over plain http (the code here sets `Secure` only when
`PUBLIC_BASE_URL` is https).

Start the app, open `/signin` at that address, and ask the developer to click
"Sign in with Wink" and finish on their phone or laptop. Never do the WinkKey step for
them. When they say it worked, confirm the protected page loads and sign-out works.
If it fails, use the troubleshooting list below.

### 5. Before going live

- [ ] `WINKKEY_API_KEY` and `SESSION_SECRET` set in the host's secret settings, not in code.
- [ ] `PUBLIC_BASE_URL` set to the real https address.
- [ ] Cookies HttpOnly + SameSite=Lax + Secure (automatic when the address is https).
- [ ] Every protected page and API checks the session on the server, not only the request filter.
- [ ] `return_to` only accepts paths inside the app.

## Troubleshooting

- **Sign-in loops back to `/signin`:** the session cookie was dropped. Usually `Secure`
  cookies over plain http, or `PUBLIC_BASE_URL` not matching the address in the browser.
- **"took too long / already used":** the pending cookie is missing (different address for
  start and complete, cookies blocked) or more than 10 minutes passed.
- **Redirects go to `0.0.0.0` or an internal host:** behind a proxy or container host,
  `req.url` is the internal address. Build every redirect from `PUBLIC_BASE_URL`.
- **WinkKey returns 401/403:** wrong or missing key, or a key for a different Wink
  environment (each key belongs to one environment, e.g. test or production; a Wink ID
  made in one doesn't exist in the other).
- **404 with an empty body:** wrong host in `WINKKEY_BASE_URL`.
- **Sign-in never finishes when the link is opened from Gmail, Slack or LinkedIn on an
  iPhone:** those in-app browsers can break the WinkKey step. Ask people to open the app in
  Safari; for links you send by email or chat, detect in-app browsers and show
  "Open in Safari".
- **Same person, different account:** you keyed users on email. Use `winkTag`; the email
  on someone's Wink account can differ from the one you know.
