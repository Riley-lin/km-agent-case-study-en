# Safe Change Management and Scheduled Automation

Once knowledge documents are converted, they still need ongoing maintenance — and source content updates irregularly without any notification. This covers two problems: how to make edits safely, and how to know when something was changed.

## Built a local maintenance dashboard

Rather than rewriting a script or adjusting parameters every time maintenance is needed, I built a small local maintenance tool: the backend is a REST API written with Node's native `http` module (no additional web framework), and the frontend is a plain HTML/CSS/JS page, served locally at `localhost:3210`. The maintainer issues change commands, checks sync status, and triggers a refresh through this page, without touching the SharePoint admin backend directly or hand-writing one-off scripts each time.

## Command-based editing, not direct file edits

Knowledge document changes go through **command-based editing**: issue a command describing the change (action / scope / target), rather than opening a file and editing it directly.

- **A diff preview is shown first, and a save only happens after manual confirmation.**
- **An automatic backup is taken before saving**, preserving a history of changes.
- If a command executes but produces no actual change, the system **proactively raises an error**, rather than silently doing nothing.

That last rule matters more than it sounds: "no change occurred, but it was treated as a success" is a trap many automation tools overlook, and it's exactly the kind of silent failure that only surfaces once someone notices a wrong answer and traces it back. The whole approach follows the same principle as version control (the diff preview before a git commit): every change should be previewable, traceable, and reversible — never a one-way overwrite.

**The actual sync mechanism (internally called sync v2):** uses a file fingerprint to determine whether content has genuinely changed, rather than relying on timestamps alone — timestamps can falsely flag a file as "changed" just from being copied or moved, while fingerprint comparison looks at actual content. When both sides have modified the same file, it's flagged as a conflict rather than auto-resolved; any overwrite is always preceded by a backup of the previous version.

## A database gatekeeper: real-time monitoring and one-click rollback

Command-based editing protects "how to make my own changes safely." This protects something different: "how do I find out immediately when someone else accidentally touches something." If a knowledge file is mistakenly edited or deleted without anyone noticing, the agent starts giving wrong answers — but typically, no one catches it until someone notices something's off, and that gap is pure risk.

The solution: monitor knowledge document change events and send a **real-time notification** the moment something changes (who changed it, what file, when, with a direct link to review it), so the maintainer sees it before the problem spreads; if it turns out to be an accidental edit, it can be **rolled back to the previous version in one click**. This treats "preventing mistakes" and "how fast a mistake is caught and fixed" as two separate concerns — no process is careful enough to prevent one hundred percent of human error, so beyond error-proofing, monitoring plus fast rollback serves as the last line of defense.

## From "passive searching" to "active detection"

No one proactively notifies the maintainer when the public website or internal documents are updated, and waiting for someone to notice and manually search isn't sustainable. The solution is scheduled automation: using Windows Task Scheduler with a `.cmd` batch script, **running automatically once each morning and once each afternoon**, automatically checking for new content and processing it according to existing rules — no manual trigger needed. Running twice a day, rather than hourly, is a deliberate frequency trade-off — the public website doesn't update that frequently, and running too often just wastes resources and raises the risk of being flagged as abnormal traffic.

## File locking: preventing two runs from interfering with each other

The same data can be triggered for an update from multiple entry points at once (e.g. a manual trigger from the local maintenance tool plus the scheduled batch job running at the same time), and without coordination, both could write at the same time and collide. The solution is a shared lock file: while one run is in progress, the other can't start simultaneously. The lock itself needs a **timeout** — if a run is unexpectedly interrupted and fails to release the lock normally, the lock can't be allowed to stay stuck forever and block every subsequent scheduled run. This is basic deadlock-avoidance practice in concurrency control.

## Incremental mode vs. full mode

The normal run is **incremental mode**, comparing only what's changed since the last baseline — fast and resource-efficient. But incremental comparison inherently can't detect "the source deleted an entire entry" — if you're only looking for additions and updates, deletions are invisible. So a separate **full mode** is also run periodically (e.g. once a quarter) to do a complete rescan and catch deleted items; full mode is designed to be **read-only — it only checks, never downloads or modifies data** — to minimize execution risk.

## Sync never overwrites blindly; conflicts are left for a human to decide

When the automated sync encounters a situation where "what I'm about to update was just changed by someone else," it never unilaterally picks one side to overwrite — instead, it logs the conflict and produces a report for a human to decide which version to keep. The boundary of automation should stop exactly where certainty stops; anything uncertain is left for a human to decide.
