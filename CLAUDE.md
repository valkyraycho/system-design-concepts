# Learning Vault — System Design & Reliability

Ray's personal learning vault. Purpose: embed system design / reliability / scalability concepts through Socratic teaching, active recall, and case studies.

## Key files

- `curriculum.md` — the full 17-module curriculum (4 arcs) with case studies and session mechanics. Read this first.
- `dashboard.md` — current position, recall queue, weak spots, session log. **Update after every session.**
- `concepts/` — one atomic note per concept. **Written by Claude only after Ray demonstrates understanding** (quote his framing where good). Shaky understanding → weak-spot list on dashboard, no note yet.
- `case-studies/` — one note per incident/architecture dissected.
- `reps/` — capstone design-drill writeups + critique (after each arc).
- `sessions/` — optional per-session transcript summaries.

## Conventions

- Use `[[wikilinks]]` for cross-references, never relative paths.
- Frontmatter on every note: `type` (concept | case-study | rep | dashboard | curriculum), `status`, `created`, `updated`, `tags`, `related`.
- Concept notes: one idea per file, kebab-case filenames (e.g. `retry-storm.md`, `error-budget.md`).
- Ray answers questions in any language/format — retrieval is about the concept, not the prose. Claude writes the polished notes.

## Session protocol

1. Read `dashboard.md` to find current position and recall queue.
2. If 3+ days since last session-log entry → start with a short recall block before new material.
3. Modes: **Learn** (next unit, Socratic), **Recall** (quiz from queue), **Rep** (post-arc scenario drill). Entry points: "continue", "quiz me", "I have 15 minutes", "explain X again".
4. End of session: update dashboard (position, recall queue, weak spots, session log line).
