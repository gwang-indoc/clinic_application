# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

`clinic_application` is scaffolded (Next.js 16 App Router + TypeScript + Tailwind, `src/` layout, Supabase clients, `@supabase/ssr` session refresh). Read `AGENTS.md` before writing framework code — this Next.js version has breaking changes from what most training data expects; the docs bundled in `node_modules/next/dist/docs/` are the authoritative reference.

**Known drift from earlier guidance in this file:** Next.js 16 renamed the `middleware.ts` file convention to `proxy.ts` (exported function `proxy`, same session-refresh behavior). The session-refresh logic itself still lives in `src/lib/supabase/middleware.ts` (a plain helper, not a special file) and is called from `src/proxy.ts`.

No Supabase project is linked yet — `supabase/migrations/` is empty and `src/lib/database.types.ts` is a placeholder. Run `npx supabase init`-derived commands below once a project is linked.

## Target stack

- **Next.js (App Router) + React + TypeScript** — the web application.
- **Supabase** — Postgres database *and* authentication. No separate auth service, no custom session store.
- **Vercel** — hosting. Preview deployments per branch, production on `main`.

Because Supabase is both DB and auth, authorization lives primarily in **Postgres Row Level Security policies**, not in application code. Treat RLS as the security boundary; app-level checks are UX, not enforcement.

## Scaffolding (already done)

Provisioned with:

```bash
npx create-next-app@latest . --typescript --app --eslint --tailwind --src-dir --import-alias "@/*"
npm install @supabase/supabase-js @supabase/ssr server-only
npx supabase init          # creates supabase/ with config + migrations
```

## Commands

```bash
npm run dev            # local dev server (localhost:3000)
npm run build          # production build — run before pushing; Vercel fails on type/lint errors
npm run lint
npx tsc --noEmit       # type check without emitting
```

Supabase local stack (requires Docker):

```bash
npx supabase start                          # local Postgres + Auth + Studio
npx supabase stop
npx supabase migration new <name>           # author a schema change
npx supabase db reset                       # rebuild local DB from migrations + seed
npx supabase db push                        # apply migrations to the linked remote project
npx supabase gen types typescript --local > src/lib/database.types.ts
```

Tests (once a runner is added — prefer Vitest for units, Playwright for e2e):

```bash
npx vitest                       # watch
npx vitest run                   # once
npx vitest run path/to/file.test.ts -t "test name"   # single file / single test
npx playwright test --ui
npx playwright test tests/login.spec.ts:12
```

## Supabase client architecture (the main thing to get right)

There are **three distinct Supabase clients**, and using the wrong one is the most common source of bugs. Use `@supabase/ssr`, never the deprecated `auth-helpers` packages.

| Context | Where | Key used | Notes |
|---|---|---|---|
| Browser client | Client Components (`'use client'`) | anon | created with `createBrowserClient` |
| Server client | Server Components, Route Handlers, Server Actions | anon | `createServerClient` wired to Next's `cookies()`; carries the user's session, so RLS applies |
| Admin client | server-only modules | **service role** | bypasses RLS entirely — never import from anything reachable by client code |

Conventions:

- Keep these in `src/lib/supabase/{client,server,admin}.ts` so the import path itself signals which context you're in.
- **`src/proxy.ts` (Next.js 16's renamed `middleware.ts` convention — exported function `proxy`) must refresh the auth session on every request**, via `updateSession()` in `src/lib/supabase/middleware.ts`, and write refreshed cookies back onto the response. Without it, Server Components see stale/expired sessions and users get randomly logged out.
- Use `supabase.auth.getUser()` (validates with the auth server) for anything security-relevant, not `getSession()` (reads the cookie).
- Any table added must ship with RLS enabled and explicit policies in the same migration. A table without policies is either fully open or fully closed — both are bugs.

## Environment variables

- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` — safe in the browser; RLS is what protects the data.
- `SUPABASE_SERVICE_ROLE_KEY` — server-only. Must never be prefixed `NEXT_PUBLIC_`, never referenced in a Client Component, and never in a module a Client Component imports.

Keep `.env.local` untracked; mirror the variable names (no values) in `.env.example`. Vercel needs the same set configured per environment (Production / Preview / Development).

## Schema changes

Migrations in `supabase/migrations/` are the source of truth for the database. Do not make schema edits in the Supabase Studio UI on a shared project — write a migration, apply it locally with `db reset`, regenerate `database.types.ts`, then push. Commit the regenerated types alongside the migration so type errors surface at build time.

## Clinical-data caution

This app handles patient/clinic data. Do not log, cache, or send request/response bodies containing patient identifiers to third-party services (analytics, error trackers, LLM calls) without the user explicitly deciding on it. Prefer soft-delete plus audit columns (`created_by`, `updated_at`) over hard deletes for clinical records.
