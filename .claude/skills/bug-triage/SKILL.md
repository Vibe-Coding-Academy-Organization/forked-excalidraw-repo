---
name: bug-triage
description: Triage one bug end to end by running the bug-reproducer subagent, then (only if it reproduced the bug) the bug-analyst subagent, and returning a fixed short answer with both reports. Use it when the user types /bug-triage, or asks to "triage", "check and assess" or "reproduce and analyze" a bug, given in plain English or as an issue URL. It is a manager: it never reads code, writes tests or fixes bugs itself, and it never asks questions, so it can run in a scheduled task with nobody watching.
---

# /bug-triage

Your single mission: take **one bug**, run the two expert subagents on it in order (first **bug-reproducer**, then **bug-analyst**), and answer back in a fixed form.

You are a **manager**. You do none of the experts' work yourself.

## Input

One bug: plain English as a user would write it, or the URL of an issue. The input may contain other information around it (for example a Slack channel and a thread timestamp). Ignore what you do not need. The bug is the bug description or issue URL, nothing more.

## Rules

- **Never ask the user anything.** Run from start to finish on your own: this skill must work in a scheduled task with nobody in front of the screen.
- **Never read the code, never write a test, never fix the bug.** The subagents do the work.
- **Never rewrite, shorten or summarize a report.** Not the bug-reproducer report before you give it to bug-analyst, and not the reports in your answer. Copy them character for character.
- **If a subagent fails** (error, no report, or a report that does not follow its template), answer in one line, `<subagent name> failed: <why>`, and stop.
- No Notion board is set up: file no ticket and do not use any Notion or other ticketing tool.

## Process (in this order)

### 0. Check the subagents exist

Check that both `.claude/agents/bug-reproducer.md` and `.claude/agents/bug-analyst.md` exist. If one is missing, answer in one line, `Missing subagent: <name> (.claude/agents/<name>.md not found)`, and stop.

### 1. Run bug-reproducer

Call the **bug-reproducer** subagent (Agent tool, `subagent_type: bug-reproducer`) with the bug **exactly as it was received**: the plain-English text or the issue URL, without rewording.

Wait for its report.

### 2. Read the verdict

Read the first line of its report: `**Reproduced: yes | no | stuck | needs info**`.

- `yes`: go to step 3.
- `no`, `stuck` or `needs info`: do **not** call bug-analyst. Go straight to the answer ("Not reproduced" form).

### 3. Run bug-analyst (only if the verdict is "yes")

Call the **bug-analyst** subagent (Agent tool, `subagent_type: bug-analyst`) with exactly two things:

```
Original bug:
<the bug exactly as received>

bug-reproducer report:
<the full bug-reproducer report, copied as is>
```

Wait for its report. Read the recommendation on its first line: `**Recommendation: GO | LATER | NO-GO**`.

### 4. Answer

Use one of the two forms below, and nothing else: no introduction, no comment, no conclusion.

## Output

### Reproduced form (verdict "yes")

```
Reproduced: yes
Recommendation: <GO | LATER | NO-GO, from bug-analyst's first line>

<the full bug-reproducer report, copied as is>

<the full bug-analyst report, copied as is>
```

(No "Ticket:" line, since no Notion board is set up.)

### Not reproduced form (verdict "no", "stuck" or "needs info")

```
Reproduced: <no | stuck | needs info, the bug-reproducer verdict unchanged>
Why: <copied from the bug-reproducer report>
```

What goes after `Why:`, copied as is:
- verdict `no`: the "What I tried" block;
- verdict `stuck`: the "Where it's stuck" block;
- verdict `needs info`: the list of what is missing.

### Filled-in examples

Reproduced:

```
Reproduced: yes
Recommendation: GO

**Reproduced: yes**

## 1. Steps to reproduce
...(full bug-reproducer report)...

**Recommendation: GO**

## 1. Analysis of the fix
...(full bug-analyst report)...
```

Not reproduced:

```
Reproduced: needs info
Why: Missing information: which tool was selected; what the user expected to see.
```
