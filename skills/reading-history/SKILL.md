---
name: reading-history
description: Show the user's reading history by date range or topic. Use when the user asks what they read this week, last week, on a specific date, or about a specific topic like SRE or infra.
disable-model-invocation: true
---

**Vault base:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/`
**Vault config:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/Config.md`

---

Parse the user's request:
- No filter → last 7 days
- Single date like `2026-04-01` → only that date
- Date range like `2026-04-01 to 2026-04-07` or `2026-04-01..2026-04-07` → all dates in range
- "this week" / "last week" → compute range from context date
- Topic keyword → read vault `Config.md` for the Named Topics list. Use semantic judgment to match the keyword to the closest named topic.
  - If matched to a **named topic**: read `Topics/[Topic].md` and display the table.
  - If no named topic matches (e.g. "cooking" — a candidate not yet promoted): read `Topics/Other.md` and filter rows where the Candidate Topic column matches the keyword semantically. Display only those rows, with a note: "*(candidate topic — not yet promoted)*"
  - If nothing found in either case, say so and list available named topics.

**Date-range query:**
Use Glob with pattern `Daily/*.md`, path `[vault base]`. Filter filenames to date range. Read all matching files in parallel using multiple simultaneous Read tool calls. Display grouped by date, newest first.

**Topic query:**
Read `[vault base]Topics/[Topic].md`. Display the table.

End with "**X item(s) found.**"
