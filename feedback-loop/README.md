# Dual-Track Feedback Growth Loop

"Couldn't answer" and "answered wrong" are two fundamentally different kinds of failure, and worth tracking separately:

- **Knowledge gap** — the agent honestly says "no data found," meaning the knowledge base has a gap that needs filling.
- **Downvote feedback** — the agent gave an answer, but the user flagged it as wrong, meaning the existing knowledge or logic needs correcting.

Mixing the two together makes the resulting data hard to act on — a spike in knowledge gaps points to insufficient content coverage, while a spike in wrong-answer reports points to a problem with existing content or retrieval logic, and the two point to completely different fixes. Each track maintains its own independent list, and Power Automate sends a scheduled summary report to the maintainer on a fixed cycle (e.g. every Monday morning), so nothing needs to be manually watched day to day.

## A reporting UX detail: no need to retype the question

Every answer includes a fixed "Is this answer wrong?" quick-report entry. When a report is triggered, the system reuses context that's already present in the conversation — what the user just asked, what the agent just answered — which are already stored in shared variables, and carries them straight into the report form. The user only needs to fill in two things: what was wrong, and what the expected answer should have been — no need to retype the original question.

This detail matters more than it sounds: every extra bit of typing required is one more chance for the user to just give up. Keeping the reporting threshold as low as possible is what makes people actually willing to click that button — and it's exactly why the volume of knowledge-gap and downvote data is enough to support ongoing accuracy tracking after launch (see the Methodology and Evaluation section in the [root README](../README.md)).

## Why this loop works

Neither track is enough on its own: a pile of "no data found" records with no priority ranking is just noise; a single "wrong answer" report with no ongoing tracking just means the same issue resurfaces the next time someone phrases the question differently. Tracking both on a fixed cycle is what turns "the agent's quality on launch day" into "the agent keeps getting better over time" — instead of the quality at launch being the quality it's stuck with forever.
