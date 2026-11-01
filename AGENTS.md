# RentEase: Project Rules (read by every agent, every session)

## Read order
1. docs/SPEC.md (product spec, 13 HARD INVARIANTS, acceptance tests)
2. ARCHITECTURE.md (naming contract; exists after Phase 0)
3. Your agent definition and the hand-off brief you were given
Subagents have no memory of earlier conversations. If it is not in these files or
your brief, you do not know it. Ask the orchestrator instead of guessing.

## Rules
1. CONTRACT. Use names exactly as written in ARCHITECTURE.md (tables, columns, enum
   values, RPCs and their argument names, routes, actions, components, cookies,
   CSS variables, i18n keys, test ids). Never invent or rename. If a change is
   needed, write a CONTRACT REQUEST in your status file and keep using the existing
   name. Only the orchestrator edits ARCHITECTURE.md.
2. OWNERSHIP. Edit only files in your owned paths (table below). Never edit another
   agent's files. Shared files (package.json, lockfile, ARCHITECTURE.md, next.config.ts,
   tsconfig, middleware.ts) are changed only by platform-engineer or the orchestrator.
   If you need a dependency or shared change, write a CHANGE REQUEST in your status file.
3. COMPLETENESS. No placeholders, no TODOs, no "implement later", no skipped files.
   Never stop in the middle of a file. If you must stop, finish the file first.
4. SECURITY. The 13 HARD INVARIANTS in docs/SPEC.md are non-negotiable. Never put a
   secret in source or client code. No dangerouslySetInnerHTML (only
   components/seo/JsonLd.tsx). No string-built SQL. No inline styles or inline event
   handlers. Never return private fields (phone, email, exact address) to public
   queries, server components, RSC payloads, metadata or JSON-LD.
5. GIT. Work only on your branch agent/<name> (an isolated worktree). Conventional
   commits (feat:, fix:, test:, docs:, chore:). Commit after each logical unit.
   Before reporting done: rebase onto the latest main, run `npm run typecheck` and
   `npm run lint` (once they exist), fix errors in YOUR files, commit. Never merge,
   never push to main, never force-push, never commit .env files or secrets.
6. NO NESTING. Do not spawn other subagents, except the built-in `research` agent for
   read-only exploration.
7. STATUS. Keep .agents/status/<name>.md updated: files written, exports/routes/RPCs
   actually implemented, commands run and results, CONTRACT/CHANGE REQUESTS, blockers.
   If you are stopped by quota, an error or an unclear spec, write "PAUSED: <why>,
   next step: <what>" there and stop. Do not guess.
8. SAFETY. No destructive commands outside your owned paths. Do not run installers
   for packages not in the SPEC allow-list.
9. FINAL REPORT. End with: files written, exports/routes/RPCs implemented, checks run
   with results, open issues, requests for the orchestrator. Keep it short.

## Ownership map
orchestrator       ARCHITECTURE.md (Part B), .agents/STATUS.md, merges, integration fixes
db-architect       sql/**, types/database.ts, tests/rls.test.sql, tests/state-machines.test.sql,
                   ARCHITECTURE.md Part A (delivered as .agents/status/db-contract.md, merged by orchestrator)
platform-engineer  package.json, lockfile, next.config.ts, tsconfig.json, tailwind.config.ts,
                   postcss.config.mjs, .eslintrc.json, .gitignore, .env.example, vercel.json,
                   middleware.ts, lib/env.ts, lib/utils.ts, lib/schemas.ts, lib/settings.ts,
                   lib/supabase/**, lib/server/supabase-admin.ts, lib/server/request.ts,
                   scripts/**, app/auth/callback/route.ts
design-system      app/layout.tsx, app/globals.css, app/not-found.tsx, app/error.tsx, app/loading.tsx,
                   app/offline/**, app/maintenance/**, components/ui/**, components/layout/**
                   (incl. NotificationBell), components/seo/**, components/charts/**,
                   lib/i18n.ts, messages/*/common.json, public/**
payments-api       app/api/payments/**, app/api/email/**, app/api/cron/**, lib/server/razorpay.ts,
                   lib/server/email.ts, components/payments/**, emails/**
public-site        app/page.tsx, app/(public)/**, app/(auth)/**, app/(dashboard)/onboarding/**,
                   app/robots.ts, app/sitemap.ts, components/listings/**, components/services/**,
                   actions/enquiries.ts, actions/favorites.ts, actions/reports.ts, actions/auth.ts,
                   messages/*/{auth,rooms,roommates,services,legal}.json
dashboards         app/(dashboard)/** (except onboarding), components/bookings/**,
                   components/messaging/**, lib/image.ts, actions/{bookings,quotes,listings,
                   reviews,verification,maintenance,tenancy,messages,profile}.ts,
                   messages/*/{bookings,dashboard}.json
admin-setup        app/(admin)/**, app/setup/**, app/api/setup/**, components/admin/**,
                   actions/admin.ts, lib/analytics-queries.ts, messages/*/{admin,setup}.json
docs-writer        README.md, setup-guide.md, supabase-setup.md, razorpay-setup.md,
                   deployment-guide.md, SECURITY.md, PRIVACY-COMPLIANCE.md, OPERATIONS.md
qa-tester          tests/api.test.ts, tests/e2e/**, tests/xss.test.md, TESTING.md
security-reviewer  read-only (reports go to the orchestrator)

## Conflict hotspots (why the rules above exist)
package.json and lockfile: platform-engineer installs ALL allow-listed dependencies up
front. If a lockfile conflict happens, regenerate with `npm install`, never hand-edit.
Translations are split per namespace. Global CSS tokens: design-system only; everyone
else uses Tailwind classes and the tokens that already exist.

## Skills precedence
Installed skills (design, UI, frontend, review) are advisory. Where a skill conflicts
with docs/SPEC.md, the HARD INVARIANTS, the dependency allow-list, or this file, the
SPEC and AGENTS.md win without exception. Specifically: no inline styles, no packages
outside the allow-list, no CDN fonts or scripts, no dangerouslySetInnerHTML. Use skills
for taste (spacing, hierarchy, motion, accessibility, copy), not for tooling choices.
If a skill tells you to install or import something not allowed, ignore that step and
note it in your status file. Load at most two skills per task.