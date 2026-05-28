---
name: recent-digest
description: Use when the user asks to summarize, review, or recap their recent journal notes — phrasings like "summarize my recent notes", "what have I been journaling about lately", "show me my latest entries", "review my notes". Pulls the 10 most-recent notes and groups them by tag. Does NOT promise a time window — see honesty rules below.
---

# Recent Digest

You help the user review what they've journaled most recently.

## What the data actually looks like

`notes://recent` returns markdown previews of the **10 most-recent notes**
(ordered newest-first by `created_at`). Each entry exposes:

- the note `id`
- its `tags` (possibly empty)
- a content preview truncated to **150 characters**

The resource does **not** carry timestamps in the preview. You cannot tell
from the response how many days the 10 notes span — they might all be from
today, or scattered across months if the user journals sparsely.

## Steps

1. Read the resource `notes://recent` from the `journal` MCP server.

2. If the response says "No notes yet", say so plainly and suggest the user
   add a note via the `add_note` tool. Stop here.

3. **Group** the notes by their tags. A note with multiple tags appears in
   more than one group. A note with no tags goes into a `misc` group.

4. **Render** the digest as markdown with this exact shape:

```
# Recent digest — 10 most-recent notes

## <tag-name> (<N> notes)
- [<id>](notes://<id>) — <one-sentence summary of the preview>
- ...

## <next-tag> (<N> notes)
- ...

---
**Reflection prompt:** <one open-ended question, drawn from the dominant tag,
that nudges the user to think about gaps or next steps>
```

5. If the user wants more detail on a note, point them at `notes://<id>` —
   that resource returns the **full content**, not the 150-char preview.

## Honesty rules

- Do **not** frame the digest as "this week" or "last 7 days" — the resource
  carries no timestamps, so you cannot verify any time window.
- Do **not** invoke `add_note` or `delete_note` — this skill is read-only.
- Do **not** invent notes, tags, or summaries; only describe what
  `notes://recent` actually returned. If a preview ends with "…", treat the
  rest as unknown.
- Keep each per-note summary to **one sentence** drawn from the visible
  preview text.
