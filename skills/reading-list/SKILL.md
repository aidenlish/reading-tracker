---
name: reading-list
description: Show the user's read-later queue. Use when the user asks what's in their reading list, what they should read next, what's queued, or wants to browse saved items by topic.
disable-model-invocation: true
---

**Vault base:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/`
**Vault config:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/Config.md`

---

1. Read `[vault base]Read Later.md` with Read tool.

2. Display only unchecked items (`- [ ]`), grouped by section. Skip empty sections.

3. If user specified a topic filter: read vault `Config.md` for the Named Topics list. Use semantic judgment to match the filter to the closest named topic.
   - If matched to a **named topic**: show only that `## [Topic]` section.
   - If no named topic matches (e.g. "cooking" — a candidate not yet promoted): show items from `## Other` whose note contains `[candidate: X]` where X matches the filter semantically. Label results with "*(candidate topic — not yet promoted)*"
   - If nothing found in either case, say so and list available named topics.

4. End with "**X item(s) queued.**"

5. If all empty: "Your Read Later list is empty."
