# Self-Review

A self-review is the full review of a PR the user owns. Steps 1 through 7 run exactly as written: same working copy, reviewer panel, verifier, filtering, and blocking bar. What changes is the deliverable. Nothing is drafted for GitHub, and nothing is posted. The output is one markdown file with two readers:

1. **The user**, reading for their own learning and comprehension. They may not remember every detail of the change, and they want to understand each finding well enough to agree or disagree with it.
2. **The implementing agent**, in the separate chat where the PR was built. The user hands it the file, and it decides what to address.

Steps 8 and 9 don't apply. Follow this file instead. Step 10 still applies.

## How it differs

| Normal review | Self-review |
|---|---|
| Draft comments for another author | No comments. Findings written for the user and the implementing agent |
| Canvas above three items | Never a canvas. Always the markdown file |
| Summary note with thanks | No summary note |
| Verdict (comment, approve, request changes) | A readiness call instead. GitHub doesn't let you approve your own PR anyway |
| Severity never stamped on posted text | Plain-text severity on every finding. The agent needs it to prioritize |
| Evidence stays internal | A short evidence trace on every finding, so the agent can verify instead of trusting |
| Publishing gated on permission | Publishing never happens. Don't call `gh pr comment`, `gh pr review`, or any GitHub review tool |

The Communication-artifacts exemption from `feedback-style.md` doesn't carry over either. There's no social framing to preserve, so `unslop` applies in full.

## Where the file goes

```
~/.cursor/reviews/<repo>/pr-<number>-self-review.md
```

Keep it outside the repository. A file inside the working copy can end up committed into the very PR it reviews, and one inside a review worktree disappears at teardown. Create the directory if it doesn't exist. Tell the user the absolute path so they can open it or attach it in another chat.

One file per PR. Later rounds update it in place (see Re-review below).

## The file

Use this structure. Omit any section that would be empty.

````markdown
---
pr: <owner>/<repo>#<number>
url: <PR URL>
title: <PR title>
reviewed_sha: <head SHA the review covered>
reviewed_at: <YYYY-MM-DD>
round: 1
reviewers: <models on the panel>
---

# Self-review: <PR title> (#<number>)

## For the implementing agent

This is an independent review of this PR, run in a separate chat from the one
that implemented it. Treat each finding as a claim to check, not an instruction
to follow. Read the code it points to, then decide whether it's real and worth
addressing. You know things this review didn't, like what you already tried,
the constraints you hit, and what the user asked for. Use that.

Start with anything marked Blocking. Those are traced user-facing bugs or
breaking changes, the findings most likely to be real.

When you're done, update each finding's Status line in this file: `Addressed in
<commit>`, or `Declined: <one-sentence reason>`. Then tell the user what you
addressed and what you declined. Don't reopen anything under Dismissed, and
don't change code beyond what a finding calls for.

## Readiness

<Ready to merge | Ready after fixes | Needs another pass>. <One sentence on why.>

## What this PR does

<Four to six sentences for the user. Which area of the codebase this touches,
what that code is responsible for, how it's reached at runtime, and what the
change does to it. Name the entry points and key files.>

## Findings

### <ID>. <Short title>

IDs follow "Item IDs" in `SKILL.md`: `B` for Blocking, `C` for Should fix, `S`
for Consider. For example, `B1`, then `C1`, `C2`, then `S1`.

- Severity: <Blocking | Should fix | Consider>
- Location: `<path>:<line>`
- Raised by: <N of M models>
- Confidence: <High | Medium | Low>
- Status: Open

**What this code does.** <One or two sentences of context for the user.>

**The concern.** <What's wrong and what it causes. One to three sentences.>

**Evidence.** <The traced path that makes this real. A call chain, a minimal
input traced to the wrong output, or a short snippet.>

**Suggested direction.** <What would fix it, if there's a clear option. Leave
this out rather than guess.>

### <ID>. <Short title>

...

## Open questions

<Things the code alone couldn't settle, where the implementing agent likely
knows the answer. One line each, with IDs Q1, Q2.>

## Noted

<Nits and low-priority observations. One line each, with IDs N1, N2.>

## Dismissed

<Findings the panel raised that didn't hold up, one line each with the reason.
IDs D1, D2. The user can overturn any of them.>

## Takeaways

<One to three patterns worth remembering, each tied to a finding ID. Skip this
section if nothing generalizes.>
````

### Severity

Map the Step 7 buckets onto findings like this:

- **Blocking.** The same bar as a normal review: a verifiable user-facing bug or breaking change that you traced, not suspected. Keep it rare.
- **Should fix.** An "Act on" item that isn't blocking.
- **Consider.** The "Consider" bucket.

The "Noted" bucket goes in Noted, and "Dismissed" goes in Dismissed. Order Findings by severity, blocking first.

### Readiness

- **Ready to merge.** Nothing blocking, and the remaining findings are optional.
- **Ready after fixes.** Some Blocking or Should fix findings are worth addressing first, but the approach is sound.
- **Needs another pass.** The approach itself is in question, or the problems are structural enough that fixing them changes the PR substantially.

### Writing the findings

The two readers want different things from the same entry. The context lines are for the user; the evidence is for the agent. Keep them in their own labeled parts so neither reader has to dig.

- **IDs are stable.** The user and the agent will refer to findings as "C2" across chats, so the rules in "Item IDs" in `SKILL.md` matter most here. Never renumber, even when a later round resolves or adds findings.
- **Evidence is a trace, not a story.** "`useSubmit` calls `submitOrder` from the `onPress` at `checkout.tsx:88` with no in-flight guard, so a double tap sends two requests." Not a paragraph about how you found it.
- **Suggested direction is a direction.** The implementing agent decides how to fix it. Point at the fix; don't write the patch.
- **Takeaways are for learning, so make them specific.** "Debounce or guard any handler that fires a non-idempotent request (C1)" teaches something. "Think about edge cases" doesn't.
- Run all prose through `unslop`.

## Re-review

The user may run another round after the implementing agent has worked through the file. Read the existing file first. Its frontmatter and Status lines are the state; don't trust memory.

1. Review only the delta from `reviewed_sha` to the current head, using the same commands as Resuming in `SKILL.md`.
2. For each `Addressed` finding, check that the delta really fixes it. Mark it `Resolved in <commit>` if so. If the fix is incomplete, leave it open and add a sentence saying what's still missing.
3. Leave `Declined` findings alone. If the panel raises the same point again, don't reopen it. The user can reopen it themselves.
4. Append new findings, continuing each prefix's numbering from the highest ID used.
5. Update `reviewed_sha`, `reviewed_at`, `round`, and the Readiness call.

## In chat

Keep the chat response short. Give the readiness call, finding counts by severity, any Blocking findings in a sentence each, and the absolute path to the file. The file holds everything else.
