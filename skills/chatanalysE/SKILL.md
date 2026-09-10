---
name: chatanalysE
description: Convert a WhatsApp group export into a prioritised, searchable issue backlog with an HTML triage board. Use when the user says 'analyze my WhatsApp support chat', 'extract issues from WhatsApp export', 'turn WhatsApp messages into a task list', or attaches/references a WhatsApp chat ZIP or _chat.txt file.
sources: [cowork]
---

# WhatsApp Support Chat -> Issue Backlog

This skill turns a WhatsApp group export (`.zip` or `_chat.txt`) into:

1. **Structured message dataset** — every message parsed and theme-classified
2. **Contract/entity reference list** — IDs mentioned in the chat
3. **Prioritised task backlog** — CSV + JSON, importable into Jira/Linear
4. **Executive brief** — markdown summary of key findings
5. **Searchable HTML triage board** — filterable by severity, status, module, category

---

## When to trigger this skill

User says any of:
- "analyze my WhatsApp support chat"
- "extract issues from WhatsApp export"
- "turn this WhatsApp chat into a task list"
- "build a backlog from our WhatsApp group"
- Attaches or points to a WhatsApp export ZIP or `_chat.txt`

---

## FIRST: Compare timelines, identify gaps, then ask how to proceed

### Step A — Parse the new export headers only (fast scan, no full processing)
Quickly read the new export to get its full date range: `new_export_start` and `new_export_end`.

### Step B — Check for existing analysis
Look for `output/messages.json` in the project folder or session context. If found, read it to get: `existing_start` (earliest message date) and `existing_end` (latest message date).

### Step C — Identify what's missing
Compare the two ranges and find the uncovered portion(s):

```
Example:
  Existing analysis: 15 Jun 2025 -> 10 Sep 2026
  New export:        01 Jan 2021 -> 10 Sep 2026
  Missing gap:       01 Jan 2021 -> 14 Jun 2025  (pre-join history)
```

The missing portion could be:
- **Before** existing range (e.g. group history before you joined)
- **After** existing range (new messages since last run)
- **Both** (first-time user with a longer export)
- **None** (fully overlapping — nothing new to process)

### Step D — Always present options before doing anything

Report what was found and ask:

> "I can see the following:
> - Already analyzed: [existing_start] to [existing_end] ([N] messages, [N] tasks in backlog)
> - New export covers: [new_export_start] to [new_export_end]
> - Missing / not yet analyzed: [gap description]
>
> How would you like to proceed?
> 1. **Process missing only** — analyze the [N] messages in the gap and merge into the existing backlog
> 2. **Full re-analysis** — reprocess everything from scratch (replaces existing results)
> 3. **Show existing results only** — open the current board without processing anything"

Wait for the user's choice. Only if the user explicitly says "redo" or picks option 2 will everything be reprocessed from scratch.

**Special case — fully overlapping (no gap):**
> "This export covers [dates], which is fully covered by the existing analysis. Nothing new to process. Want to see the current board?"

---

## Processing missing gap only (user picks option 1)

1. Parse the new export but **filter to only messages within the missing date range(s)** — skip any dates already in `output/messages.json`
2. Assign new message IDs continuing from max existing ID (for future messages) or a separate block for historical messages
3. Run Stages 2–5 on the gap messages only
4. **Merge into existing backlog** using two-pass deduplication across old + new issues
5. For matches with existing tasks: update `first_raised` if the historical instance is older, increment `occurrences`, append `message_ids`
6. For genuinely new tasks: insert at the correct priority rank
7. Re-score and re-rank the full merged set
8. Append gap messages to `output/messages.json` (sorted by datetime_iso)
9. Report: "[N] messages processed ([gap dates]). [Y] new tasks added, [Z] existing tasks updated."

---

## Full re-analysis (user picks option 2 or explicitly says 'redo')

Discard existing output files and run the complete pipeline from Stage 1 on the entire export. IDs restart from PREFIX-001.

---

## Show existing results only (user picks option 3)

Load `output/backlog.json` and `output/triage_board.html` and present them — no processing. Share the existing artifact URL if one exists.

---

## Pipeline overview (5 stages)

```
_chat.txt  (full export — skill filters to only the missing date range internally)
   |
   v  Stage 1: parse_chat.py  [filtered to gap dates]
messages_gap.json  (only the missing messages)
   |
   v  Stage 2: extract_refs.py + analyse.py
references_gap.json  +  analysis_delta.json
   |
   v  Stage 3: make_sessions.py
chunk_00.json ... chunk_09.json
   |
   v  Stage 4: 10 parallel subagents
_all_raw.json  (raw issues from gap messages only)
   |
   v  Stage 5: consolidate.py  [merge with existing output/backlog.json]
backlog.csv / .json  +  brief  +  board
```

---

## Stage 1 — Parse the chat

**Key considerations:**
- WhatsApp adds Unicode bidi marks (U+202F, U+200E/F, U+2068/9) inside sender names. Strip with a translation table, not a simple strip().
- Open with `newline=''` and normalise CRLF before parsing.
- Message regex (12-hour): `^\[(\d{1,2}/\d{1,2}/\d{2,4}),\s*(\d{1,2}:\d{2}:\d{2}\s*[AP]M)\]\s+([^:]+):\s*(?P<body>.*)?)`
- Continuation lines append to the previous message body.
- Classify types: `text`, `image`, `video`, `document`, `audio`, `sticker`, `deleted`, `system`.
- Detect `@mentions`.
- **Gap filtering**: keep only messages within the missing date range(s). Write to `messages_gap.json` — merge into main only after Stage 5 completes successfully.

**Output columns:** `id, date, time, datetime_iso, weekday, sender, type, message, attachment, mentions, chars`

---

## Stage 2 — Reference extraction + thematic analysis

**extract_refs.py**: Find contract/entity IDs matching `[A-Za-z]{2,6}\d{6,9}(CO|BO|PR|CR|WO|LN|CL)`. Merge new references into existing `references.json`.

**analyse.py**: Theme-classify gap messages. Theme groups: Rates & Pricing, Contracts, Billing & Invoicing, Documents & PDF, Integrations, Vehicles & Fleet, Users & Access, Performance, Security, Mobile App. Guard `statistics.median([])` with a length check. Exclude chatter from substantive counts.

---

## Stage 3 — Session segmentation

Group gap messages into sessions using a **90-minute gap threshold**. Filter to issue-bearing sessions using the PROB regex (`not working, error, broken, failed, issue, problem, bug, can't, unable, incorrect, missing, stuck, slow, crash, down, urgent`). Split into 10 chunk files.

---

## Stage 4 — Parallel issue extraction

10 simultaneous subagents, one per chunk. Required fields per issue:
```
title, module, type, severity, status, recurring, description,
suggested_fix, evidence_quote, reported_by, contract_refs,
message_ids, session_id, date
```
Prompt: "Read the sessions in [chunk file]. For each distinct problem or support request, output one JSON issue record. Return ONLY a valid JSON array."

Merge all outputs into `extract/_all_raw.json`.

---

## Stage 5 — Consolidate and deduplicate

Two-pass deduplication across combined existing backlog + new raw issues:
- Pass 1: normalized title similarity >= 0.86
- Pass 2: same module + type, similarity >= 0.72

For matches: push `first_raised` earlier if historical, increment `occurrences`, append `message_ids`.

**Priority scoring:**
```
severity_base:     critical=120, high=70, medium=35, low=12
category_bonus:    Security incident=+60, Outage/incident=+40, Bug=+10
occurrences_bonus: min(occurrences, 20) * 3
status_bonus:      unresolved=+35, recurring-resolved=+25, unclear=+12, resolved=0
span_bonus:        +15 if span > 60 days
recency_bonus:     +20 if last_raised within past 6 months
```

Sort descending by score. Re-rank the full merged set.

---

## Executive brief

1. Headline stat — total tasks, open %, full date range, message count
2. Security incidents — table, most urgent first
3. Largest structural module — most open tasks, slowest turnaround
4. Recurring manual chores — high-occurrence ops-request items
5. Performance/stability history
6. Prioritised fix list — top 10 with IDs and one-line rationale
7. Caveats — open items that may already be fixed

---

## HTML triage board

Self-contained HTML with: severity / status / category chips (multi-select), module dropdown, text search, expandable rows with evidence and contract refs, priority rank, light + dark themes. Update the existing Claude artifact URL if one exists.

---

## Configuration knobs

| Setting | Default | File |
|---|---|---|
| `SESSION_GAP_MINUTES` | 90 | make_sessions.py |
| `N_CHUNKS` | 10 | make_sessions.py |
| `DEDUP_THRESHOLD_1` | 0.86 | consolidate.py |
| `DEDUP_THRESHOLD_2` | 0.72 | consolidate.py |
| Module list | 14 modules | consolidate.py |
| Contract ID suffixes | CO/BO/PR/CR/WO/LN/CL | extract_refs.py |
| PROB regex | common failure words | make_sessions.py |
| `ID_PREFIX` | e.g. FRAC, PROJ | consolidate.py |

---

## Common pitfalls

- **Bidi marks**: strip U+202F, U+200E/F, U+2068/9 with a translation table.
- **CRLF**: open with `newline=''`, replace `\r\n` and `\r` before parsing.
- **Empty message bodies**: make body capture group optional.
- **StatisticsError**: guard `statistics.median([])` with a length check.
- **Security regex**: scope to attack vocabulary (ddos, brute-force, attacker, malicious), not generic terms like 'OTP'.
- **Subagent output**: strip markdown code fences before parsing JSON.
- **ID continuity**: load max existing IDs before assigning new ones.
- **Historical first_raised**: push earlier when a historical instance predates the known one.
- **Partial overlap**: only process the non-overlapping gap — never re-analyze already-processed messages unless user says redo.

---

## Output summary

| File | Contents |
|---|---|
| `output/messages.json` / `.csv` | All parsed messages, cumulative, sorted by datetime |
| `output/references.json` / `.csv` | Entity/contract references, cumulative |
| `output/themes_by_month.csv` | Theme volumes, full date range |
| `output/analysis.json` | Turnaround stats, load patterns |
| `output/backlog.csv` / `.json` | Prioritised deduplicated tasks, cumulative |
| `output/issues_raw.json` | Un-deduplicated raw issues, cumulative |
| `output/backlog_brief.md` | Executive brief, updated each run |
| `output/triage_board.html` | Standalone searchable board, updated each run |
