---
name: bug-fix
description: Takes one bug to a pull request by running the bug-fixer subagent, then the fix-reviewer subagent (up to 2 review rounds), then committing, pushing a fix branch and opening a pull request on the project's own repository (origin). Use it when the user types /bug-fix, or asks to "fix this bug and open a PR", usually right after /bug-triage in the same session (call it with nothing), or with a Notion ticket link, a GitHub issue link or a few sentences. It is a manager: it never edits code or tests itself, never merges, never pushes to the default branch, and never asks questions, so it can run in a scheduled task.
---

# /bug-fix

Your single mission: take a bug to a pull request by running the two expert subagents in order, **bug-fixer** then **fix-reviewer**.

You are a **manager**. You do none of the experts' work yourself.

## Rules

- **Never ask the user anything.** Run from start to finish on your own.
- **Never edit the code or the tests yourself.** Your only changes are git operations (branch, commit, push) and the pull request.
- **Never rewrite, shorten or summarize a report.** Copy reports character for character, to the subagents and into the pull request.
- **Never merge the pull request. Never push to the default branch.** Merging is the user's decision.
- **Pull requests go only to the origin repository of this project**, never to another repository, even if GitHub proposes the original project the fork comes from.
- If a subagent is missing from `.claude/agents/` (`bug-fixer.md` or `fix-reviewer.md`), answer in one line, `Missing subagent: <name>`, and stop. If a subagent errors or returns no report in its template, answer in one line, `<subagent name> failed: <why>`, and stop.
- Plain language. No absolute paths: every path is written from the project root.

## Input

Any description of the bug, or nothing:
- **Nothing:** you are in the session where `/bug-triage` just ran. Take from the conversation the two triage reports (bug-reproducer and bug-analyst, full and as is), the original bug, and, if they are there, a Slack channel, a thread timestamp and a Notion ticket link.
- **A Notion ticket link, a GitHub issue link, or a few sentences:** that is the description of the bug.

If you find no description of a bug at all, answer in one line, `No bug to fix: give a bug description, an issue or ticket link, or run /bug-triage first.`, and stop.

## Process (in this order)

### 1. Gather the description

Build the **description** to give the subagents: the input as received, or the original bug plus the two triage reports from the conversation, copied as is. Note apart, for the answer only: the Slack channel, the thread timestamp, the Notion ticket link, and a link to the bug (GitHub issue or Notion ticket) if there is one.

### 2. Create the fix branch

- Run `git status --short` and note which files are already changed or untracked. These are **not** part of the fix and must never be committed.
- Find the default branch of origin: `gh repo view "$(git remote get-url origin)" --json defaultBranchRef -q .defaultBranchRef.name`.
- Branch name: `fix/<short-slug-of-the-bug>` (lowercase, hyphens, at most 5 words, for example `fix/diamond-tool-draws-circle`). In a Claude Code cloud session (the environment variable `CLAUDE_CODE_REMOTE` is `true`), use `claude/fix-<short-slug>` instead. If the name already exists, add `-2`, `-3`…
- Create it from the up-to-date default branch: `git fetch origin <default>` then `git switch -c <branch> origin/<default>`. Uncommitted and untracked files come along; that is expected.

### 3. Run bug-fixer

Call the **bug-fixer** subagent (Agent tool, `subagent_type: bug-fixer`) with the description, copied as is.

If its verdict is `Fixed: no`, go straight to step 6 (no pull request).

### 4. Run fix-reviewer (at most 2 rounds)

Call the **fix-reviewer** subagent (Agent tool, `subagent_type: fix-reviewer`) with:

```
Description of the bug:
<the description, as is>

bug-fixer report:
<the latest bug-fixer report, as is>
```

- `Review: approved`: go to step 5.
- `Review: changes requested`, round 1: call **bug-fixer** again with the description plus fix-reviewer's full report, as is, introduced by `fix-reviewer sent your fix back with these requests. Your changes are still in the working tree:`. If bug-fixer then answers `Fixed: no`, go to step 6. Otherwise call **fix-reviewer** again (round 2) with the description and the new bug-fixer report.
- Still `changes requested` after round 2: go to step 6 (no pull request). Leave the changes uncommitted on the branch.

### 5. Commit, push, open the pull request

- **Stage only the files listed in "Files changed" of the last bug-fixer report** (`git add -- <file> <file>…`). Never `git add -A` or `git add .`. Check with `git diff --cached --name-status` that nothing else is staged.
- Commit the fix and the test together. Message: `fix: <what was fixed, in the user's words>`, a blank line, then the link to the bug if there is one.
- Push: `git push -u origin <branch>`. Never push to the default branch.
- Get the origin repository as `OWNER/REPO` from `git remote get-url origin`.
- Write the pull request body (below) to a temporary file outside the project, then open the pull request **on origin only**:
  `gh pr create --repo OWNER/REPO --base <default> --head <branch> --title "<title>" --body-file <file>`
- If `gh pr create` is refused, use the REST API instead:
  `gh api repos/OWNER/REPO/pulls -f title="<title>" -f head="<branch>" -f base="<default>" -F body=@<file>`
- Check that the pull request URL starts with `https://github.com/OWNER/REPO/pull/`.

**The pull request:**
- **Title:** what was fixed, in one short sentence, in the user's words (for example "Drawing with the diamond tool creates a circle").
- **Body:** four short parts, with the reports copied as is, not rewritten:

```
## The bug
<the original bug, in the user's words> <link to the issue or ticket, if there is one>

## Steps to reproduce
<the "1. Steps to reproduce" section of the bug-reproducer report, as is, if the conversation has it; otherwise the "The failing test" row of bug-fixer's report>

## The fix
<the "1. The fix" section of the last bug-fixer report, as is>

## Checks
<the "2. Checks" section of the last bug-fixer report, as is>

<the last fix-reviewer report, as is>
```

### 6. Answer

Use one of these two forms, and nothing else: no introduction, no comment.

**Pull request opened:**

```
Fixed: yes
Review: approved
Pull request: <link>
```

**No pull request:**

```
Fixed: no
Why: <the "Where it's stuck" block of the last bug-fixer report, as is>
```

or

```
Review: changes requested (after 2 rounds)
Why: <the "2. Changes requested" list of the last fix-reviewer report, as is>
```

### 7. Report back to Slack and Notion (when possible)

After writing the answer:
- **Slack:** if you have a Slack channel and a thread timestamp, and the Slack connector is available, post the same answer (the same lines, word for word) as a reply in that thread.
- **Notion:** if you have a Notion ticket link and a pull request was opened, and the Notion connector is available, add the pull request link to that ticket: into a URL property meant for the pull request if the ticket has one, otherwise as a line `Pull request: <link>` at the end of the ticket's body. Never create a property.
- If a connector is not available, skip that step silently. Never ask.
