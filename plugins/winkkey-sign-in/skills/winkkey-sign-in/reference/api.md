# WinkKey API (sign-in)

Base URL: `https://winkkey.net/v2` (override with `WINKKEY_BASE_URL`). Every call is made
from the app's server with `Authorization: Bearer <WINKKEY_API_KEY>`. Never call WinkKey
from the browser and never send the key to it.

Each key belongs to one Wink environment (for example test or production). The key alone
decides which; a Wink ID created in one environment doesn't exist in the other.

## 1. Create a sign-in request

`POST /api/merchant/auth-request` with JSON:

```json
{
  "ttlSeconds": 600,
  "callbackUrl": "https://your-app.example.com/api/sign-in/complete",
  "displayTitle": "Your App",
  "displaySubtext": "Sign in to continue"
}
```

| Field | Meaning |
|---|---|
| `callbackUrl` | Where WinkKey sends the person afterwards. https in production; `http://localhost:<port>` works for local dev. Sent with every request; nothing to register. |
| `ttlSeconds` | How long the link works. 600 for normal sign-in. |
| `displayTitle`, `displaySubtext` | Shown to the person on winkkey.net. Use the app's name. |
| `kind` (optional) | Leave out for sign-in. `"enroll"` first walks someone through creating a Wink ID (useful for invites, see `access-control.md`). |
| `sso` (optional) | Single sign-on for this one link: `true` lets it complete from the person's existing WinkKey sign-in, `false` always asks for a face scan or passkey. Leave out to use the key's setting (section 4). |
| `prompt` (optional) | `"login"` always asks for a fresh face scan or passkey, for payments or admin actions. `"none"` never shows a page (section 4). |
| `maxAgeSeconds` (optional) | Accept an existing sign-in only if its face scan or passkey is at most this old. |

Reply: `{ "rid": "…", "url": "https://winkkey.net/…", "expiresAt": "…" }`.
Send the browser to `url`. Keep `rid` on your side (in a signed cookie or your database),
never as a plain query parameter you trust.

## 2. Read the result

WinkKey redirects the person to `callbackUrl`. Your handler then calls:

`GET /api/merchant/auth-request/{rid}`

```json
{
  "status": "consumed",
  "consumedBy": { "winkTag": "…", "email": "person@example.com" },
  "consumedMethod": "winkkey",
  "sso": false,
  "authTime": 1795620000000,
  "winkAccessToken": "…",
  "winkAccessTokenExpiresAt": "…"
}
```

| Field | Meaning |
|---|---|
| `status` | `consumed` = the person finished; only then sign them in. `expired` = the link ran out. Anything else = not finished. |
| `consumedBy.winkTag` | The person's stable Wink id. Use it as their user id. |
| `consumedBy.email` | Email on their Wink account. May be missing or differ from one you know. |
| `consumedMethod` | How they confirmed: `winkkey` (passkey) or `wink_cloud` (face scan). Fine to store. |
| `sso` | `true` when the link completed from the person's existing WinkKey sign-in instead of a new scan. |
| `authTime` | When that face scan or passkey happened (unix ms). |
| `winkAccessToken` | Only needed to call other Wink APIs as this person. Don't put it in the browser. |

## 3. End the Wink session (optional, on sign-out)

`POST /api/merchant/signout` with JSON:

```json
{ "winkTag": "…" }
```

| Field | Meaning |
|---|---|
| `winkTag` | The person's Wink id (from `consumedBy.winkTag` at sign-in). Preferred. |
| `userId` | Alternative — the WinkKey `userId` if you stored that instead. One of the two is required. |

Reply: `204 No Content`. Idempotent — a person with no live Wink session returns 204 too, so
it's safe to call on every sign-out without a check first.

**When to call:** on sign-out, if you want the next sign-in to show the face camera again.
It ends the person's Wink session and their WinkKey single sign-on (section 4) in your key's
environment. If you don't call it, the next "Sign in with Wink" may resume that session and
send them straight back without another face match.

**When to skip:** if the person uses Wink for several apps and signing out of yours
shouldn't sign them out everywhere, or if your app has no sign-out (e.g. a single-use
confirmation link). Clearing your own session cookie is always enough to end the person's
session in your app.

## 4. Single sign-on across your apps (optional)

When the same people use more than one of your apps, single sign-on lets someone who signed
in to one app go straight into the next, with no second face scan. Nothing changes in the
sign-in code: WinkKey completes the link at once and redirects to `callbackUrl` as usual,
and step 2 reads the result as usual (with `sso: true`).

Every key starts with it off. Turn it on for all links from a key, once, with that key:

```
PATCH /api/merchant/settings
{ "ssoEnabled": true }
```

`GET /api/merchant/settings` shows the current value. The developer can also switch it, see
sign-in stats and rotate the key at `https://winkkey.net/manage`, signed in with the Wink ID
the key is registered to. A link's own `sso` field overrides the key setting.

It applies only when the person already signed in with WinkKey in the same browser, in the
same Wink environment, within the last 8 hours of use and 24 hours of the face scan. In-app
browsers (Gmail, Slack on iPhone) keep their own cookies, so people sign in normally there.

Keep a fresh scan where it matters: send `prompt: "login"` (or `sso: false`) on links that
approve payments or admin actions. `prompt: "none"` never shows a page: without an existing
sign-in WinkKey returns to `callbackUrl` with `?error=login_required`, and the result has
`status: "failed"` and `error: "login_required"`.

## Error codes to pass back to the sign-in page

Redirect to `/signin?signin_error=<code>` and show a friendly line:

| Code | When | Message |
|---|---|---|
| `expired` | Pending cookie missing, bad or older than 10 minutes, or status `expired` | "The sign-in took too long or was already used. Please try again." |
| `not_completed` | Status isn't `consumed` | "WinkKey didn't finish the sign-in. Please try again." |
| `winkkey` | A WinkKey call failed | "We couldn't reach WinkKey. Please try again in a minute." |
| `no_access` | Consumed, but not on the allowlist | "This Wink account doesn't have access. Ask the app's owner to add you." |
