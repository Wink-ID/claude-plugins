# Wink plugins for Claude

## Sign in with Wink (`winkkey-sign-in`)

Adds WinkKey biometric sign-in (selfie, Face ID, Touch ID or passkey) to your web app.
Claude asks for your WinkKey key, builds the sign-in pages, and walks you through your
first sign-in. Next.js and Node/Express are covered in detail; other stacks follow the same
two API calls.

### 1. Get a WinkKey key

Create one on the WinkKey portal. It starts with `wkey_`.

### 2. Install the skill

In your app's folder, in Claude Code:

```
/plugin marketplace add wink-cloud/claude-plugins
/plugin install winkkey-sign-in@wink
```

### 3. Ask Claude

> Add Sign in with Wink to this app.

Claude asks for your key (paste it in the chat, or put `WINKKEY_API_KEY=wkey_…` in your
`.env.local` / `.env` yourself), builds sign-in, and asks you to sign in once to check it.

WinkKey only returns people to https addresses, so for local testing Claude will suggest
an https tunnel (for example `npx cloudflared tunnel --url http://localhost:3000`).

---

## Maintainers

- Skill source: `plugins/winkkey-sign-in/skills/winkkey-sign-in/`
- Bump `version` in both `plugins/winkkey-sign-in/.claude-plugin/plugin.json` and
  `.claude-plugin/marketplace.json` on each change.
- `claude plugin validate .` before publishing.
