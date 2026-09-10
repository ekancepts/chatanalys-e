# chatanalys-e

**by mouryanE**

Turns a WhatsApp group export into a prioritised issue backlog, executive brief, and searchable HTML triage board — in one go.

---

## What it does

Most support teams manage issues through WhatsApp groups — but nothing in that chat ever becomes a proper task list. Messages get buried, problems repeat, and there's no way to know what's actually open vs fixed.

**chatanalys-e** reads your WhatsApp export and turns the entire conversation history into a structured, prioritised backlog — the same thing a project manager would produce after reading 4,000 messages, done in minutes.

---

## How to get your WhatsApp export

1. Open the WhatsApp group on your phone
2. Tap the group name → **More** (three dots) → **Export Chat**
3. Choose **Without Media** (media files are not needed)
4. Save or share the `.zip` file — it contains a `_chat.txt` inside

---

## How to use

Attach the `.zip` or `_chat.txt` to your Claude session and say any of:

- *"analyze my WhatsApp support chat"*
- *"extract issues from WhatsApp export"*
- *"turn this WhatsApp chat into a task list"*
- *"build a backlog from our WhatsApp group"*

Claude will take it from there.

---

## Smart incremental updates

If you've run this before, Claude checks what's already been analyzed and shows you exactly what's new:

> "Already analyzed: 15 Jun 2025 → 10 Sep 2026 (4,655 messages, 391 tasks)
> New export covers: 01 Jan 2021 → 10 Sep 2026
> Missing: 01 Jan 2021 → 14 Jun 2025 (pre-join history)"

It then asks how you want to proceed:

1. **Process missing only** — analyze the gap and merge into your existing backlog
2. **Full re-analysis** — reprocess everything from scratch
3. **Show existing results** — open the current board without processing anything

Nothing runs until you choose. Nothing gets overwritten unless you say so. Only if you explicitly say **"redo"** will the existing analysis be replaced.

---

## How it works (pipeline)

The skill runs a 5-stage pipeline automatically:

### Stage 1 — Parse
Every message in the export is parsed into a structured dataset: sender, timestamp, message type, text, mentions, and any attachment references. Unicode quirks and Windows line endings in WhatsApp exports are handled automatically.

### Stage 2 — Classify & Reference
Messages are classified into themes (Billing, Contracts, Security, Performance, etc.). Contract and entity IDs (like `SABAA21009833CO`) are extracted with their full mention history — who raised them, who handled them, and when.

### Stage 3 — Session segmentation
Messages are grouped into conversation sessions (90-minute gap = new session). Sessions that contain a problem signal (*"not working"*, *"error"*, *"stuck"*, *"urgent"*, etc.) are filtered out as issue-bearing sessions and split into 10 parallel batches.

### Stage 4 — Issue extraction (parallel AI)
10 AI agents run simultaneously, one per batch. Each reads its sessions and extracts structured issue records: title, module, severity, status, description, suggested fix, evidence quote, and message IDs linking back to the original conversation.

### Stage 5 — Deduplicate, score & output
All extracted issues are deduplicated (the same bug reported 12 times becomes one task with `occurrences: 12`). Each task is scored by severity, how often it recurred, how long it's been open, and whether it's still active. The result is a ranked backlog.

---

## What you get

| File | Contents |
|---|---|
| `output/messages.csv` / `.json` | Every parsed message with theme classification |
| `output/references.csv` / `.json` | All contract/entity IDs and their mention history |
| `output/themes_by_month.csv` | Theme volumes month by month |
| `output/analysis.json` | Turnaround times, load by hour/day, top senders |
| `output/backlog.csv` / `.json` | Prioritised deduplicated task list (importable to Jira/Linear) |
| `output/issues_raw.json` | All raw extracted issues before deduplication |
| `output/backlog_brief.md` | Executive brief — headline findings, security incidents, fix priority list |
| `output/triage_board.html` | Standalone searchable board (works offline, no server needed) |

### The triage board
A self-contained HTML file you can open in any browser or share with your team. Filter by:
- **Severity** — critical / high / medium / low
- **Status** — unresolved / unclear / resolved / recurring-resolved
- **Category** — Security incident / Outage / Bug / Enhancement / Recurring ops chore
- **Module** — Billing, Contracts, Performance, etc.
- **Text search** — across title, description, and task ID

Each row expands to show the original evidence quote and any contract references, with a direct link back to the message ID in the raw dataset.

### The executive brief
A markdown document structured for a technical lead or project manager:
1. Headline numbers (total tasks, open %, date range)
2. Security incidents — flagged first, always
3. Largest problem module
4. Recurring manual chores that could be automated
5. Top 10 priority fixes with one-line rationale each
6. Caveats (items that may already be fixed but were never confirmed in chat)

---

## Priority scoring

Each task gets a numeric priority score:

| Factor | Points |
|---|---|
| Critical severity | 120 |
| High severity | 70 |
| Security incident | +60 bonus |
| Outage/incident | +40 bonus |
| Unresolved status | +35 |
| Each recurrence (up to 20) | +3 each |
| Open longer than 60 days | +15 |
| Still active (raised in last 6 months) | +20 |

Tasks are ranked by this score — the most dangerous, most frequent, longest-running issues rise to the top.

---

## Configuration

These defaults work for most support groups but can be adjusted per project:

| Setting | Default | What it controls |
|---|---|---|
| Session gap | 90 minutes | How long a silence before a new conversation starts |
| Parallel chunks | 10 | Number of AI agents running simultaneously |
| Dedup threshold 1 | 0.86 | How similar two issue titles must be to merge |
| Dedup threshold 2 | 0.72 | Secondary merge pass (same module + type) |
| ID prefix | e.g. FRAC, PROJ | Prefix for task IDs in the backlog |
| Contract ID pattern | CO/BO/PR/CR/WO/LN/CL | Document type suffixes to recognise as entity references |

---

## Author

mouryanE
