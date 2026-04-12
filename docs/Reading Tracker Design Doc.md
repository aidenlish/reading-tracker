# Reading Tracker — Design Document

**Author:** Aiden Li
**Date:** 2026-04-11
**Status:** v1.0

---

## 1. Problem Statement

I read broadly across infra/platform engineering, SRE, security, and adjacent topics daily. The problem had three parts:

1. **No persistence** — what I read existed only in browser history or memory.
2. **No organization** — reads were scattered with no topic structure.
3. **No friction tolerance** — any system requiring manual effort (copy-pasting into a spreadsheet, opening a notes app) would be abandoned within days.

The goal: a zero-friction reading tracker that captures what I read, organizes it by topic, and is queryable across sessions — without me thinking about it.

---

## 2. Requirements

### 2.1 Functional Requirements

| ID | Requirement |
|----|-------------|
| F1 | Log a read: title, URL, topic, takeaways |
| F2 | Save to a read-later queue |
| F3 | Query today's reading log |
| F4 | Query read-later list, filterable by topic |
| F5 | Query reading history by single date, date range, or topic |
| F6 | Auto-organize reads by topic |
| F7 | Auto-promote an emerging topic when it reaches a threshold (3 entries) |
| F8 | All state persists across Claude Code sessions |

### 2.2 Non-Functional Requirements

| ID | Requirement | Notes |
|----|-------------|-------|
| NF1 | **Zero friction** — no commands to memorize | Natural language triggers |
| NF2 | **Token efficiency** — minimize context window usage per session | Affects cost and response latency |
| NF3 | **Trigger accuracy** — right skill fires for right intent | Minimize false positives and missed triggers |
| NF4 | **Cross-session persistence** | Works in any Claude Code session without re-explaining context |
| NF5 | **No external runtime dependencies** | Must not depend on MCP servers staying alive |
| NF6 | **Compaction resilience** — skills survive long sessions | Avoid skill content being dropped after context compaction |

---

## 3. Architecture

### 3.1 Storage — Obsidian Vault (Markdown Files)

All state is stored as plain markdown files in the Obsidian vault. Claude interacts with them via native filesystem tools (Read, Write, Edit, Glob).

```
Reading Log/
├── Config.md                ← single source of truth: topics, inference bias, threshold
├── Candidate Topics.md      ← tracks emerging topic counts
├── Read Later.md            ← read-later queue, sectioned by topic
├── Daily/
│   └── YYYY-MM-DD.md        ← one note per day
└── Topics/
    ├── Infra - Platform Engineering.md
    ├── SRE - Reliability.md
    └── [topic].md           ← auto-created on promotion
```

### 3.2 Behavior Layer — Claude Code Skills

Five focused skills in `~/.claude/skills/`, each with its own `SKILL.md`.

```
~/.claude/skills/
├── log-read/         auto-trigger: "I just read / finished..."
├── read-later/       auto-trigger: "save this for later"
├── reading-today/    auto-trigger: "what did I read today?"
├── reading-list/     manual only:  /reading-list
└── reading-history/  manual only:  /reading-history
```

### 3.3 Configuration Split

| Config                      | Location                   | Type    | Reason                                                                                |
| --------------------------- | -------------------------- | ------- | ------------------------------------------------------------------------------------- |
| Vault path                  | Inlined in each `SKILL.md` | Static  | Two lines — a separate file would add a Read call on every invocation with no benefit |
| Topics, inference bias, threshold | `Reading Log/Config.md`    | Dynamic | Grows as topics are promoted; single source of truth                                  |
*Note*: Since we currently only have 2-line static configs, having per-skill config.md is an overkill. Later, if static configs expand, we can consider adding them.

---

## 4. Design Decisions & Tradeoffs

### 4.1 MCP vs. Filesystem Tools

**Decision:** Use native filesystem tools (Read, Write, Edit, Glob). Deny my previously built Obsidian MCP tools except `obsidian_open_note`.

|                              | MCP                             | Filesystem Tools |
| ---------------------------- | ------------------------------- | ---------------- |
| Dependency                   | Requires MCP server running     | None             |
| Capability for this use case | Same                            | Same             |
| Debuggability                | Black box                       | Transparent      |
| Cross-session reliability    | Fragile (server can disconnect) | Always available |

**Verdict:** MCP adds a failure mode with no benefit for local markdown files.

---

### 4.2 Slash Commands vs. Skills vs. CLAUDE.md

Three mechanisms were evaluated for triggering reading tracker behavior:

| Mechanism | Trigger | User effort | Context cost |
|-----------|---------|-------------|--------------|
| Slash commands (`~/.claude/commands/`) | Manual `/command` | Must memorize commands | Loaded only on invocation |
| CLAUDE.md standing instructions | Claude infers from conversation | Zero — fully natural | Always in context (every session) |
| Skills (`~/.claude/skills/`) | Auto via description match + manual `/name` | Zero for common ops | Descriptions at start; full content on invocation only |

**Decision:** Skills. They provide auto-triggering without the constant context cost of CLAUDE.md, and without requiring command memorization.

---

### 4.3 Single SKILL.md vs. Multiple

**Decision:** Split into 5 focused skills.

The decisive factors (from Claude Code docs):

1. **Compaction budget** — re-attached skills share a 25,000-token budget. Each skill gets up to 5,000 tokens. A single large file competes with itself during compaction; 5 small files share the budget gracefully.

2. **Description truncation** — descriptions over 250 characters are truncated. One broad description for 6 operations is imprecise. Five focused descriptions each nail their trigger.

3. **`disable-model-invocation: true`** — query-only skills (reading-list, reading-history) should only fire when explicitly requested. This flag gives zero context cost until invoked. Only possible with separate skills.

---

### 4.4 Static Config vs. Dynamic Config.md

**Decision:** Topics list lives in vault `Config.md`, not hardcoded in skills.

**Problem with hardcoding:** When a candidate topic is promoted, it becomes a named topic. If topics are hardcoded in 5 `SKILL.md` files, all 5 must be updated — fragile and easy to miss.

**Solution:** `Config.md` in the vault is the single source of truth. The promote operation updates it as one of its steps. All skills read it at runtime via Read tool.

**Topic resolution** (inference and querying) relies on the model's semantic judgment against the Named Topics list — no keyword mapping table. The LLM already understands that "k8s" → infra, "postmortem" → SRE, "llm" → AI/ML. A mapping table adds maintenance overhead and a failure mode (unmapped keywords) in exchange for nothing the model can't infer directly.

---

### 4.5 Auto-Promotion Design

When entries filed under "Other" accumulate, Claude infers a candidate topic label and tracks counts in `Candidate Topics.md`. At threshold (3), the promote operation is triggered:

1. Writes a transaction log (`Promotion In Progress.md`) recording the topic and last completed step
2. Creates a new topic file with migrated entries *(additive)*
3. Updates `Config.md` and `Read Later.md` *(additive)*
4. Cleans `Candidate Topics.md` *(minor removal)*
5. Cleans `Other.md` *(destructive — intentionally last)*
6. Deletes the transaction log
7. Notifies the user

Steps are ordered additive-first, destructive-last. Each step is idempotent — safe to re-run. On the next `log-read` invocation, if a transaction log is found, promotion resumes from the last completed step before processing the new read.

This removes the need for the user to manually create topic files or reorganize entries.

**Querying candidate topics before promotion:** If a user queries a topic that hasn't been promoted yet (e.g. "cooking" with only 1 entry), query skills fall back to searching within Other:
- `reading-history`: filters `Topics/Other.md` rows by the Candidate Topic column
- `reading-list`: filters `## Other` items by `[candidate: X]` in the note text

Results are labeled "*(candidate topic — not yet promoted)*" so the user understands the context.

> For full implementation details, see `~/.claude/skills/log-read/SKILL.md`.

---

## 5. Skill Invocation Summary

| Skill | Trigger mode | `disable-model-invocation` | When Claude fires it |
|-------|-------------|---------------------------|----------------------|
| log-read | Auto | false | User mentions finishing/reading something |
| read-later | Auto | false | User says "save for later" / "add to my list" |
| reading-today | Auto | false | User asks what they read today |
| reading-list | Manual only | true | `/reading-list` |
| reading-history | Manual only | true | `/reading-history` |

---

## 6. File Structure Reference

### Reading Log/Config.md (dynamic)
```markdown
## Named Topics
- Infra - Platform Engineering
- SRE - Reliability
- ...

## Inference Bias
User focuses on infra/platform engineering and SRE. Lean toward these when content could fit multiple topics.

## Promotion Threshold
3 entries in a candidate topic triggers auto-promotion to a named topic.
```

Topic resolution (for both logging and querying) uses the model's semantic judgment against the Named Topics list — no keyword mapping table needed.

---

## 7. Known Limitations

- No deduplication — same URL can be logged twice
- Promote operation is not truly atomic — mitigated by: additive-first ordering, idempotency checks, and a write-ahead transaction log that enables self-healing on the next run
