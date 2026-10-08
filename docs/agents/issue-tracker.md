# Issue tracker

This repo tracks work via **GitHub Issues**.

Skills that read this file—like `to-tickets`—will call `gh issue create` and read open issues via the GitHub CLI. You'll need:

1. `gh` installed
2. A GitHub token in `$GITHUB_TOKEN` or `~/.config/gh/hosts.yml`

## Settings

| Setting | Value |
| --- | --- |
| Tracker | GitHub Issues |
| Owner/repo | `hari-houdini/terraform-multi-cloud` |
| PRs as requests | OFF |

## PRs as a request surface

By default, PRs are not surfaced as requests to agents. To change this, edit the setting above to `ON`. Agents will then present open PRs in the triage queue, as a request type.
