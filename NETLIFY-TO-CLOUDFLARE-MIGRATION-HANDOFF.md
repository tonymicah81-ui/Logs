# Netlify → Cloudflare Workers Migration Handoff
**Project:** Brokerforcloudfare  
**Documentation repository:** https://github.com/tonymicah81-ui/Logs  
**Application repository:** https://github.com/tonymicah81-ui/Brokerforcloudfare  
**Last updated:** 2026-10-09  
**Status:** Deployment builds, but the deployed Worker returns HTTP 500 at runtime. Migration is not yet complete.

> Purpose: Preserve the actual steps, errors, fixes, current findings, and unresolved work from this migration. Read this before editing code or repeating deployment attempts. Update this document as new evidence is collected so it can guide future Netlify-to-Cloudflare migrations.

## 1. Safety and project identity

- The populated application repository is **`tonymicah81-ui/Brokerforcloudfare`**.
- There is another similarly named repository with a hyphen in its name that is empty. **Do not use or modify the empty repository.**
- The application being migrated was previously hosted on Netlify. The goal is to run the same working application on Cloudflare first, not redesign or migrate its database.
- Deleting/recreating deployments repeatedly is not necessary for normal code changes. Prefer pushing a fix and allowing the connected Cloudflare build to deploy it.
- The user is new to Cloudflare and wants the work done one stage at a time, with explanations and a reusable written handoff.
- Do not create Cloudflare databases/storage or change billing as part of this hosting-only migration unless a demonstrated requirement appears.

## 2. Current deployment details

- Worker name: `brokerforcloudfare`
- Current test URL: https://brokerforcloudfare.tonyemicah200.workers.dev
- Current observed result: HTTP **500 Internal Server Error** on the homepage.
- Cloudflare config file: `wrangler.jsonc`
- Next.js: 15.5.27 (from build log)
- OpenNext Cloudflare adapter observed in build log: `@opennextjs/cloudflare 1.20.9`
- OpenNext AWS package observed in build log: `@opennextjs/aws 4.1.8`
- Wrangler observed in build log: 4.148.0
- Node observed in build log: 24.18.0
- The Worker was built/deployed far enough to execute requests; current blocker is runtime, not the original TypeScript build failure.

## 3. Build/deployment timeline: errors and changes

### Stage A — Missing OpenNext adapter during type checking

Initial Cloudflare build:
- `npm ci` / clean install succeeded (494 packages).
- Cloudflare ran `npm run build`, which invoked `next build`.
- Next.js compiled, but TypeScript failed in `open-next.config.ts` at line 1:
  `Cannot find module '@opennextjs/cloudflare' or its corresponding type declarations.`

Relevant config:
```ts
import { defineCloudflareConfig } from "@opennextjs/cloudflare";

export default defineCloudflareConfig();
```

**Change made:** Commit `6ae652e7a1f63717e8a966d196408229e79aa20e` added an on-demand install of `@opennextjs/cloudflare@latest` before the regular Next build.

**Outcome:** Next build/type checking succeeded and all 78 static pages were generated. But Cloudflare deploy then failed with:
`ERROR Could not find compiled Open Next config, did you run the build command?`

**Lesson:** `next build` by itself is not the complete OpenNext Cloudflare build. The adapter must run its own build step to generate `.open-next/`.

### Stage B — OpenNext build was accidentally made recursive

**Attempted change:** The build script was changed to run Next build and then `npx --yes @opennextjs/cloudflare build` in the same `build` script.

**Outcome:** Cloudflare build ran for over 30 minutes and timed out.

The full log was saved in this repository as `Log`. It showed this cycle:
1. Cloudflare runs `npm run build`.
2. The script runs `next build`.
3. It starts the OpenNext build.
4. OpenNext says it is building the Next.js app and calls the project's `npm run build` again.
5. The same script starts again and repeats until timeout.

**Lesson:** Do not make the project's ordinary `build` script call the OpenNext build if OpenNext in turn calls the project build script. Keep the Next build and Cloudflare adapter build commands separate.

### Stage C — Separate Cloudflare build script

**Fix committed:** `fd4231a7990c44a08cc1bc1faa614582ea6ffa74`.

Current relevant `package.json` scripts:
```json
{
  "scripts": {
    "dev": "next dev --port 5000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "build:cloudflare": "npm install --no-save --package-lock=false @opennextjs/cloudflare@latest && npx --yes @opennextjs/cloudflare build",
    "preview:cloudflare": "npm install --no-save --package-lock=false @opennextjs/cloudflare@latest && npx --yes @opennextjs/cloudflare build && npx --yes @opennextjs/cloudflare preview",
    "deploy:cloudflare": "npm install --no-save --package-lock=false @opennextjs/cloudflare@latest && npx --yes @opennextjs/cloudflare build && npx wrangler deploy",
    "cf-typegen": "npx wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts"
  }
}
```

**Cloudflare Workers Builds settings for this project:**
- Build command: `npm run build:cloudflare`
- Deploy command: `npx wrangler deploy`

Do not revert to the old build command `npm run build` as the Cloudflare build command, and do not put `@opennextjs/cloudflare build` back into the ordinary `build` script.

Current `wrangler.jsonc`:
```json
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "brokerforcloudfare",
  "main": ".open-next/worker.js",
  "compatibility_date": "2026-10-06",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS"
  },
  "observability": {
    "enabled": true
  }
}
```

No `wrangler.toml` was found/reported; `wrangler.jsonc` is the config in use.

## 4. Latest runtime error from the new logs (2026-10-09)

The user uploaded separate Cloudflare log entries named around `2026-10-09 08:05:00.189` and `2026-10-09 08:05:02`. The most diagnostic record states:

```
EvalError: Code generation from strings disallowed for this context
    at Function (<anonymous>)
    at f4 (worker.js:97128:22)
    at s4.get (worker.js:112303:73)
    at j.resolve (worker.js:96066:196)
    at s4.resolveAll (worker.js:112335:78)
    at l4.resolveAll (worker.js:97006:82)
    at l4.resolveAll (worker.js:97006:82)
    at l4.resolveAll (worker.js:97006:82)
    at l4.resolveAll (worker.js:118540:41)
    at l4.fromJSON (worker.js:118465:132)
```

Log facts:
- Request `GET /` to the Worker returned HTTP 500.
- Request `GET /favicon.ico` also returned HTTP 500 with the same EvalError.
- One event reported about 412 ms wall time / 397 ms CPU time for `/`; favicon was about 17 ms.
- The stack trace is minified and does not itself identify the original package or source module.
- This is a runtime compatibility error, not evidence of a missing Cloudflare database or a need to change billing.
- It is not yet proven which dependency causes the dynamic code generation. Do not label the cause as confirmed until the code path is isolated or a test verifies it.

## 5. First code path to investigate

Files inspected:
- `app/page.tsx`
- `lib/platformSettings.ts`
- `lib/firebase.ts`
- `open-next.config.ts`
- `next.config.ts`
- `package.json`
- `wrangler.jsonc`
- `.env.example`
- `CLOUDFLARE_DEPLOYMENT.md`

The homepage (`app/page.tsx`) is force-dynamic and calls, in parallel:
- `loadAllSettings()`
- `getInvestmentPlans()`
- `getPlatformStats()`

The settings module `lib/platformSettings.ts` imports:
```ts
import { doc, getDoc, setDoc } from 'firebase/firestore';
import { db } from './firebase';
```
It creates Firestore document references for `platform_settings/config` and `platform_secrets/config`, then reads them in `loadAllSettings()`.

`lib/firebase.ts` initializes the Firebase web SDK and exports `getFirestore(app)`. The project currently uses the Firebase web configuration (some values have fallbacks in code).

**Working hypothesis, not confirmed root cause:** Importing/using the Firebase client Firestore SDK in the Next.js server runtime inside a Cloudflare Worker may trigger a dependency that attempts dynamic code generation. The EvalError and stack trace make this worth testing first, but they do not prove Firebase is the cause.

**Next investigation:**
1. Inspect the full settings/auth/admin code paths before changing implementation, particularly how the secrets document is protected and how admin writes work.
2. Identify the exact module/dependency responsible. Use targeted isolation or a minimal runtime test rather than guessing from minified stack frames.
3. If server-side Firestore SDK usage is confirmed as the cause, consider using a Worker-compatible server-side data access approach (for example, Firestore REST API via `fetch` with correct authentication) or another supported server integration. Preserve Firebase Authentication, Firestore Security Rules, and all intended access controls. Do not make the secrets document publicly accessible as a workaround.
4. Check other server dependencies for dynamic code generation if Firestore is not confirmed.
5. Make a small, reviewable change, deploy, and re-check logs and homepage before proceeding to unrelated fixes.

## 6. Firebase configuration and secrets

The Firebase web config is currently defined in `lib/firebase.ts` with environment-variable overrides and fallback values:
- `NEXT_PUBLIC_FIREBASE_API_KEY`
- `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`
- `NEXT_PUBLIC_FIREBASE_PROJECT_ID`
- `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`
- `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- `NEXT_PUBLIC_FIREBASE_APP_ID`

The user intentionally keeps the non-secret Firebase web configuration in code during rapid testing so they do not need to re-enter the same values on each deployment. This is generally acceptable for Firebase's normal web configuration, provided Firebase Auth, Firestore Rules, and other security controls are correctly configured.

**Never treat actual credentials as ordinary client config.** SMTP passwords, private API tokens, GitHub personal access tokens, Cloudinary API secrets, Telegram bot tokens, service-account private keys, and signing/encryption secrets must not be committed in source code. Move those to Cloudflare Secrets/Environment Variables before production. If a real secret has already been committed, remove it from code and rotate/revoke it; removing it in a later commit does not erase it from Git history.

Do not make the user configure all Firebase variables in Cloudflare just to investigate the current runtime error; code fallbacks already exist for the Firebase web config.

## 7. What does and does not need to move to Cloudflare

This is currently a hosting migration, not a full backend migration.

Keep the existing services until a demonstrated reason to change:
- Firebase Authentication
- Firebase Firestore
- Cloudinary media storage, if used
- Existing email/Telegram integrations, after their runtime compatibility and secrets are checked

Cloudflare offers optional services such as D1 (SQL database), KV (key-value storage), R2 (object/file storage), and Durable Objects. None must be created just because the app is hosted on Workers. The existing Firebase backend can remain. Do not change billing or create D1/KV/R2 during this diagnostic step.

Cloudflare dashboard locations useful for this task:
- Worker runtime logs: **Workers & Pages → brokerforcloudfare → Logs / Observability → Live** (labels may vary).
- Environment variables/secrets: **Workers & Pages → brokerforcloudfare → Settings → Variables and Secrets**.
- Trigger a fresh homepage request while logs are open, then inspect the exception and matching request ID.

## 8. Known useful links

- Application repo: https://github.com/tonymicah81-ui/Brokerforcloudfare
- Migration logs repo: https://github.com/tonymicah81-ui/Logs
- Current full build log in Logs repo: https://github.com/tonymicah81-ui/Logs/blob/main/Log
- Cloudflare test Worker: https://brokerforcloudfare.tonyemicah200.workers.dev
- OpenNext Cloudflare guide: https://developers.cloudflare.com/workers/framework-guides/web-apps/opennext/
- Cloudflare real-time logs: https://developers.cloudflare.com/workers/observability/logs/real-time-logs/
- Cloudflare Worker logs: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
- Cloudflare runtime errors: https://developers.cloudflare.com/workers/observability/errors/
- Cloudflare variables and secrets: https://developers.cloudflare.com/workers/configuration/environment-variables/
- Cloudflare storage options: https://developers.cloudflare.com/workers/platform/storage-options/

## 9. Reusable checklist for future Netlify → Cloudflare projects

Use this as a starting checklist, not a one-size-fits-all recipe. Inspect each project's framework, functions, environment, and dependencies.

1. **Identify the correct repo and working branch.** Confirm the populated project; do not assume similarly named repositories are interchangeable.
2. **Record the current working Netlify behavior.** Note framework/build command, output/functions, environment variables, redirects/rewrites, scheduled functions, webhooks, and storage/services.
3. **Inspect the framework and adapters.** For Next.js, choose a compatible Cloudflare deployment approach. For this existing Next.js app, OpenNext is currently configured.
4. **Separate build commands.** Keep the normal framework build separate from the platform adapter build if the adapter invokes the framework build internally.
5. **Set Cloudflare build/deploy commands.** For this app specifically: `npm run build:cloudflare` and `npx wrangler deploy`.
6. **Read the build log from the first real error.** Distinguish install/typecheck/compile/adapter/deploy errors; don't fix later symptoms first.
7. **Check runtime logs after successful deployment.** A successful build does not prove the app can execute on Workers.
8. **Test routes one by one.** Homepage, API endpoints, login/auth, admin functions, webhooks, uploads, and any dynamic routes.
9. **Audit platform-specific dependencies.** Netlify functions, Node APIs, native packages, dynamic eval/code generation, filesystem assumptions, and server SDKs may need adaptation.
10. **Keep the existing backend unless migration is explicitly intended.** Hosting migration does not require database migration.
11. **Keep real secrets out of Git.** Configure Cloudflare Secrets for sensitive values; rotate secrets exposed in Git.
12. **Document every change.** Record commit SHA, original error, change made, result, and any unresolved risks in this handoff.
13. **Verify before declaring success.** Open the live site, inspect logs, test important workflows, and confirm environment-specific services.

## 10. Current next action

**Do not start another speculative code edit yet.** First inspect and isolate the cause of `EvalError: Code generation from strings disallowed for this context`, beginning with the server-side Firestore path described above. Then append the finding, exact fix, commit SHA, deployment outcome, and post-deploy test results to this file.

Keep this handoff updated as work continues so future ChatGPT sessions can continue from evidence instead of repeating failed attempts.
