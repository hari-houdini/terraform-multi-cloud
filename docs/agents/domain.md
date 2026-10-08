# Domain documentation

This repo uses a **single-context** layout for domain documentation.

## Where to put it

- **`CONTEXT.md`** at the repo root: domain knowledge, architecture decisions, and consumer rules (required)
- **`docs/adr/`**: architecture decision records, one `.md` file per decision (optional, but recommended)

## How agents read it

Skills like `to-spec`, `to-tickets`, and `codebase-design` read `CONTEXT.md` first. They use it to understand:

- What you're building and why
- Key constraints and non-goals
- The current architecture shape
- Design principles and trade-offs
- Where critical code lives and what makes it critical

If the repo grows to have multiple independent domains (e.g., a monorepo with separate packages, each with distinct concerns), you can graduate to a **multi-context** layout: rename the root `CONTEXT.md` to `CONTEXT-MAP.md` (a guide to all domains), and add per-context `CONTEXT.md` files alongside each domain's code.

## Next steps

1. Create `CONTEXT.md` at the repo root. Start with a one-paragraph summary of the project, then add sections on architecture, critical code paths, and key decisions.
2. (Optional) Create `docs/adr/` and add architecture decision records as `.md` files (e.g., `0001-terraform-multi-cloud-strategy.md`).

Agents will read these files automatically once they exist.
