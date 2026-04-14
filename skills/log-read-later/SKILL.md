---
name: log-read-later
description: Log a read-later item the user has now finished reading. Use when the user says they finished, completed, or read something that was previously saved in their read-later list — e.g. "I finished [article]", "I read the CQRS post", "mark [X] as read", "done with [title/URL]". Always use this skill instead of log-read when the item is likely already in the Read Later list.
---

**Vault base:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/`
**Vault config:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/Config.md`

---

0. **Resume check** — Read `[vault base]Promotion In Progress.md`. If it exists, run Promote Topic (resume mode) before doing anything else.

1. **Find the item in Read Later** — Read `[vault base]Read Later.md`. Locate the item by matching the URL or title the user provided. Extract:
   - **Title** — from the link text in the checklist item
   - **URL** — from the markdown link
   - **Topic** — from the `## [Topic]` section heading it lives under
   - **Candidate topic** — from the `[candidate: X]` annotation, if present (means topic is Other)
   - **Takeaways** — from the user's message, or `—` if none given

   If the item isn't found, tell the user and stop. Don't proceed with a guess.

2. **Daily note** — `[vault base]Daily/YYYY-MM-DD.md` (today's date from context).
   Read with Read tool. Write fresh if missing, Edit to append if exists.
   Named topic entry:
   ```
   ### [Title]
   - **URL:** [url]
   - **Topic:** [topic]
   - **Takeaways:** [takeaways]
   ```
   Other entry adds:
   ```
   - **Candidate Topic:** [candidate topic]
   ```

3. **Topic table** — `[vault base]Topics/[Topic].md`.
   Read with Read tool. Write fresh if missing, Edit to append a row if exists.
   Named topic table:
   ```
   # [Topic] — Reading Log

   | Date | Title | URL |
   |------|-------|-----|
   | YYYY-MM-DD | [Title] | [url] |
   ```
   Other table includes extra column:
   ```
   # Other — Reading Log

   | Date | Title | URL | Candidate Topic |
   |------|-------|-----|----------------|
   | YYYY-MM-DD | [Title] | [url] | [candidate topic] |
   ```

4. **Remove from Read Later** — Edit `[vault base]Read Later.md` to delete the checklist line for this item.

5. **If Other — update `[vault base]Candidate Topics.md`:**
   Read it. Increment count if candidate exists, else append `| [candidate topic] | 1 |`.
   If updated count >= promotion threshold from Config.md, run Promote Topic before confirming.

6. Confirm: "Logged **[Title]** under **[Topic]** and removed from Read Later[, candidate: [candidate topic]]."

---

## Promote Topic (auto-runs when candidate hits threshold)

Let `T` = the candidate topic being promoted.
**Vault base:** `/Users/ldy/Documents/aiden-li_obsidian/aiden-li_obsidian/Reading Log/`

### Step 0 — Resume check (transaction log)
Read `[vault base]Promotion In Progress.md` if it exists.
- If found: read the last completed step number and resume from the next step.
- If not found: write it now with:
  ```
  # Promotion In Progress
  Topic: [T]
  Last completed step: 0
  Started: YYYY-MM-DD
  ```
After each step below, Edit this file to update `Last completed step: [N]`.

---

Steps are ordered **additive first, destructive last** to minimize data loss on failure.

**Step 1 — Create new topic file** *(additive)*
Check if `[vault base]Topics/[T].md` already exists (idempotency).
- If it does not exist: read `[vault base]Topics/Other.md`, collect all rows where Candidate Topic = T, then Write `[vault base]Topics/[T].md`:
  ```
  # [T] — Reading Log

  | Date | Title | URL |
  |------|-------|-----|
  | ...migrated rows... |
  ```
- If it already exists: skip (already done in a previous run).

**Step 2 — Update Config.md** *(additive)*
Read `[vault base]Config.md`.
- If `- [T]` is not already in the Named Topics list, Edit to add it.

**Step 3 — Update Read Later.md** *(additive)*
Read `[vault base]Read Later.md`. If `## [T]` section does not already exist, Edit to append it (empty).

**Step 4 — Clean Candidate Topics.md** *(minor removal)*
Read `[vault base]Candidate Topics.md`. If T's row still exists, Edit to delete it.

**Step 5 — Clean Other.md** *(destructive — last)*
Read `[vault base]Topics/Other.md`. If the migrated rows still exist, Edit to remove them.

---

### Step 6 — Delete transaction log
Delete `[vault base]Promotion In Progress.md` using Bash: `rm "[vault base]Promotion In Progress.md"`

Notify: **New topic promoted: "[T]" created with [N] entries migrated from Other.**

---

### Resume logic
At the start of every log-read-later run, before step 1, check if `[vault base]Promotion In Progress.md` exists. If it does, run Promote Topic immediately (it will resume from the last completed step) before processing the new read.
