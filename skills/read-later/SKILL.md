---
name: read-later
description: Save something to the user's read-later list. Use when the user says "save this for later", "add to my list", "read later", or wants to queue a resource without reading it now.
---

**Vault base:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/`
**Vault config:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/Config.md`

---

1. Extract: title, URL, optional note. Read vault `Config.md` for the current Named Topics list and inference bias. Use semantic judgment to match to the closest named topic. If Other, also infer a candidate topic.

2. Read `[vault base]Read Later.md` with Read tool.

3. Check if `## [Topic]` section exists in the file content.
   - If it **does not exist**: Edit to append to the end of the file:
     ```

     ## [Topic]

     - [ ] [Title](URL) — [note] *(added YYYY-MM-DD)*
     ```
   - If it **exists**: Edit to insert the new item on the line immediately after the `## [Topic]` heading:
     ```
     - [ ] [Title](URL) — [note] *(added YYYY-MM-DD)*
     ```
   No URL: use `- [ ] Title` without a link.
   If Other, append candidate to note: `[note] [candidate: X]`

4. Confirm: "Added **[Title]** to Read Later under **[Topic]**."
