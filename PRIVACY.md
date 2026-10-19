# Privacy policy: High Five

Last updated 2026.

High Five is a Claude plugin that contains one skill: written instructions that tell Claude how to run five security checks on your project. It has no servers, hooks, MCP servers, scripts, analytics or tracking.

## What we collect

Nothing. The author of High Five receives no data from you or your project, and the plugin stores nothing.

## What happens when you use it

When you ask for a check, Claude reads your project's code, configuration, git history and database metadata, and runs commands in your environment with your usual permission prompts. Your conversation with Claude is covered by Anthropic's privacy policy: https://www.anthropic.com/legal/privacy

## Outside services the checks may contact

- Dependency audits (for example `npm audit`) send package names and versions to your project's package registry or advisory database.
- The default secret scanner, gitleaks, runs entirely on your machine. If trufflehog is used instead with `--only-verified`, each secret it finds is tested against the service that issued it, to confirm whether it is still live.
- Endpoint and rate-limit tests send requests only to the copy of your app that you point it at.

Secret values are never printed in reports.

## Changes

Changes to this policy are published in this file, and the git history shows every change.

## Contact

Open an issue: https://github.com/ymahrous/high-five/issues
