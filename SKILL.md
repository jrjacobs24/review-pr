---
name: review-pr
description: Deeply review a pull request by spawning multiple independent multi-model reviewers against a local working copy, then synthesizing their findings. For someone else's PR, the findings become constructive draft comments tracked on a canvas. For the user's own PR, a self-review produces a markdown report to read and hand to the implementing agent. Use when reviewing a PR, given a GitHub PR link to review, asked to self-review a PR, or asked for a thorough, grounded, or multi-model review of a pull request.
disable-model-invocation: true
mode: true
icon: git-pull-request
color: purple
---

# Review PR

Review a pull request authored by someone else. Thoroughness comes from spawning several reviewers across different models against a real local checkout; the delivered feedback is constructive, prioritized, and written for the PR author to read.

When the PR is the user's own, run a **self-review** instead. It's the same review with a different deliverable: a markdown report for the user to learn from and hand to the agent that implemented the PR. See Self-Review below.

## IMPORTANT

**Never post any comments or submit any reviews without explicit permission to do so.** All comments and feedback are run by the user for proofreading first. Drafting is always allowed; publishing never is until the user says so.

**Permission never persists.** It is granted for one publish action and specific content, and it expires the moment that action completes. A review session can run for hours across many turns, which makes this stricter, not looser:

- Approval given twenty turns ago is not standing approval now.
- Approval for one comment is not approval for the rest of the batch.
- Approval to post a summary comment is not approval to submit a review verdict.
- "COMPLETE CODE REVIEW" is a teardown signal and approves nothing.

When in doubt, ask again. Asking twice costs a turn. Posting something unapproved costs trust.

**A self-review never publishes anything**, with or without permission. If the user asks to post from a self-review, confirm that they want to switch to a normal review first.

## Self-Review

Run a self-review when the PR's author is the authenticated `gh` user (Step 1 checks this), or when the user asks for one. Say which mode you're in before spawning reviewers, so the user can correct it.

Steps 1 through 7 run unchanged. A self-review is not a lighter review. Steps 8 and 9 are replaced by `references/self-review.md`, which defines the report file, where it lives, and how later rounds update it. Step 10 still applies. There's no canvas, no draft comment, no summary note, and no verdict.

## The Three Texts

Every finding ends up written three times, for three audiences. Do not blend them.

| Text | Audience | Shape |
|------|----------|-------|
| Reviewer findings (Step 5) | Internal only, never seen by anyone else | Adversarial and as long as needed. Full evidence, traced paths, no praise. |
| Review context (Step 8) | The user, who may not know this area of the code | Explanatory. What the code does, how it's reached, why the concern matters. |
| Draft comment (Step 8) | The PR author, who knows this code well | Short. One to three sentences, collaborative, no orientation. |

The adversarial pass finds things. The context pass helps the user judge them. The draft comment communicates them to someone already fluent in the code.

The most common failure is letting these collapse into one: pasting reviewer evidence into a comment, or explaining the codebase to the person who wrote it. The author gets the short version. The user gets the explanation.

A self-review has no draft comment. Its report carries the review context for the user, plus a short evidence trace for the implementing agent, in separate labeled parts of each finding.

## Running as a Mode

This skill is built to run as a Custom Mode, pinned for a whole review session, because a real review is a conversation: present findings, the user edits drafts, they grant permission for some of them, the author pushes a fix, repeat. Activate it from the `/` menu with Option+Enter, or "Use as Mode" on the skill entry.

Two consequences follow from a session that outlives any single turn.

**Never trust memory for state.** Context gets compacted over hours. Anything Step 10 needs must be recoverable from a durable source:

| State | Recover from |
|---|---|
| Which PR | The canvas filename, or the review worktree's branch name |
| Working copy path | `git worktree list --porcelain` |
| Feedback items, drafts, statuses | The canvas `.canvas.data.json` sidecar |
| What's already posted | `gh pr view <pr> --comments` — authoritative, beats any local record |
| Commit the review covered | `reviewedSha` in the canvas state |
| Self-review findings, statuses, commit covered | The report file's Status lines and frontmatter |

Record the reviewed head SHA in canvas state as soon as the canvas exists. It's what makes incremental re-review possible.

**The checklist below is per review, not per turn.** Don't restart the pipeline on every message.

### Resuming

On any turn after the first, work out where you are before acting. If findings already exist, you are resuming, not starting.

When the author has pushed new commits, review only the delta:

```bash
gh pr view <pr> --json headRefOid
git -C <working-copy> fetch origin pull/<pr>/head
git -C <working-copy> diff <reviewedSha>..FETCH_HEAD
```

Spawn reviewers on that diff, with the original intent plus a note on what changed. Re-running all reviewers over the full PR wastes a fan-out and regenerates findings the user already triaged.

Merge the results into the existing tracker rather than rebuilding it:

- New findings get appended.
- Items the new commits fixed get marked resolved, not deleted. The user should see that a concern was raised and addressed.
- Items the user already dropped stay dropped. Don't resurrect them because a reviewer raised them again.
- Update `reviewedSha` to the new head.

For anything else the user asks mid-session — explain a finding, redraft a comment, reconsider a dismissal, check one file — just do that. A pinned mode doesn't mean every message restarts a ten-step process.

## Workflow

Copy this checklist and track progress:

```
Review Progress:
- [ ] Step 1: Intake the PR
- [ ] Step 2: Establish a local working copy (requires permission)
- [ ] Step 3: Gather context
- [ ] Step 4: State the intent
- [ ] Step 5: Spawn reviewers
- [ ] Step 6: Synthesize
- [ ] Step 7: Lead judgment
- [ ] Step 8: Deliver feedback (canvas if >3 items; self-review: write the report)
- [ ] Step 9: Publish (only on explicit permission; self-review: never)
- [ ] Step 10: Teardown on "COMPLETE CODE REVIEW"
```

---

## Step 1: Intake the PR

Expect a GitHub PR link (`https://github.com/<owner>/<repo>/pull/<n>`) or a `gh`-resolvable PR number.

**If the user didn't provide one, stop and ask.** Do not infer the target from the current branch, recent history, or a local `git diff`. Wait for a reply.

Resolve the PR's metadata in one call:

```bash
gh pr view <pr> --json number,title,body,url,author,headRefName,baseRefName,isCrossRepository,additions,deletions,changedFiles,statusCheckRollup,reviewDecision
```

Then check whether this is the user's own PR:

```bash
gh api user --jq .login
```

If it matches `author.login`, this is a self-review.

## Step 2: Establish a local working copy

Reviewers must read the real codebase, not just a diff. A diff alone hides callers, callees, types, and sibling modules — which is exactly where the substantive findings live.

First check whether the branch is already checked out:

```bash
git branch --show-current
git status --short
```

**If the PR's `headRefName` is the current branch**, confirm it's current with the remote (`git log --oneline -1` against `gh pr view <pr> --json headRefOid`) and use the existing working directory. Continue to Step 3.

In a self-review, the current branch is often the one the implementing agent is still working in. If that tree is dirty or ahead of the PR head, the reviewers would be reading unpushed work, and another chat may still be editing those files. Point this out and offer a worktree on the PR head instead.

**If it is not checked out, ask permission before touching the repository state.** Present both options without recommending one, and include the tradeoff that applies right now — specifically whether the working tree is dirty, since that determines how disruptive an in-place checkout would be:

- **Check out the branch in place** — switches the current working directory to the PR branch. Simple, reuses installed dependencies and build artifacts, but replaces whatever is currently checked out.
- **Create a git worktree** — leaves the current branch and any uncommitted work untouched. Costs disk space, and a fresh worktree has no `node_modules`, so lint/tests/typecheck won't run there without an install first.

Wait for the answer. Do not proceed on an assumption.

### In-place checkout

```bash
gh pr checkout <pr>
```

This handles fork PRs correctly.

### Worktree

```bash
git fetch origin pull/<pr>/head:pr-<pr>-review
git worktree add ../<repo>-pr-<pr> pr-<pr>-review
```

Fetching `refs/pull/<n>/head` works for fork PRs too.

**Use the `pr-<pr>-review` branch name exactly.** That convention is what lets teardown find this worktree later without relying on remembered state. State the path to the user so they can open it themselves.

If the review will need lint, typecheck, or tests, tell the user the worktree needs a dependency install first and ask whether to run it. Pure reading needs no install.

## Step 3: Gather context

Before reading code, understand the change:

1. **PR description and linked issue** — what problem is this solving? Note who filed the linked ticket. If it's the user, the summary note should thank the author for picking it up specifically. Check the issue author with `gh` for GitHub issues, or the reporter field for a Jira key in the description, but only when a linked ticket is actually referenced. Don't go hunting.
2. **Size** — over ~400 changed lines, note it, and flag the PR as a split candidate if the change set is genuinely separable.
3. **CI status** — from `statusCheckRollup`. Failing checks are the author's to fix; don't spend review budget re-deriving what CI already reported, but do read failures for signal about untested paths.
4. **Existing review activity** — `gh pr view <pr> --comments`. Don't repeat points already made by another reviewer; reinforce them by reference instead.
5. **Repo conventions** — read the repo's `CLAUDE.md`, `AGENTS.md`, or rules files if present. A pattern that looks wrong in isolation is often the house style, and flagging it wastes the author's time.

Collect the diff for the reviewer prompt:

```bash
gh pr diff <pr>
```

## Step 4: State the intent

Write one clear paragraph describing what this change is trying to accomplish, derived from the PR description, linked issue, commit messages, and the code itself.

This paragraph anchors every reviewer. If the PR description is thin or contradicts the code, say so — an unclear intent is itself a review finding, and it changes what the reviewers should scrutinize. If you genuinely cannot determine the intent, ask the user before spawning reviewers.

## Step 5: Spawn reviewers

Launch all reviewers in a single message using the Task tool, so they run in parallel.

**Model selection.** Use the `interrogate reviewers` line in `~/.cursor/rules/pstack-models.mdc`. Spawn one reviewer per entry, labeled Reviewer A, B, and so on to match the entry count. If the file or that line is missing, stop and ask the user to run `/setup-pstack`.

The adversarial signal comes from model diversity, not from assigned personas. Give every reviewer the same prompt and rubric. Keep every model on this panel. This is a cross-reference pass, so do not collapse it to the parent model or to the single work model.

For each reviewer:
- `subagent_type`: `generalPurpose`
- `model`: the configured entry
- `run_in_background`: `false`

If a configured value is `inherit-parent` or `auto`, omit `model` entirely. Those are aliases, not broken slugs.

If a model slug is rejected as unresolvable, read the valid slugs from the error, pick the closest equivalent from the same family at the highest reasoning tier available, and spawn with that. Tell the user which substitution you made. Never block the review on a slug issue.

Build each prompt from `references/reviewer-prompt.md`, filling in:
1. The intent paragraph from Step 4
2. The absolute path to the working copy from Step 2
3. The diff from Step 3
4. The rubric — inline the contents of `references/rubric.md`
5. The code-quality lens — inline the contents of `references/code-quality-review.md`

Reviewers explore the working copy read-only. The prompt template instructs them not to modify anything; keep that instruction intact.

## Step 6: Synthesize

Spawn one `generalPurpose` verifier and wait for it. Set `model` to the `bug-fix` line in `~/.cursor/rules/pstack-models.mdc`. If that line is missing, use `claude-opus-5-5-medium`. This is the work pass. It reads the working copy and the reviewer findings. It does not write the review the user sees, and it does not modify files.

The verifier returns:

1. **Consensus.** Findings raised independently by 2+ models.
2. **Lone-model findings.** Weighted lower, except security and correctness bugs, which stay at full scrutiny.
3. **Deduped items.** Same issue, different wording, with the models that raised each one.
4. **Disagreements.** One model flagging what another endorses.
5. **Traces.** For each item that could reach "Act on": the file, the call chain, and whether a caller, an upstream check, or the type system already prevents it.

## Step 7: Lead judgment

You are the lead reviewer — a pragmatic senior engineer, not a neutral aggregator. Read `references/lead-judgment.md` for the full filtering framework. Judge from the verifier's traces. Open a file yourself when a trace is missing, or when you are about to mark something blocking. The buckets, the verdict, and every sentence the user or the author reads are yours.

Sort every finding into one of four buckets:

- **Act on** — real issues in correctness, security, or maintainability that you want the author to address.
- **Consider** — legitimate, but you're unsure the fix is worth the cost right now. Worth the author's attention.
- **Noted** — technically valid, not actionable. Context-dependent or low-impact.
- **Dismissed** — wrong, nitpicky, or missing context. Include a brief reason.

Verify before you promote. Use the verifier's trace. A hypothetical that the type system or an upstream validation already prevents belongs in Dismissed. If the trace does not settle it, open the working copy before the item reaches "Act on."

Keep "Act on" short. More than five items usually means you haven't filtered hard enough.

**"Blocking" is a high bar.** Reserve it for a verifiable bug or severely degraded code, which in practice means a user-facing bug or a breaking change that you traced rather than suspected. Everything else is important-but-not-blocking, no matter how strongly the reviewers phrased it. Their severity labels don't transfer.

**Default the verdict to neither approve nor request changes.** Leaving comments and withholding approval is the normal outcome and carries plenty of signal. Reserve Request Changes for that same verifiable-bug bar; reserve Approve for changes that are genuinely fine, which nits don't disqualify. When you land in the middle, name what's holding you back in the relevant line comment and reference it in the summary note. `references/lead-judgment.md` has the full ladder.

## Step 8: Deliver feedback

**In a self-review, skip this step and Step 9.** Read `references/self-review.md` and write the report it describes. The `unslop` locator below still applies.

Read `references/feedback-style.md` and apply it to every finding you're keeping. It governs severity labels, length, collaborative phrasing, and the human-tone pass.

Three rules do the heavy lifting here:

**Keep draft comments short.** One to three sentences is the default and covers nearly everything. Roughly 80 words or a second paragraph is the ceiling. Don't narrate the code back to the author, don't report how you investigated, and never paste a reviewer's evidence block into a comment. Spend words only on a concrete trace or a code snippet when the finding genuinely needs one.

**Make them sound human.** Run every draft through the `unslop` skill, and the review prose you write for the user too. One deliberate exemption: the Communication artifacts rules (20 and 22) don't apply, because collaborative sign-offs and genuine praise are how real reviewers write.

`unslop` ships in the pstack plugin and won't appear in your skill list, because it sets `disable-model-invocation: true`. Read it directly. The install path includes a commit hash that changes on every pstack update, so resolve it with a glob rather than a fixed path:

```bash
ls -t ~/.cursor/plugins/cache/*/pstack/*/skills/unslop/SKILL.md | head -1
```

Read the file that command prints. If it prints nothing, pstack isn't installed: tell the user once, then apply the tells listed under "Sounding Human" in `references/feedback-style.md` as a fallback.

**Posted comments carry no severity label.** No emoji, no `[blocking]`, no `[important]`, and no invented substitute. The one exception is a lowercase `nit:` prefix for genuinely trivial items. Everything else conveys its stakes in the prose: state the concrete consequence for a blocker, say "not blocking" outright for an optional, ask a question when it's a question. Severity still drives the ordering in the review below, the canvas pills, and the verdict you recommend. It just never gets stamped onto the text the author reads.

**Give the user context the author doesn't need.** Assume the user hasn't read the diff and may not know this area of the codebase. Explain what the affected code does and how it's reached, so they can judge each finding rather than take it on faith. This context goes in the review you present and in the canvas. It never goes in the draft comment.

Translate each surviving finding into a comment the PR author would be glad to receive. An adversarial reviewer's "this is broken" becomes "what happens if `items` is empty here?" without losing the substance.

Present the review to the user in this structure:

```markdown
### Intent
[The paragraph from Step 4]

### Reviewers
- Reviewer [label]: [model], [N] findings

### Summary
[Two or three sentences: what this PR does, and your overall read on it.]

### Orientation
[For the user, not the author. What area of the codebase this touches, what the
affected code is responsible for, and how it gets reached at runtime. Name the
entry points and the key files. Assume the user hasn't read the diff and may
not know this area. Four to six sentences.]

### Strengths
[What was genuinely done well. Not filler — if there's nothing notable, omit the section.]

### Main Concerns
[The "Act on" bucket: what you want the author to address. Mark the rare true
blocker with 🔴 and say why it clears that bar; everything else here is
important but not blocking. For each, in this order:
  - **What this code does** — one or two sentences of context for the user.
  - **The concern** — what's wrong and what it causes.
  - **Draft comment** — the short author-facing text, quoted so it's obvious
    this is the part that would be posted. No severity label, `nit:` aside.
  - Which models raised it.]

### Suggestions
[🟡 important and 💡 suggestion items. Same shape, plus the tradeoff.]

### Questions
[❓ genuine clarifications where the code is ambiguous and the author knows something you don't.]

### Noted
[🟢 nits and low-priority observations. Brief list. These rarely need context.]

### Dismissed
[Findings you rejected, with one-line rationale each. This is a trust mechanism — it lets the user override your judgment.]

### Agreement Map
[Where models converged, where they diverged, and what that pattern says about the change.]

### Draft Summary Comment
[The actual note that would be posted as the review body, quoted. One to three
sentences per `references/feedback-style.md`: thanks, something genuinely
positive, where the review landed. Not a recap of the findings above.

Always show this, canvas or not. When there's no canvas this is the user's only
chance to see the body before it goes up.]

### Recommended Verdict
[💬 Comment / ✅ Approve / 🔄 Request Changes, with one sentence of justification.
Default to Comment. Reserve Request Changes for a verifiable user-facing bug or
breaking change, and say what's holding you back if you're recommending neither.]
```

### Canvas — automatic above three feedback items

**Count the feedback items surviving Step 7** — everything headed to the author across Main Concerns, Suggestions, Questions, and Noted. Dismissed findings don't count.

**More than three? Build a canvas. Don't ask.** Three or fewer fits in chat; beyond that, tracking each item's location, draft comment, and include/drop decision in linear markdown means the user is scrolling back and forth to proofread, and it's easy to lose an item.

The canvas holds **everything that would be posted, in editable form**: the summary comment and verdict at the top, then one tracker entry per item with its severity, location, concern, draft comment, and status. After that it explains the PR — change overview grouped core-logic-first, so the feedback has context.

Read `references/canvas-review.md` for the full structure, then `~/.cursor/skills-cursor/canvas/SKILL.md` before writing the file.

Three things to get right:

- **Every draft lives in an editable field backed by `useCanvasState`**, which persists to a `.canvas.data.json` sidecar. This makes the canvas the proofreading surface — the user edits in place instead of dictating changes through chat — and Step 9 reads those edits back.
- **The summary comment is drafted here, not at publish time.** It's the review body, the first thing the author reads, and the easiest artifact to publish unseen because it isn't anchored to a line. Draft it in the canvas with the verdict as a control beside it.
- **No canvas control may publish anything.** Include/drop marks and the verdict selection are intent, not authorization. The permission gate stays in chat.

When a canvas exists, keep the chat response short: the intent, the verdict, counts by severity, and a link to the canvas. Don't duplicate every finding in both places — the canvas holds the detail. Still surface blocking items in chat so they're visible without opening it.

If the PR is large or complex but produced three or fewer feedback items, a canvas is optional. Build one if the change is genuinely hard to comprehend — sprawling diff, many subsystems, core logic buried in boilerplate — and skip it otherwise.

## Step 9: Publish — only on explicit permission

**Stop here by default.** Present the review and wait.

The user must explicitly approve publishing. "Looks good" about the review's *content* is not permission to post it. If there's any ambiguity, ask.

**If a canvas was built, read its state sidecar before composing anything.** The user has likely edited drafts and toggled items there, and that state — not your original synthesis — is the source of truth:

```bash
ls ~/.cursor/projects/<workspace>/canvases/ | grep 'pr-<pr>-review'
```

Read the `.canvas.data.json` sidecar next to the canvas file. Post the draft text as the user edited it, include only items marked `include`, and skip anything marked `drop` or already `posted`. If the sidecar doesn't exist yet, the user never opened the canvas — confirm they've reviewed the drafts before posting.

**The summary comment comes from the canvas too.** Take the review body from `summary` and the verdict from `verdict`. Never compose a summary at publish time. The body is the first thing the author reads and it frames every line comment, so it's the last thing that should reach the PR unseen. If the user hasn't seen it, it isn't ready to post.

Before posting, confirm the summary still matches what's actually being published. If items were dropped or resolved after it was drafted, it may claim findings that are no longer in the review. Re-sync it and show the user, rather than publishing a body that contradicts its own comments.

**Approving the line comments is not approving the summary.** They're separate artifacts and they need separate confirmation, along with the verdict. Requesting changes on someone's PR is a consequential act, and it should never happen as a side effect of approving a batch of nits. State plainly what you're about to post: the body, the verdict, and how many line comments.

Once permission is explicit, `gh` handles it:

```bash
# Single summary comment
gh pr comment <pr> --body-file <draft.md>

# A review with a verdict
gh pr review <pr> --comment --body-file <draft.md>
gh pr review <pr> --request-changes --body-file <draft.md>
gh pr review <pr> --approve --body-file <draft.md>
```

For line-specific comments, use the GitHub MCP tools: `pull_request_review_write` with method `create` to open a pending review, `add_comment_to_pending_review` for each line comment, then `pull_request_review_write` with method `submit_pending`. The same permission gate applies to the submit step — a pending review is a draft, submitting it is publishing.

Post only what the user proofread. If they edit your draft, post their version.

Before posting in a resumed session, check what's already on the PR with `gh pr view <pr> --comments`. That's authoritative; a stale local status would otherwise duplicate a comment.

After posting, mark the published items `posted` and set `summaryPosted` in the canvas state so the tracker reflects what's already on the PR. A second round of review comments shouldn't re-post the first round, and a follow-up round needs its own body rather than a repeat of the original.

## Step 10: Teardown on "COMPLETE CODE REVIEW"

When the user sends **"COMPLETE CODE REVIEW"**, tear down the review environment.

This phrase is a teardown signal, not permission to publish. If drafted feedback is still unpublished — check the canvas state for items marked `include` that were never posted — ask what to do with it before removing anything. Once the worktree is gone, so is the context behind those comments.

**Find the worktree from git, not from memory.** By teardown time the session may have run for hours and the path may be long out of context:

```bash
git worktree list --porcelain
```

Look for the entry whose `branch` is `refs/heads/pr-<pr>-review`; the `worktree` line above it is the path. Then remove it:

```bash
git worktree remove <worktree-path>
git branch -D pr-<pr>-review
```

If no such worktree exists, the review either ran on an in-place checkout or was already cleaned up. Don't guess at a path, and don't remove a worktree you can't tie to this PR's branch — other worktrees in the list may be the user's own work.

If `git worktree remove` reports uncommitted changes, do not force it. Show the user what's there and ask. Those changes might be theirs.

If the branch was checked out in place instead, no cleanup is needed. Ask whether they want to return to their previous branch; don't switch on your own.

**Leave the canvas in place.** It lives outside the worktree, so removing the worktree doesn't touch it, and it's the durable record of what was raised and what was posted. Note that its jump-to-code controls stop working once a worktree is removed, and offer to delete it — don't delete it unprompted. The same goes for a self-review report. The implementing agent may still be working from it.

Confirm what was removed.

## Additional Resources

- `references/reviewer-prompt.md` — the template for each reviewer subagent
- `references/rubric.md` — the review lenses reviewers apply
- `references/code-quality-review.md` — the structural code-quality standard
- `references/lead-judgment.md` — how to filter reviewer findings
- `references/feedback-style.md` — author-facing tone, severity labels, and comment templates
- `references/canvas-review.md` — feedback-tracker canvas structure and state handling
- `references/self-review.md` — the self-review report, its location, and re-review rounds
