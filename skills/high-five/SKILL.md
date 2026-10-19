---
name: high-five
description: "Pre-launch security gate for web apps: checks RLS on every table, server-side auth and validation, secrets in code and git history, dependencies, and rate limits, then fixes gaps and reports."
---

# High Five — five checks before you go live

Use this skill when someone is about to launch ("go live", "pre-launch checklist", "is it safe to ship?", "launch checklist") or asks for any of the five checks below. Each check ends with **evidence**, never with "looks fine". Claim a pass only after running something that would have failed.

The five checks:

1. **RLS on every table** — row-level security is on, and no role outside the app can read rows.
2. **Checks live in the backend** — auth, authorisation and validation happen on the server, not only in the UI.
3. **No secrets in the code or git history** — including deleted secrets in old commits and the client bundle.
4. **Dependencies up to date and known** — no known vulnerabilities, nothing unknown or suspicious.
5. **Rate limits on APIs and login** — every entry point has a limit that can't be dodged.

## Step 0 — Map the project (2 minutes)

- Stack: read `package.json` / `requirements.txt` / `pyproject.toml` / `Gemfile` / `go.mod` / `Cargo.toml`.
- Database and access model: look for `supabase/`, `prisma/schema.prisma`, `drizzle`, migrations, `firebase.json`, `firestore.rules`, connection strings in `.env.example`. Note whether the **browser talks to the database directly** (Supabase anon key, Firebase client SDK, PostgREST, Neon Data API) or only the server does. That decides how strict check 1 must be.
- Entry points: API routes, server actions, RPC/GraphQL resolvers, edge/cloud functions, webhooks, cron jobs.
- Is it a git repo? (`git rev-parse --is-inside-work-tree`). Is there a test suite and a way to run the app locally?
- Existing helpers: if the repo already has scripts such as `check:secrets`, `check:deps`, or tests named `rls`, `security`, `ratelimits`, use them and say so.

Use a task list with one item per check plus "Re-run and report".

## Check 1 — RLS on every table

**Postgres / Supabase / Neon** — list every table and its RLS state:

```sql
SELECT n.nspname AS schema, c.relname AS table, c.relrowsecurity AS rls, c.relforcerowsecurity AS forced
FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','p') AND n.nspname NOT IN ('pg_catalog','information_schema') AND n.nspname NOT LIKE 'pg_%'
ORDER BY 1, 2;

SELECT schemaname, tablename, policyname, roles, cmd, qual, with_check FROM pg_policies ORDER BY 1, 2;
```

Fail if any table has `rls = false`, or if a policy grants `anon` / `authenticated` / `public` a blanket `USING (true)` on data that isn't meant to be public (that's RLS switched on but doing nothing). In Supabase also check Storage buckets and that the `service_role` key is never shipped to the client.

**If the browser talks to the database:** every table needs real per-user policies (e.g. `auth.uid() = user_id`) for SELECT, INSERT, UPDATE and DELETE, and `WITH CHECK` on writes.

**If only the server talks to the database:** still enable default-deny RLS on every table as defence in depth, with one policy for the app's role, so a leaked read-only credential, a BI tool or an exposed data API sees nothing:

```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;          -- for every table (loop over pg_tables in a DO block)
CREATE ROLE app_rw NOLOGIN;                          -- group role for the app
CREATE POLICY app_all ON <t> TO app_rw USING (true) WITH CHECK (true);
-- owner role bypasses RLS; a least-privilege login can be: CREATE ROLE app_web LOGIN PASSWORD '…' IN ROLE app_rw;
```

Put it at the end of the migrations so new tables are always covered, and add a test that fails if any table lacks RLS.

**Firebase:** `firestore.rules` / `storage.rules` must not contain `allow read, write: if true` or rules without `request.auth` checks on private data.

**Prove it:** connect (or call the REST/data API) as a role outside the app — e.g. a throwaway role granted `SELECT` — and show it gets **0 rows**; then show the app still works under its own role.

## Check 2 — Checks live in the backend

List every server entry point and confirm, for each one: authenticated? authorised (role **and** ownership of the record)? input validated on the server (schema, lengths, enums, unknown fields rejected)? Useful searches:

- Next.js: `"use server"` files, `app/**/route.ts`, `pages/api/**`; never rely on `middleware`/`proxy` alone (CVE-2025-29927 showed middleware can be bypassed) — each page/action/route must check for itself.
- Express/Fastify/Hono: `router.(get|post|put|patch|delete)`; Django/DRF: views without `permission_classes`; Rails: controllers without `before_action :authenticate`.

Red flags: role checks only in JSX (`{isAdmin && <Button/>}`), validation only in the form, `userId`/`role`/`price` taken from the request body, `findById(req.params.id)` without an owner filter (IDOR), mass assignment (`update(req.body)`), admin endpoints protected by obscurity.

**Prove it:** call each mutating endpoint (1) with no session, (2) as a normal user, (3) as user A on user B's record. Expect 401/403/404 and no data change. Best: a test that **enumerates every exported action/route** and asserts the refusal, so new endpoints are covered automatically. Also try a cross-origin POST to confirm CSRF protection.

## Check 3 — No secrets in the code or git history

Scan the working tree **and every commit** (a secret deleted later is still in history):

```bash
gitleaks detect --source . --log-opts="--all" --redact          # preferred
trufflehog git file://. --only-verified                          # alternative
# fallback without tools:
git log -p --all | grep -nE 'AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY|sk_live_|ghp_[A-Za-z0-9]{36}|xox[abprs]-|AIza[0-9A-Za-z_-]{35}|postgres(ql)?://[^:]+:[^@]+@'
```

Also check:
- `.gitignore` covers `.env` and `.env.*` (keep `!.env.example`); `.env.example` holds placeholders only.
- The **client bundle**: anything in `NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`, `EXPO_PUBLIC_*` is public. Grep the build output (`.next/static`, `dist/`) for keys. Supabase `service_role` must never appear there.
- Dockerfiles, CI configs, test fixtures and seed files.

If a real secret is found: **rotate it first** (removing it from git does not un-leak it), then purge history (`git filter-repo` / BFG) — rewriting history and force-pushing need the user's explicit go-ahead. Add a pre-commit hook or CI step that runs the scanner. When showing findings, redact values.

If the folder isn't a git repo yet, scan the files, say history couldn't be checked, and leave a scan the user can run once it is.

## Check 4 — Dependencies up to date and known

```bash
npm audit --omit=dev && npm audit          # or: pnpm audit / yarn npm audit / pip-audit / bundle audit / govulncheck ./... / cargo audit
npm outdated
npm audit signatures                       # registry signatures & provenance
npm query ":attr(scripts, [postinstall]), :attr(scripts, [preinstall]), :attr(scripts, [install])"
```

Pass when: 0 known vulnerabilities in production dependencies (dev-only issues explained), a lockfile is committed with integrity hashes, every package resolves from the official registry (no stray git/tarball URLs), direct dependencies are well-known (check `npm view <pkg> maintainers time repository` for anything unfamiliar or typo-like), install scripts are understood, and the runtime (Node/Python) is a supported LTS.

Update patch and minor versions; take a major only after typecheck, tests and build pass on it. Keep type packages (e.g. `@types/node`) on the runtime's major. Don't do risky upgrades the day before launch — report them instead.

## Check 5 — Rate limits on APIs and login

Build an inventory table: every entry point → its limit → what it's keyed on. Must include: sign-in, sign-up, password reset / OTP / magic link (email bombing), API key auth failures, public APIs, expensive endpoints (exports, uploads, AI or third-party calls), contact/waitlist forms, webhooks.

Check the limiter itself:
- **Shared storage** (Postgres/Redis/edge KV) — in-memory limits reset per instance and are useless on serverless.
- **Correct identity:** behind a proxy, the client IP comes from a trusted number of `X-Forwarded-For` hops (spoofed headers must not reset limits); group IPv6 by /64; API quotas per **account**, not per key (extra keys must not multiply a plan).
- **Login:** per-IP throttle + per-account lockout/backoff, identical errors for unknown user and wrong password, uniform timing.
- Responses: 429 with `Retry-After` (and ideally `RateLimit` headers); password hashing has bounded concurrency so a login flood can't exhaust memory.

**Prove it:** loop requests until a 429 appears, for login and for each API; repeat with a spoofed `X-Forwarded-For` and show it doesn't help. Add one test per limit.

## Fixing

Fix gaps that are small and safe right away, each with a regression test (RLS-on-every-table test, endpoint enumeration test, secret-scan script, per-limit tests). Ask first before: non-additive schema changes, deleting data, rewriting git history or force-pushing, major-version upgrades close to launch, or changing limits that customers will notice. After fixing, re-run every check from scratch and run the project's typecheck, tests and build.

## Report

Finish with a short report in the user's language:

| Check | Result | Evidence | Fixed now | Left to do |
|---|---|---|---|---|
| RLS on every table | ✅ / ⚠️ / ❌ | e.g. "13/13 tables; outside role saw 0 rows" | … | … |

Use ✅ only with evidence, ⚠️ for "passes but needs an operator step" (e.g. rotate a key, enable a CI scan, run `npm audit signatures` where the network allows), ❌ for open problems. List the commands to re-run the checks, and record the results in the repo's security doc if it has one (e.g. `SECURITY.md`). Never paste a secret's value.