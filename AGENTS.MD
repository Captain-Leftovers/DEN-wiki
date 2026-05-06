# DAI Wiki

A personal knowledge base maintained by DAI.
Based on Andrej Karpathy's LLM Wiki pattern.

## Purpose

This wiki is DAI's persistent memory for the Nix ecosystem.
It exists so DAI can answer questions about Nix, NixOS, Home Manager, flakes, Den, and related frameworks quickly and reliably — without re-reading raw sources every time.

The wiki is organized around **concepts**, not source files.
Pages are connected with `[[wiki-links]]` inline in the text, forming a traversable knowledge graph — not a flat list of summaries.

---

## Folder structure

```
raw/          -- source documents (immutable, never modify)
wiki/         -- concept pages maintained by DAI
wiki/index.md -- master concept map, grouped by theme
wiki/log.md   -- append-only progress log
```

---

## Three phases

### Phase 1 — Extract

Read raw source files one at a time. Do not write concept pages yet.
For each file, extract every concept encountered and record it in `wiki/log.md` under an extraction entry.

A concept is anything that:
- Has a name and a distinct meaning
- Could be the subject of a question
- Relates to at least one other concept

After each file, append to `wiki/log.md`:
- The file processed
- Every concept found, one line each, with a one-sentence description
- Any relationships spotted between concepts (e.g. "aspects use parametric dispatch")
- The next file to process

Do not create wiki pages during Phase 1. Only populate the log.
Stop after each file and wait for the human to say "continue".

### Phase 2 — Synthesize

Once all raw files are extracted, review `wiki/log.md` to see all concepts and their relationships.
Then write concept pages one at a time — one page per session.

Each concept page:
- Covers exactly one concept
- Synthesizes all sources that mention it (not just one file)
- Uses `[[wiki-links]]` inline in the prose wherever a related concept is mentioned — not just at the bottom
- Links bidirectionally: when page B links to page A, update page A to link back to page B
- Follows the page format below

After writing each page:
- Update `wiki/index.md`
- Append to `wiki/log.md`
- Stop and wait for "continue"

### Phase 3 — Grow

Once the core concept pages exist, the wiki deepens through use.
When the human asks a question:
1. Read `wiki/index.md` to find relevant pages
2. Read those pages
3. Answer directly from the wiki — precise, no fluff
4. If the answer reveals a gap or adds nuance not captured yet, update the relevant page
5. If a genuinely new concept emerges, offer to add it

The wiki grows from real questions, not from re-reading sources.

---

## Page format

Every concept page must follow this structure:

```markdown
# Concept Name

**What it is**: One precise sentence.

**Why it exists**: One or two sentences — what problem this concept solves.

**Relates to**: [[concept-a]], [[concept-b]], [[concept-c]]

---

## Core idea

The essential explanation. Lead with the mental model, not the mechanics.
Use a concrete example to ground it.
Embed `[[wiki-links]]` inline wherever a related concept appears in the prose.

## Details

Deeper facts, edge cases, full mechanics.
Keep it dense — this is memory, not a tutorial.

## Gotchas

Non-obvious things. What the docs bury. What trips people up.

## In the Nix ecosystem

How this concept relates to the broader Nix world.
Compare to similar concepts in other tools when relevant.
```

---

## Index format

`wiki/index.md` is the master concept map.
Group pages by theme. Each entry is one line:

```
- [[page-name]] — one sentence on what concept this covers
```

Themes reflect the natural structure of the Nix ecosystem, not any one tool's doc layout.
Suggested starting themes: Nix Fundamentals, The Module System, Flakes, Den Core Concepts, Den Framework, Home Environments, Ecosystem & Tooling.

The index is also the first thing DAI reads when answering a question — it must stay accurate and complete.

---

## Log format

`wiki/log.md` is append-only. Never edit past entries.

**Phase 1 extraction entry:**
```
## YYYY-MM-DD — Extract: <filename>

### Concepts found
- **concept-name**: one sentence description
- **concept-name**: one sentence description

### Relationships spotted
- concept-a → concept-b: how they relate
- concept-c → concept-a: how they relate

### Next file
`path/to/next/file.txt`
```

**Phase 2 synthesis entry:**
```
## YYYY-MM-DD — Synthesize: <concept-name>

**Page created**: `wiki/page-name.md`
**Sources drawn from**: list of raw files that informed this page
**Pages linked to**: list of [[wiki-links]] used in the page
**Pages updated with backlink**: list of pages that were updated to link back

### Next concept to synthesize
<concept-name>
```

**Phase 3 growth entry:**
```
## YYYY-MM-DD — Grow: <question or topic>

**Pages read**: list
**Pages updated**: list with one line on what changed and why
```

---

## Linking rules

These rules are what make the wiki a knowledge graph rather than a flat list:

1. Every concept mentioned by name in a page body must be `[[linked]]` on first mention
2. Links go both ways — when you add a link from A to B, add a link from B to A
3. The `Relates to` header field lists the most important connections at a glance
4. A page with no inbound links (nothing links to it) is an orphan — flag it during lint
5. Prefer linking to a specific section with `[[page#section]]` when the relationship is to a sub-concept

---

## Retrieval behavior

When the human asks a question:
1. Read `wiki/index.md` — find the relevant concept(s)
2. Read those pages
3. Answer from the wiki — direct and precise
4. If not covered, say so clearly and name the raw file that would contain it
5. Never guess or hallucinate facts not in the wiki or raw sources

---

## Lint

When asked to lint:
- List orphan pages (no inbound links)
- List concepts in the log that still have no page
- List pages missing required format sections
- List `[[links]]` that point to pages that don't exist yet
- Report as a numbered list with suggested fixes

---

## Rules

- Never modify anything in `raw/`
- One file per session during Phase 1, one concept per session during Phase 2
- Always update `wiki/index.md` and `wiki/log.md` after every session
- Page names lowercase with hyphens (e.g. `den-aspects.md`)
- `[[wiki-links]]` inline in prose, not just in a footer list
- Bidirectional links — always update both sides
- Dense and precise beats friendly and padded — this is memory, not documentation
- When a concept appears across multiple tools, one comparative page beats separate per-tool pages
