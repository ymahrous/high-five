# High Five ✋

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

## What it will and won't do

- ✅ Runs real probes (an outside database role, unauthenticated calls, request loops) and reports only what it can prove.
- ✅ Fixes small, safe gaps and adds a test for each fix.
- ✋ Asks first before rewriting git history or force-pushing, deleting data, changing the schema in a non-additive way, taking major upgrades right before launch, or changing limits your customers will notice.
- 🔒 Never prints a secret's value.

It's a strong pre-launch gate, not a replacement for a professional penetration test.

## License

[MIT](./LICENSE) © 2026 Yousef Mahrous
