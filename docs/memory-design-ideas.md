# Memory System MVP (OpenClaw-Style Local Markdown)

This document defines a **simple MVP** for local user memory so we can ship fast, reduce risk, and learn from real usage before adding complex logic.

## MVP Principles

- Keep memory in a single local markdown file.
- Prefer predictable behavior over "smart" behavior.
- Update infrequently to avoid noise.
- Keep prompt injection small and deterministic.

---

## 1) Memory File Format (Locked)

Use **Option A** only: strict sectioned markdown.

```md
# User Memory

## Stable Preferences
- Tone: concise

## Formatting Rules
- Use bullet points for summaries.

## Domain Vocabulary
- "PCI" => "percutaneous coronary intervention"

## Recent Context
- Working on cardiology consult notes.

## Do Not Store
- Passwords, API keys, one-time codes, personal identifiers.

## Metadata
- Last updated: 2026-02-14T12:34:56Z
```

No YAML frontmatter for MVP.

---

## 2) When Memory Is Created

Create memory in one of two ways:

1. User clicks **Enable Memory**.
2. Automatic fallback: after **3 completed sessions**.

If neither condition is met, no memory file is created.

---

## 3) When Memory Is Updated

Use one safe rule for MVP:

- Only consider updates at **session end**.
- Run at most **once every 12 hours**.
- Apply at most **1 update per day**.

If the worker has nothing meaningful, skip writing.

---

## 4) Background Flow (Minimal)

1. Client ends a session.
2. Client sends a compact session summary to server.
3. Server LLM returns a **full replacement markdown body for known sections**.
4. Client writes the updated file locally.

MVP intentionally avoids patch/rebase logic.

---

## 5) Injection Back Into Formatting Calls

Keep injection deterministic and small:

- Always include:
  - `Stable Preferences`
  - `Formatting Rules`
- Optionally include up to **3** `Domain Vocabulary` or `Recent Context` bullets most related to the current transcript.
- Hard cap: **~300 tokens** total memory context.

If user gives explicit instructions in current turn, current-turn instruction wins.

---

## 6) Safety Guardrails (MVP)

- Never store anything listed under `Do Not Store`.
- Never inject the entire memory file by default.
- Keep one local backup copy before each write: `user-memory.backup.md`.

---

## 7) What We Deliberately Skip in MVP

To reduce mistakes, we skip these for now:

- patch-based merges
- embeddings-based retrieval pipelines
- confidence scoring systems
- multi-file memory packs
- user diff review UI

These can be added only after real-world feedback.

---

## 8) Suggested Default Values

- Create after 3 sessions (or manual enable).
- Update at session end, max every 12h.
- Max 1 write/day.
- Inject ~300 tokens of memory.

This is intentionally simple and should be the first implementation.
