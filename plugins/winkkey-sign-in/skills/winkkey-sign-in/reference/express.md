# Node with Express

No database and no extra packages beyond Express: the pending sign-in and the session are
signed, HttpOnly cookies. Node 18+ (built-in `fetch`). For Fastify, Koa, Hono or plain
`http`, keep `winkkey.js` and port the four handlers.

## `winkkey.js`

```js
import { createHmac, randomBytes, timingSafeEqual } from "node:crypto";

const WINKKEY = (process.env.WINKKEY_BASE_URL || "https://winkkey.net/v2").trim().replace(/\/+$/, "");
export const APP_URL = (process.env.PUBLIC_BASE_URL || "http://localhost:3000").trim().replace(/\/+$/, "");
export const SESSION_COOKIE = "wink_session";
export const PENDING_COOKIE = "wink_pending";
const SESSION_HOURS = 8;
const PENDING_SECONDS = 600;

const apiKey = () => {
  const k = (process.env.WINKKEY_API_KEY || "").trim();
  if (!k) throw new Error("WINKKEY_API_KEY is not set");
  return k;
};
const secret = () => {
  const s = (process.env.SESSION_SECRET || "").trim();
  if (s.length < 32) throw new Error("SESSION_SECRET is missing or too short");
  return s;
};
const mac = (body) => createHmac("sha256", secret()).update(body).digest("base64url");
const now = () => Math.floor(Date.now() / 1000);

export function seal(payload) {
  const body = Buffer.from(JSON.stringify(payload)).toString("base64url");
  return `${body}.${mac(body)}`;
}
export function unseal(value) {
  if (!value) return null;
  const [body, sig] = value.split(".");
  if (!body || !sig) return null;
  const a = Buffer.from(sig), b = Buffer.from(mac(body));
  if (a.length !== b.length || !timingSafeEqual(a, b)) return null;
  try {
    const p = JSON.parse(Buffer.from(body, "base64url").toString());
    return typeof p.exp === "number" && p.exp > now() ? p : null;
  } catch { return null; }
}

function readCookie(req, name) {
  const hit = (req.headers.cookie || "").split(/;\s*/).find((c) => c.startsWith(name + "="));
  return hit ? decodeURIComponent(hit.slice(name.length + 1)) : undefined;
}
function setCookie(res, name, value, maxAge) {
  const secure = APP_URL.startsWith("https://") ? "; Secure" : ""; // Secure on plain http drops the cookie
  res.append("Set-Cookie", `${name}=${encodeURIComponent(value)}; Path=/; Max-Age=${maxAge}; HttpOnly; SameSite=Lax${secure}`);
}
export function safeReturnTo(v) {
  return typeof v === "string" && v.startsWith("/") && !v.startsWith("//") && !v.includes("://") && !v.startsWith("/api/") ? v : "/";
}

/** The signed-in person ({ winkTag, email, method, exp }) or null. */
export const getSession = (req) => unseal(readCookie(req, SESSION_COOKIE));

/** Express middleware for protected pages and APIs. */
export function requireSession(req, res, next) {
  const s = getSession(req);
  if (s) { req.session = s; return next(); }
  if (req.path.startsWith("/api/")) return res.status(401).json({ error: "Sign in first" });
  res.redirect(`${APP_URL}/signin?return_to=${encodeURIComponent(req.originalUrl)}`);
}

export function mountWinkSignIn(app) {
  app.get("/api/sign-in/start", async (req, res) => {
    const returnTo = safeReturnTo(req.query.return_to);
    try {
      const r = await fetch(`${WINKKEY}/api/merchant/auth-request`, {
        method: "POST",
        headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey()}` },
        body: JSON.stringify({
          ttlSeconds: PENDING_SECONDS,
          callbackUrl: `${APP_URL}/api/sign-in/complete`,
          displayTitle: "Your App", // ← the app's name
          displaySubtext: "Sign in to continue",
        }),
      });
      if (!r.ok) throw new Error(`WinkKey couldn't start sign-in (${r.status})`);
      const { rid, url } = await r.json();
      if (!rid || !url) throw new Error("WinkKey's reply was missing the sign-in link");
      setCookie(res, PENDING_COOKIE, seal({ rid, returnTo, n: randomBytes(12).toString("base64url"), exp: now() + PENDING_SECONDS }), PENDING_SECONDS);
      res.redirect(url);
    } catch (e) {
      console.error("[sign-in/start]", e.message);
      res.redirect(`${APP_URL}/signin?signin_error=winkkey&return_to=${encodeURIComponent(returnTo)}`);
    }
  });

  app.get("/api/sign-in/complete", async (req, res) => {
    const pending = unseal(readCookie(req, PENDING_COOKIE));
    setCookie(res, PENDING_COOKIE, "", 0); // single use
    const fail = (code) => res.redirect(`${APP_URL}/signin?signin_error=${code}`);
    if (!pending) return fail("expired");
    try {
      const r = await fetch(`${WINKKEY}/api/merchant/auth-request/${encodeURIComponent(pending.rid)}`, { headers: { Authorization: `Bearer ${apiKey()}` } });
      if (!r.ok) throw new Error(`WinkKey couldn't read the sign-in (${r.status})`);
      const poll = await r.json();
      const winkTag = poll.consumedBy?.winkTag;
      if (poll.status !== "consumed" || !winkTag) return fail(poll.status === "expired" ? "expired" : "not_completed");
      // Allowlist goes here (access-control.md): if (!allowed(poll.consumedBy?.email)) return fail("no_access");
      setCookie(res, SESSION_COOKIE, seal({ winkTag, email: poll.consumedBy?.email ?? null, method: poll.consumedMethod ?? null, exp: now() + SESSION_HOURS * 3600 }), SESSION_HOURS * 3600);
      res.redirect(`${APP_URL}${pending.returnTo}`); // APP_URL, never the request's host
    } catch (e) {
      console.error("[sign-in/complete]", e.message);
      fail("winkkey");
    }
  });

  app.post("/api/sign-out", (req, res) => {
    setCookie(res, SESSION_COOKIE, "", 0);
    res.redirect(303, `${APP_URL}/signin?signed_out=1`);
  });
}
```

## Wiring it up

```js
import express from "express";
import { mountWinkSignIn, requireSession, getSession, safeReturnTo } from "./winkkey.js";

const app = express();
app.set("trust proxy", true); // behind a proxy or container host
mountWinkSignIn(app);

const ERRORS = {
  expired: "The sign-in took too long or was already used. Please try again.",
  not_completed: "WinkKey didn't finish the sign-in. Please try again.",
  winkkey: "We couldn't reach WinkKey. Please try again in a minute.",
  no_access: "This Wink account doesn't have access. Ask the app's owner to add you.",
};
app.get("/signin", (req, res) => {
  const returnTo = safeReturnTo(req.query.return_to);
  if (getSession(req)) return res.redirect(returnTo);
  const error = req.query.signin_error ? ERRORS[req.query.signin_error] ?? ERRORS.not_completed : "";
  // Render with the app's own templates; escape anything taken from the query string.
  res.send(`<h1>Sign in</h1>${error ? `<p role="alert">${error}</p>` : ""}
    <a href="/api/sign-in/start?return_to=${encodeURIComponent(returnTo)}">Sign in with Wink</a>`);
});

app.use("/dashboard", requireSession);             // protect what needs it
app.get("/dashboard", (req, res) => res.send(`Signed in as ${req.session.email ?? req.session.winkTag}`));
```

Load the env file at startup (`node --env-file=.env server.js` on Node 20.6+, or `dotenv`).
