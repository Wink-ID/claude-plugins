# Next.js (App Router)

No database needed: the pending sign-in and the session are signed, HttpOnly cookies.
Works on Next.js 13.4+; file names below assume `app/` and `lib/` at the project root
(use `src/app`, `src/lib` if the project has `src/`). Adjust the `@/` import alias if the
project doesn't define it.

## Files to add

| File | Purpose |
|---|---|
| `lib/winkkey.ts` | WinkKey calls, signed cookies, settings |
| `lib/session.ts` | `getSession()` / `requireSession()` for pages and handlers |
| `app/api/sign-in/start/route.ts` | Starts sign-in, redirects to WinkKey |
| `app/api/sign-in/complete/route.ts` | WinkKey returns here; sets the session |
| `app/api/sign-out/route.ts` | Ends the session (POST) |
| `app/signin/page.tsx` | "Sign in with Wink" page |
| `proxy.ts` (Next 16+) or `middleware.ts` (older) | Sends signed-out visitors to `/signin` |

## `lib/winkkey.ts`

```ts
import { createHmac, randomBytes, timingSafeEqual } from "node:crypto";

const WINKKEY = (process.env.WINKKEY_BASE_URL || "https://winkkey.net/v2").trim().replace(/\/+$/, "");
/** The https address people use to reach the app. Every redirect is built from it. */
export const APP_URL = (process.env.PUBLIC_BASE_URL || "http://localhost:3000").trim().replace(/\/+$/, "");
export const SESSION_COOKIE = "wink_session";
export const PENDING_COOKIE = "wink_pending";
export const SESSION_HOURS = 8;
export const PENDING_SECONDS = 600;

export interface Session {
  winkTag: string;
  email: string | null;
  method: string | null;
  exp: number; // seconds since epoch
}

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
const mac = (body: string) => createHmac("sha256", secret()).update(body).digest("base64url");

/** A tamper-proof cookie value: base64 JSON plus an HMAC. */
export function seal(payload: { exp: number } & Record<string, unknown>) {
  const body = Buffer.from(JSON.stringify(payload)).toString("base64url");
  return `${body}.${mac(body)}`;
}

/** The payload of a sealed value, or null if it's missing, altered or expired. */
export function unseal<T extends { exp: number }>(value: string | undefined): T | null {
  if (!value) return null;
  const [body, sig] = value.split(".");
  if (!body || !sig) return null;
  const a = Buffer.from(sig);
  const b = Buffer.from(mac(body));
  if (a.length !== b.length || !timingSafeEqual(a, b)) return null;
  try {
    const p = JSON.parse(Buffer.from(body, "base64url").toString()) as T;
    return typeof p.exp === "number" && p.exp > Date.now() / 1000 ? p : null;
  } catch {
    return null;
  }
}

/** Secure only over https; on plain http localhost a Secure cookie would be dropped. */
export const cookieOptions = (maxAge: number) => ({ httpOnly: true, sameSite: "lax" as const, secure: APP_URL.startsWith("https://"), path: "/", maxAge });

/** Only paths inside the app, so a shared sign-in link can't send anyone elsewhere. */
export function safeReturnTo(v: string | null | undefined) {
  return v && v.startsWith("/") && !v.startsWith("//") && !v.includes("://") && !v.startsWith("/api/") ? v : "/";
}

export async function createAuthRequest(): Promise<{ rid: string; url: string }> {
  const res = await fetch(`${WINKKEY}/api/merchant/auth-request`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey()}` },
    body: JSON.stringify({
      ttlSeconds: PENDING_SECONDS,
      callbackUrl: `${APP_URL}/api/sign-in/complete`,
      displayTitle: "Your App", // ← the app's name
      displaySubtext: "Sign in to continue",
    }),
    cache: "no-store",
  });
  if (!res.ok) throw new Error(`WinkKey couldn't start sign-in (${res.status})`);
  const data = (await res.json()) as { rid?: string; url?: string };
  if (!data.rid || !data.url) throw new Error("WinkKey's reply was missing the sign-in link");
  return { rid: data.rid, url: data.url };
}

export async function readAuthRequest(rid: string) {
  const res = await fetch(`${WINKKEY}/api/merchant/auth-request/${encodeURIComponent(rid)}`, {
    headers: { Authorization: `Bearer ${apiKey()}` },
    cache: "no-store",
  });
  if (!res.ok) throw new Error(`WinkKey couldn't read the sign-in (${res.status})`);
  return (await res.json()) as { status?: string; consumedBy?: { winkTag?: string | null; email?: string | null }; consumedMethod?: string | null };
}

export const nonce = () => randomBytes(12).toString("base64url");
```

## `lib/session.ts`

```ts
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { SESSION_COOKIE, unseal, type Session } from "./winkkey";

/** The signed-in person, or null. */
export async function getSession(): Promise<Session | null> {
  return unseal<Session>((await cookies()).get(SESSION_COOKIE)?.value);
}

/** The signed-in person; anyone else is sent to sign in. Use in every protected page, route and server action. */
export async function requireSession(returnTo = "/"): Promise<Session> {
  const s = await getSession();
  if (!s) redirect(`/signin?return_to=${encodeURIComponent(returnTo)}`);
  return s;
}
```

(On Next 14 and older `cookies()` is synchronous; `await` on it is harmless.)

## `app/api/sign-in/start/route.ts`

```ts
import { NextResponse, type NextRequest } from "next/server";
import { APP_URL, PENDING_COOKIE, PENDING_SECONDS, cookieOptions, createAuthRequest, nonce, safeReturnTo, seal } from "@/lib/winkkey";

export async function GET(req: NextRequest) {
  const returnTo = safeReturnTo(req.nextUrl.searchParams.get("return_to"));
  try {
    const { rid, url } = await createAuthRequest();
    const res = NextResponse.redirect(url);
    res.cookies.set(PENDING_COOKIE, seal({ rid, returnTo, n: nonce(), exp: Math.floor(Date.now() / 1000) + PENDING_SECONDS }), cookieOptions(PENDING_SECONDS));
    return res;
  } catch (e) {
    console.error("[sign-in/start]", e instanceof Error ? e.message : e);
    return NextResponse.redirect(`${APP_URL}/signin?signin_error=winkkey&return_to=${encodeURIComponent(returnTo)}`);
  }
}
```

## `app/api/sign-in/complete/route.ts`

```ts
import { NextResponse, type NextRequest } from "next/server";
import { APP_URL, PENDING_COOKIE, SESSION_COOKIE, SESSION_HOURS, cookieOptions, readAuthRequest, seal, unseal } from "@/lib/winkkey";

export async function GET(req: NextRequest) {
  const pending = unseal<{ rid: string; returnTo: string; exp: number }>(req.cookies.get(PENDING_COOKIE)?.value);
  const fail = (code: string) => NextResponse.redirect(`${APP_URL}/signin?signin_error=${code}`);
  let res: NextResponse;
  if (!pending) res = fail("expired");
  else {
    try {
      const r = await readAuthRequest(pending.rid);
      const winkTag = r.consumedBy?.winkTag;
      if (r.status !== "consumed" || !winkTag) res = fail(r.status === "expired" ? "expired" : "not_completed");
      // Allowlist goes here (access-control.md): else if (!allowed(r.consumedBy?.email)) res = fail("no_access");
      else {
        res = NextResponse.redirect(`${APP_URL}${pending.returnTo}`); // APP_URL, never req.url
        const exp = Math.floor(Date.now() / 1000) + SESSION_HOURS * 3600;
        res.cookies.set(SESSION_COOKIE, seal({ winkTag, email: r.consumedBy?.email ?? null, method: r.consumedMethod ?? null, exp }), cookieOptions(SESSION_HOURS * 3600));
      }
    } catch (e) {
      console.error("[sign-in/complete]", e instanceof Error ? e.message : e);
      res = fail("winkkey");
    }
  }
  res.cookies.set(PENDING_COOKIE, "", cookieOptions(0)); // single use
  return res;
}
```

## `app/api/sign-out/route.ts`

```ts
import { NextResponse } from "next/server";
import { APP_URL, SESSION_COOKIE, cookieOptions } from "@/lib/winkkey";

/** POST only, so a link on another site can't sign anyone out. */
export async function POST() {
  const res = NextResponse.redirect(`${APP_URL}/signin?signed_out=1`, 303);
  res.cookies.set(SESSION_COOKIE, "", cookieOptions(0));
  return res;
}
```

A sign-out button is a form: `<form action="/api/sign-out" method="post"><button>Sign out</button></form>`.

## `app/signin/page.tsx`

```tsx
import { redirect } from "next/navigation";
import { getSession } from "@/lib/session";
import { safeReturnTo } from "@/lib/winkkey";

const ERRORS: Record<string, string> = {
  expired: "The sign-in took too long or was already used. Please try again.",
  not_completed: "WinkKey didn't finish the sign-in. Please try again.",
  winkkey: "We couldn't reach WinkKey. Please try again in a minute.",
  no_access: "This Wink account doesn't have access. Ask the app's owner to add you.",
};

export default async function SignInPage({ searchParams }: { searchParams: Promise<{ return_to?: string; signin_error?: string; signed_out?: string }> }) {
  const q = await searchParams;
  const returnTo = safeReturnTo(q.return_to);
  if (await getSession()) redirect(returnTo);
  const error = q.signin_error ? ERRORS[q.signin_error] ?? ERRORS.not_completed : null;
  return (
    <main>
      <h1>Sign in</h1>
      <p>Confirm it's you with your face or passkey on WinkKey.</p>
      {error && <p role="alert">{error}</p>}
      {q.signed_out && !error && <p role="status">You're signed out.</p>}
      <a href={`/api/sign-in/start?return_to=${encodeURIComponent(returnTo)}`}>Sign in with Wink</a>
    </main>
  );
}
```

(On Next 14 and older, `searchParams` is a plain object: drop the `Promise<>` and the `await`.)
Style the page to match the app.

## Request filter

Next 16+: `proxy.ts` exporting `proxy`. Older: `middleware.ts` exporting `middleware`.
It only checks that a session cookie exists (middleware may run where `node:crypto` isn't
available); the real check is `requireSession()` in each protected page and handler.

```ts
import { NextResponse, type NextRequest } from "next/server";

const PUBLIC = ["/signin", "/api/sign-in/", "/api/sign-out"]; // add public pages (e.g. "/") here

export function proxy(req: NextRequest) { // `middleware` on Next 15 and older
  const path = req.nextUrl.pathname;
  if (PUBLIC.some((p) => path === p || path.startsWith(p))) return NextResponse.next();
  if (req.cookies.get("wink_session")?.value) return NextResponse.next();
  if (path.startsWith("/api/")) return NextResponse.json({ error: "Sign in first" }, { status: 401 });
  const to = new URL("/signin", req.url);
  to.searchParams.set("return_to", path + req.nextUrl.search);
  return NextResponse.redirect(to);
}

export const config = { matcher: ["/((?!_next/static|_next/image|.*\\.[a-zA-Z0-9]+$).*)"] };
```

If only part of the app needs sign-in, invert it: list the protected prefixes instead.

## Using the session

```tsx
// In a protected page
const me = await requireSession("/dashboard");
return <p>Signed in as {me.email ?? me.winkTag}</p>;
```

In route handlers and server actions call `getSession()` and return 401 / throw if null.
Pass only what the browser needs to client components, never the whole session.

## Local https

Add the tunnel host to `next.config` so the dev server accepts it:

```ts
const nextConfig = { allowedDevOrigins: ["your-tunnel-host.trycloudflare.com"] };
```
