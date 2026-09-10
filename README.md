# chatanalys-e

**by mouryanE**

Turns a WhatsApp group export into a prioritised issue backlog, executive brief, and searchable HTML triage board — in one go.

---

## What it does

Most support teams manage issues through WhatsApp groups — but nothing in that chat ever becomes a proper task list. Messages get buried, problems repeat, and there's no way to know what's actually open vs fixed.

**chatanalys-e** reads your WhatsApp export and turns the entire conversation history into a structured, prioritised backlog — the same thing a project manager would produce after reading 4,000 messages, done in minutes.

---

## Example use cases

### 1 — SaaS support team
A 6-person SaaS support team has been running their customer escalations through a WhatsApp group for 18 months. They export the chat and say:

> "analyze my WhatsApp support chat"

In minutes they get a ranked backlog of 240 tasks, grouped by module (Billing, Integrations, Mobile App). The triage board shows 12 critical issues still unresolved, 3 of them security-related. The executive brief highlights that the Billing module has the highest recurrence rate — the same payment failure has been reported 34 times.

### 2 — Fleet management ops
A fleet operations manager uses a WhatsApp group to coordinate between drivers, mechanics, and dispatchers. After exporting 3 months of chat:

> "extract issues from this WhatsApp export — build me a task list"

The skill identifies 87 distinct issues: vehicle breakdowns, document expiry warnings, route complaints. Contract IDs like `SABAA21009833CO` are automatically linked to every message that mentioned them, so the ops manager can see exactly which vehicle each fault belongs to.

### 3 — Incremental update (new history found)
A team member who joined the group 6 months ago discovers their admin exported the full group history going back 2 years. They already ran the analysis on their own 6-month window. They attach the full export and say:

> "extract issues from WhatsApp export"

The skill detects the gap and reports:
> "Already analyzed: 01 Mar 2026 → 10 Sep 2026. New export covers: 01 Jan 2024 → 10 Sep 2026. Missing: 01 Jan 2024 → 28 Feb 2026."

They choose **Process missing only**. The historical issues are merged into the existing backlog, with earlier `first_raised` dates pushing some recurring bugs up the priority ranking.

### 4 — Weekly re-run for new messages
A team runs the analysis every Friday. They export the chat and say:

> "analyze my WhatsApp support chat"

The skill finds 47 new messages since last Friday, processes only those, adds 3 new tasks, and updates occurrence counts on 6 existing ones. The triage board refreshes in place.

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

Nothing runs until you choose. Nothing gets overwritten unless you say so.

---

## How it works (pipeline)

### Stage 1 — Parse
Every message in the export is parsed into a structured dataset: sender, timestamp, message type, text, mentions, and any attachment references.

### Stage 2 — Classify & Reference
Messages are classified into themes (Billing, Contracts, Security, Performance, etc.). Contract and entity IDs are extracted with their full mention history.

### Stage 3 — Session segmentation
Messages are grouped into conversation sessions (90-minute gap = new session). Issue-bearing sessions are filtered and split into 10 parallel batches.

### Stage 4 — Issue extraction (parallel AI)
10 AI agents run simultaneously, one per batch, each extracting structured issue records with title, module, severity, status, description, suggested fix, and evidence quotes.

### Stage 5 — Deduplicate, score & output
All extracted issues are deduplicated and priority-scored. The result is a ranked backlog.

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

---

## Priority scoring

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

---

## Author

mouryanE
