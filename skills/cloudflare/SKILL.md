---
name: cloudflare
description: >
  Use when setting up a Next.js project for deployment on Cloudflare Workers
  using @opennextjs/cloudflare. Trigger when the user says "deploy to Cloudflare",
  "set up Cloudflare Workers", "configure wrangler", "cloudflare deployment",
  or any request to migrate a Next.js app from Vercel/other platforms to
  Cloudflare Workers.
allowed-tools: Read, Write, Edit, Bash, Glob
---

# Cloudflare Workers for Next.js

Set up a Next.js project for deployment on Cloudflare Workers using @opennextjs/cloudflare.

Reference: https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/

## Invocation

```
/cloudflare                  Full setup (steps 1-7, no deploy)
/cloudflare --deploy         Full setup + deploy to Cloudflare
```

When invoked, run all steps sequentially. Do NOT ask for confirmation between
steps — just execute. Only stop if a step fails (e.g. build error in step 7).

If `--deploy` is passed, also run step 8 (deploy). Otherwise, inform the user
how to deploy manually at the end.

## Before Starting

1. Read `package.json` to detect the package manager (`packageManager` field or lockfile: `pnpm-lock.yaml` = pnpm, `yarn.lock` = yarn, `package-lock.json` = npm). Use the detected package manager for all commands below.
2. Read `package.json` to get the project `name` field — use it as the worker name in `wrangler.jsonc`. If the name is generic (e.g. `my-app`, `my-v0-project`), derive a name from the directory name instead.
3. Read `next.config.mjs` (or `next.config.js` / `next.config.ts`) to understand the current config.
4. Read `.gitignore` to check if `.open-next` is already listed.

## Steps

### 1. Install dependencies

```bash
<pkg-manager> add @opennextjs/cloudflare@latest
<pkg-manager> add -D wrangler@latest
```

If `@opennextjs/cloudflare` has an unmet peer dependency for `next`, update `next` to the nearest compatible version.

### 2. Create `wrangler.jsonc` in project root

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "main": ".open-next/worker.js",
  "name": "<worker-name>",
  "compatibility_date": "<today's date YYYY-MM-DD>",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS",
  },
}
```

- `name`: use the project name detected in step 0 (lowercase, hyphens only)
- `compatibility_date`: use today's date (must be `2024-09-23` or later)
- `nodejs_compat` flag is **required**

### 3. Create `open-next.config.ts` in project root

```typescript
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
export default defineCloudflareConfig();
```

### 4. Update `package.json` scripts

Add these scripts while **keeping all existing scripts intact**:

```json
"preview": "opennextjs-cloudflare build && opennextjs-cloudflare preview",
"deploy": "opennextjs-cloudflare build && opennextjs-cloudflare deploy",
"cf-typegen": "wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts"
```

Do NOT overwrite existing `build`, `dev`, `lint`, `start`, or any other scripts.

### 5. Clean up `next.config`

- Remove `swcMinify: true` if present (deprecated in Next 15+, causes warnings)
- Keep everything else as-is

### 6. Add `.open-next` to `.gitignore`

Only add if not already present. Append under a `# cloudflare` comment section.

### 7. Verify the setup

Run the preview command to build and test in the local workerd runtime:

```bash
<pkg-manager> run preview
```

If the build fails, read the error output and fix the issue before proceeding.

### 8. Deploy (only if the user explicitly asks)

Do NOT deploy automatically. Inform the user they can deploy with:

```bash
<pkg-manager> run deploy
```

And that they need to run `wrangler login` first if not already authenticated.

## Important Notes

- `nodejs_compat` compatibility flag is **required**
- `compatibility_date` must be `2024-09-23` or later
- The preview command uses the actual `workerd` runtime — more accurate than `next dev` for testing Cloudflare compatibility
- Client-side libraries (Three.js, React Three Fiber, etc.) work fine since they run in the browser, not the worker
- Environment variables: `NEXT_PUBLIC_*` are inlined at build time; server-side vars need Cloudflare Workers secrets or wrangler env config
- If the project uses `@vercel/analytics` or other Vercel-specific packages, warn the user that these will not work on Cloudflare and suggest alternatives
