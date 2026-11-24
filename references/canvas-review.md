# Review Canvas

The canvas is the working surface for a review with more than three feedback items. Its job, in priority order:

1. **Show everything that would be posted, in editable form** — the summary comment, the verdict, and every line comment. Nothing reaches the PR that the user hasn't seen here first. This is the canvas's reason to exist.
2. **Track each feedback item** — what the concern is, where it lives, and whether it's still in the review.
3. **Explain the PR** — enough structure that the user can see what the change does and which parts matter, without reading the diff top to bottom.

Everything else is optional.

## Prerequisites

Read `~/.cursor/skills-cursor/canvas/SKILL.md` before writing any `.canvas.tsx` file. It owns the generation policy, design rules, the pre-delivery self-check, and — critically — the exact directory canvases must be written to. Discover exact exports and prop shapes from `~/.cursor/skills-cursor/canvas/sdk/index.d.ts` and its sibling `.d.ts` files rather than guessing.

Name the file for the PR so repeat reviews don't collide: `pr-<number>-review.canvas.tsx`.

## What Gets Posted

**Lead with this section.** Put it above the tracker, at the very top of the canvas.

A line comment is easy to preview because it's attached to a line. The summary comment is not, and that makes it the easiest thing to accidentally publish unseen. It's also the first thing the author reads and the frame for every other comment, so it's the last thing that should be a surprise. Treat it as a first-class artifact, drafted here and approved here.

### The summary comment

Render the review body in an editable `TextArea`, exactly like an item draft. Follow the summary format in `feedback-style.md`: a one-to-three sentence note, not a recap of the tracker below it.

Size the field for a short note. A tall textarea invites a wall of text and quietly undoes the format.

Never compose the summary fresh at publish time. If it isn't in the canvas, the user hasn't read it.

**Keep it in sync with the included items.** The summary references the findings, so dropped items make it wrong. When the user drops an item, marks one resolved, or edits a draft in a way that changes what's being asked for, re-derive the summary and say that you did. A body that claims four blocking issues when three are marked `include` is worse than no summary, and it's the kind of error that lands publicly.

If the user has hand-edited the summary, don't silently overwrite their wording. Point out what's now stale and offer to redraft.

### The verdict

The verdict posts with the body, so it belongs here too, as a control rather than a sentence you wrote. Comment, approve, or request changes.

Default to **comment**, meaning neither approve nor request changes. That's the normal outcome for a review with findings, per the ladder in `lead-judgment.md`. Request changes is reserved for a verifiable user-facing bug or breaking change.

Let the user override, and make the current selection obvious. This is their call, and requesting changes on someone's PR is the most consequential thing the canvas represents.

### Publish preview

Close the section with a plain manifest of what a publish would put on the PR, drawn from current state:

- The review body, with its verdict.
- Each line comment that's marked `include`, listed by `file:line`.
- Nothing else.

Counts help here: "1 review body, request changes, 5 line comments, 2 dropped." The point is that the user can see the complete set at a glance and catch anything unexpected before it's public rather than after.

## The Feedback Tracker

One entry per feedback item that survived lead judgment — the items headed to the author, not the dismissed ones.

Each entry carries:

| Part | Purpose |
|---|---|
| Severity | 🔴 blocking / 🟡 important / 🟢 nit / 💡 suggestion / ❓ question — as a `Pill`, tone-matched. Canvas only; never inside the draft comment text |
| Location | `file:line`, with a jump-to-code control (see below) |
| What this code does | One or two sentences of orientation **for the user**, not the author |
| Concern | One or two sentences: what's wrong and why it matters |
| Draft comment | The short author-facing text, **editable** (see below) |
| Consensus | Which models raised it — "3 of 3 models" is the strongest thing you can tell a reviewer |
| Status | Include / drop / posted — user-controlled |

### Separate the context from the draft

The orientation and the draft comment are for different readers, and the canvas has to make that obvious at a glance. The user must never have to guess which text would get posted.

Style them differently and label the draft explicitly. The draft comment sits in its editable field under a clear heading; the orientation reads as your commentary around it, in secondary text. If the two are visually interchangeable, the user will either edit the wrong one or assume their context notes are going to the author.

Orientation exists because the user may not know this area of the code. Say what the function or component is responsible for and how it's reached, then stop. Two sentences.

Order by severity, blocking first. Group by severity or by file, whichever makes the change easier to act on — a review concentrated in two files reads better grouped by file; one scattered across a dozen reads better by severity.

### Draft comments must be editable

Render each draft comment in a `TextArea` bound to persistent state, not as static `Text`. The user proofreads every comment before anything is posted, and editing in place beats dictating changes back through chat.

Hold all item state in a single `useCanvasState` entry keyed by finding id — one object, not a hook per item:

```tsx
type ItemStatus = "include" | "drop" | "posted" | "resolved";
type Verdict = "comment" | "approve" | "request-changes";

interface ItemState {
  draft: string;
  status: ItemStatus;
}

const [items, setItems] = useCanvasState<Record<string, ItemState>>("feedback", initialItems);
const [summary, setSummary] = useCanvasState("summary", initialSummary);
const [verdict, setVerdict] = useCanvasState<Verdict>("verdict", recommendedVerdict);
const [summaryPosted, setSummaryPosted] = useCanvasState("summaryPosted", false);
const [reviewedSha, setReviewedSha] = useCanvasState("reviewedSha", headSha);
```

`useCanvasState` persists to a `.canvas.data.json` sidecar beside the canvas source, so edits survive rebuilds, reloads, and IDE restarts. That's what makes the canvas durable across a multi-turn review — and it's how the user's edits get back to you at publish time.

Because this sidecar outlives the conversation's context, treat it as the review's source of truth rather than anything you remember:

- `summary` and `verdict` are what would be published as the review body. Read them from here at publish time rather than recomposing.
- `summaryPosted` prevents a second round from re-posting a body that's already on the PR.
- `reviewedSha` records the commit the review covered, so a later turn can diff only what the author pushed since.
- `resolved` marks an item a new commit fixed. Keep it visible with the draft intact instead of deleting it, so the user can see the concern was raised and addressed.
- `posted` and `drop` are decisions the user already made. A later round must not resurrect them because a reviewer raised the same point again.

### Jump to the code

Give each item a control that opens the file at the relevant line via the `openFile` canvas action:

```tsx
const dispatch = useCanvasAction();
dispatch({ type: "openFile", path: absolutePath, selection: { startLineNumber: line } });
```

**Use the absolute path into the review working copy**, not a workspace-relative one. If the review is running in a worktree, a relative path resolves against the user's main checkout and opens a version of the file that may not contain the change at all.

### Status is intent, not authorization

The include/drop/posted marks record what the user wants. They are not permission to publish. The canvas cannot post to GitHub — its only actions are `openFile`, `openAgent`, and `newComposerChat` — and that's the correct boundary. Publishing stays in Step 9, gated on explicit permission in chat. Never add a control that implies otherwise: no "Post" button, no "Submit review" button.

## The Change Overview

Below the tracker, give enough of the PR for the feedback to make sense.

Open with a short orientation for the user: what this change is for, which part of the codebase it touches, what that code is responsible for, and how it gets reached at runtime. Assume they haven't read the diff. A few sentences, or a small diagram if the flow is easier drawn than described.

Then the changes themselves, reorganized by reviewer value — never file-tree or alphabetical order:

1. **Core logic** — new behavior, algorithm changes, state transitions, API surface changes. Show real diffs with context via `DiffView`.
2. **Wiring & integration** — route registration, DI, config plumbing. Condensed; enough to confirm correctness.
3. **Boilerplate & mechanical** — import reordering, renames, generated code, formatting. A collapsed list of filenames and stats. No inline diffs.

Put the counts up top as `Stat`s — files changed, additions/deletions, items by severity — so the shape of the review is visible before any scrolling.

Anchor the two sections to each other: a file in the overview should show which feedback items land on it, and an item's location should be findable in the overview. Cross-referencing is the whole point of putting them on one surface.

### When the diff is hard to scan

For dense or intricate core logic — deeply nested conditions, state machines, retry/backoff flows, multi-step transformations — add a short pseudocode summary next to the diff, stripping syntax and error handling to expose the control flow. Lets the user confirm intent before reading the real code.

When a hunk changes behavior in a way that's hard to predict from reading it — reordered effects, new short-circuits, altered edge cases — trace one small, realistic input through the old and new paths side by side and mark where they diverge.

Use both sparingly. Straightforward changes don't need a pseudocode mirror, and overused callouts destroy their own signal.

## Dismissed Findings

Include them, collapsed, with the one-line rationale for each. Same trust mechanism as the markdown review: the user can see what you rejected and overturn it. Collapsed because it's reference material, not the main event.

Give each a control to promote it back into the tracker if the user disagrees — a dismissed finding the user wants is otherwise a full round-trip through chat.

## Tone

Two audiences share this canvas, so keep their text distinct:

- **Draft comments** are author-facing and get posted verbatim. Apply `feedback-style.md`: collaborative, specific, and short. One to three sentences, with no severity label in the text beyond a lowercase `nit:` prefix where it applies.
- **Your commentary** — orientation, section notes, callouts, the overview — is for the user. Explain what the code does, why something changed, and how files interact. Not what the diff already shows.

Run both through the `unslop` skill (Step 8 of `SKILL.md` shows how to find it), keeping the exemption `feedback-style.md` names for the Communication artifacts rules (20 and 22).

The severity pill lives in the tracker chrome, not in the draft field. That separation is the point: the user gets the at-a-glance triage marker, and the author gets prose that doesn't look generated.

Never let internal reviewer bluntness leak into a draft comment field. That text is one approval away from being posted under the user's name.

## Be Creative

The above is a floor, not a ceiling. The goal is the fastest path to understanding and acting on *this* review. A review of a refactor should not look like a review of a bug fix.

The SDK has charts, tables, diff views, DAG layout, collapsible sections, cards, stats, form controls, and interactive state. Reach for whatever serves the change: a severity distribution, a per-file confidence annotation, a before/after call graph, a state diagram, a table of input→output pairs. Let the canvas reflect the change in front of you.
