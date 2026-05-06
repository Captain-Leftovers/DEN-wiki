# AGENTS.md — DAI Wiki

Obsidian vault / personal wiki for the Nix/Den ecosystem. Not a software project — no build, test, or CI commands.

## Structure

- `raw/` — Immutable scraped source material (Den docs). **Never modify.**
- `wiki/` — Concept pages maintained by hand.
  - `wiki/index.md` — Master concept map. Must stay accurate.
  - `wiki/log.md` — Append-only progress log. **Never edit past entries.** (Ignore the stale "Current state" header at the top; trust the latest entries and `index.md` for phase.)
- `.obsidian/` — Obsidian config and plugins. Leave alone unless asked.
- `.trash/` — Obsidian's trash. Ignore.

## Canonical instructions

This file (`AGENTS.md`) supersedes the older `AGENTS.MD` and `CLAUDE..MD`.

## Page format

Every concept page must follow this structure:

```markdown
# Concept Name

**What it is**: One precise sentence.
**Why it exists**: One or two sentences.
**Relates to**: [[concept-a]], [[concept-b]]

---

## Core idea
## Details
## Gotchas
## In the Nix ecosystem
```

## Conventions

- Page names: lowercase with hyphens (`den-as-library.md`).
- `[[wiki-links]]` inline in prose, not just headers.
- Links are bidirectional — when page A links to B, add a backlink in B.
- One concept per page. Dense and precise; this is memory, not a tutorial.
- When a concept spans multiple tools, write one comparative page instead of separate per-tool pages.

## Workflow

The wiki follows a phased workflow (Extract → Synthesize → Grow). Currently in Phase 2/3.
- **Extract**: Read `raw/` files, append findings to `wiki/log.md`. Do not write pages yet.
- **Synthesize**: Write one concept page per session. Update `wiki/index.md` and `wiki/log.md` after each.
- **Grow**: Answer questions from the wiki. Update pages when gaps are found.

## Trial migration workflow

When the user starts a Den migration under `trial/`:
- `trial/old/` — their existing config (read-only, reference only).
- `trial/new/` — the Den conversion (write here).
- Proceed incrementally. Ask questions before making non-obvious choices.
- Typical migration order:
  1. Examine `old/flake.nix`, `old/configuration.nix`, and directory structure.
  2. Set up `new/flake.nix` with `den.flakeModule`.
  3. Map existing hosts → `den.hosts`.
  4. Map existing users → `den.aspects` + `includes`.
  5. Extract reusable features into named aspects.
  6. Verify with `nix flake check` or equivalent.
- Preserve the user's preferred patterns (e.g. namespaces) unless they ask to change them.
- Stop and ask when encountering: host-specific hardware quirks, encrypted secrets, custom builder scripts, or anything that can't be inferred from the files.

### Migration conventions

- **Target Den's idiomatic model first.** When an old pattern conflicts with Den's design (e.g. monolithic host modules vs. feature-first aspects), prefer Den's pattern and explain the change.
- **Pin to stable.** Use `github:vic/den/latest` (release tags) instead of `main` unless the user explicitly asks for bleeding-edge.
- **One concern per aspect.** Extract reusable features (bluetooth, dev-tools, gaming) into standalone named aspects in `den.aspects` rather than nesting them inside host modules.
- **Namespaces for sharing.** If the user wants to share configs across machines or users, set up `den.namespace` early; it is easier than retrofitting later.
- **Default template baseline, example features added incrementally.** Start from the `default` template structure, then add `example`-template features (namespaces, providers, angle brackets) as needed.
- **When in doubt, restructure.** Scattered legacy modules are normal in old configs. reorganize them into aspects during migration; do not maintain the old layout for nostalgia's sake. The user has explicitly approved this.
- **Remove old branding/nicknames.** If the old config contains Zaney OS nicknames, branding, or personal identifiers, strip them out during migration. Rebrand with the user's own names or remove entirely.
- **Keep `old/` untouched.** Never modify the source config. The migration is a rewrite into `new/`, not an in-place refactor.
- **Track progress.** Update `trial/MIGRATION.md` after each step so another agent can resume if interrupted.

## Lint

When asked to lint:
- Orphan pages (no inbound links).
- Concepts in log with no page.
- Pages missing required format sections.
- Broken `[[links]]` pointing to nonexistent pages.
