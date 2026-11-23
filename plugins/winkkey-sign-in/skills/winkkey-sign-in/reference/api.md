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
| `callbackUrl` | Where WinkKey sends the person afterwards. Must be https. Sent with every request; nothing to register. |
| `ttlSeconds` | How long the link works. 600 for normal sign-in. |
| `displayTitle`, `displaySubtext` | Shown to the person on winkkey.net. Use the app's name. |
| `kind` (optional) | Leave out for sign-in. `"enroll"` first walks someone through creating a Wink ID (useful for invites, see `access-control.md`). |

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
  "consumedMethod": "passkey",
  "winkAccessToken": "…",
  "winkAccessTokenExpiresAt": "…"
}
```

| Field | Meaning |
|---|---|
| `status` | `consumed` = the person finished; only then sign them in. `expired` = the link ran out. Anything else = not finished. |
| `consumedBy.winkTag` | The person's stable Wink id. Use it as their user id. |
| `consumedBy.email` | Email on their Wink account. May be missing or differ from one you know. |
| `consumedMethod` | How they confirmed (e.g. passkey, face). Fine to store. |
| `winkAccessToken` | Only needed to call other Wink APIs as this person. Don't put it in the browser. |

## Error codes to pass back to the sign-in page

Redirect to `/signin?signin_error=<code>` and show a friendly line:

| Code | When | Message |
|---|---|---|
| `expired` | Pending cookie missing, bad or older than 10 minutes, or status `expired` | "The sign-in took too long or was already used. Please try again." |
| `not_completed` | Status isn't `consumed` | "WinkKey didn't finish the sign-in. Please try again." |
| `winkkey` | A WinkKey call failed | "We couldn't reach WinkKey. Please try again in a minute." |
| `no_access` | Consumed, but not on the allowlist | "This Wink account doesn't have access. Ask the app's owner to add you." |
