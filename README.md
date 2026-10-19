# High Five

**Five security checks before you go live.** A Claude skill that audits a web app, proves each check with evidence, fixes the small gaps (with a regression test for each), and hands you a pass / warn / fail report.

| # | Check | What "pass" means |
|---|---|---|
| 1 | **RLS on every table** | Row-level security is on everywhere, and a role outside the app reads 0 rows |
| 2 | **Checks live in the backend** | Every endpoint enforces auth, ownership and validation on the server — not just in the UI |
| 3 | **No secrets in code or git history** | Nothing in the files, the full commit history, or the client bundle |
| 4 | **Dependencies up to date and known** | 0 known vulnerabilities, hash-pinned lockfile, no unknown or suspicious packages |
| 5 | **Rate limits on APIs and login** | Every entry point has a limit that holds — even with spoofed IP headers |

Works with Next.js, Supabase, Neon / Postgres, Firebase, Express, Django, Rails and more.

## Install

### Claude Code

```bash
claude plugin marketplace add ymahrous/high-five
claude plugin install high-five@high-five
```

Or from inside a session: `/plugin install high-five --marketplace ymahrous/high-five`.

To install only the skill, copy `skills/high-five/` to `~/.claude/skills/high-five/`.

### Claude.ai (Pro, Max, Team and Enterprise)

1. Download **`high-five.zip`** from the [latest release](https://github.com/ymahrous/high-five/releases/latest).
2. Make sure **Code execution and file creation** is on in **Settings → Capabilities**.
3. In the **Skills** section, choose **Upload skill** and pick the zip.

## Use it

Ask Claude something like:

- "Run the high five on this repo"
- "Pre-launch checklist before we go live"
- "Is this safe to ship?"

In Claude Code you can also call it directly: `/high-five:high-five`.

## Example use cases

- **The day before launch:** "Run the high five — we go live tomorrow." → All five checks with evidence, small gaps fixed with a test each, and a pass / warn / fail report.
- **A Supabase app built fast:** "Check RLS on all my Supabase tables." → Every table's RLS status and policies, blanket `USING (true)` policies flagged, and proof that an outside role reads 0 rows.
- **An app written mostly with AI:** "Is the backend actually checking permissions, or only the UI?" → Every endpoint called with no session, as a normal user, and as one user on another user's record. Gaps are fixed with a test each.
- **Before making a repo public:** "Scan the whole git history for secrets before I open-source this." → Every commit, file and the client bundle scanned. Found secrets are rotated first, and history is rewritten only with your go-ahead.
- **After adding a feature:** "I added an export endpoint — check it." → Login and permission checks, input validation and a rate limit for it, plus a test.
- **Dependency clean-up:** "Are our dependencies safe and up to date?" → Vulnerabilities, outdated or look-alike packages and install scripts reviewed. Safe updates applied, risky ones reported.
- **Before every release:** "Run the high five again." → Every check re-run from scratch, with results saved in `SECURITY.md` if the project has one.

## What it runs and what it sends

High Five is one skill, written instructions for Claude, with no hooks, servers or scripts. Claude runs each step in your project when you ask, with your normal permission prompts. Dependency audits (for example `npm audit`) send package names and versions to your package registry. If trufflehog is used with `--only-verified`, each secret it finds is checked against the service that issued it. Endpoint and rate-limit tests only hit the copy of your app you point it at. Nothing else leaves your machine, and secret values are never printed. See [PRIVACY.md](./PRIVACY.md).

## What it will and won't do

- ✅ Runs real probes (an outside database role, unauthenticated calls, request loops) and reports only what it can prove.
- ✅ Fixes small, safe gaps and adds a test for each fix.
- ✋ Asks first before rewriting git history or force-pushing, deleting data, changing the schema in a non-additive way, taking major upgrades right before launch, or changing limits your customers will notice.
- 🔒 Never prints a secret's value.

It's a strong pre-launch gate, not a replacement for a professional penetration test.

## License

[MIT](./LICENSE)
