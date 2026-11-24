# Feedback Style

How to write the surviving findings so the PR author is glad to receive them. Applies to Step 8 output and to anything that would ever be posted.

## The Mindset

Code review is knowledge sharing, not gatekeeping.

**Goals**: catch bugs and edge cases, ensure maintainability, share knowledge across the team, improve design, build team culture.

**Not goals**: showing off knowledge, nitpicking formatting (that's what linters are for), blocking progress unnecessarily, rewriting code to your preference.

## Translating Adversarial Findings

The reviewer subagents wrote for you. You write for the author. Every surviving finding needs this translation, and the substance must survive it intact — softening the tone does not mean softening the severity.

| Adversarial finding | Author-facing comment |
|---|---|
| "This is broken when the list is empty." | "What happens if `items` is empty here? Looks like `items[0]` on line 42 would throw." |
| "No error handling. Sloppy." | "How should this behave if the API call fails? Right now a rejection propagates to the caller unhandled." |
| "Inefficient. O(n²)." | "This loops through all users for each order. Fine at our current volume, but noticeable at 100k. Worth a lookup map?" |
| "Terrible variable name." | "nit: `userCount` instead of `uc`? Not blocking either way." |
| "Wrong pattern. Should use a service class." | "Would it make sense to extract this into a `PaymentService`? It'd make the tax logic unit-testable. Happy to pair on it if useful." |

## Length

**Default to one to three sentences.** That covers almost every comment. A nit is one sentence. Most blocking items are two or three: what's wrong, what it causes, what to do.

Hard ceiling: if a draft comment runs past roughly 80 words or a second paragraph, it's too long. Cut it or move the detail somewhere better.

The author knows this codebase, usually better than you do. They do not need the execution path narrated back to them. Name the line and the consequence, and stop.

### Cut these

- **Narrated code reading.** "When `fetchProfile` is called on line 82, it awaits `api.getProfile(userId)`, which returns a `Promise<ProfileResponse>`, and that response is then destructured on line 84..." The author wrote this. Skip to the problem.
- **Your investigation process.** "I traced the callers and checked the type definitions, and looking at `use-profile-screen.ts` I noticed..." Report the conclusion. How you found it is not interesting.
- **Restating the concern in different words.** Saying it once is enough. A second phrasing reads as padding.
- **Pre-emptive caveats.** "This may be intentional, and you may have considered this already, but if not, it's possible that..." One hedge, if any.
- **Textbook explanations.** Don't explain what a race condition is. Say where this one is.

### Compression example

Same finding, both accurate:

**Too long:**
> I want to walk through the flow here to explain my concern. When `fetchProfile` is called on line 82, it awaits `api.getProfile(userId)`, which returns a promise. That response is destructured on line 84 to pull out `bio` and `avatarUrl`. The issue is that `api.getProfile` can reject when the network is unavailable, and because there's no try/catch around this call, the rejection propagates to the caller. Looking at the callers, `useProfileScreen` calls `fetchProfile` inside a `useEffect` without a `.catch()`, which means the rejection becomes an unhandled promise rejection. In React Native that surfaces as a red box in development and may crash in release builds.

**Right:**
> `api.getProfile` can reject here, and the `useEffect` caller in `use-profile-screen.ts:40` has no `.catch()`, so a network failure becomes an unhandled rejection. Wrap it and surface an error state?

The short version loses nothing the author needs. It names the call, the unhandled path, and the fix.

### When detail earns its place

Some findings genuinely need more. Spend the words when:

- The bug needs a **concrete trace** to be believable. Give the smallest one that works: an input, two or three steps, the wrong output. A trace beats a paragraph describing a trace.
- A **code snippet** communicates the fix faster than prose. Show the snippet and cut the description of it.
- The edge case is **non-obvious** and the author will reasonably think you're wrong. Then earn it, briefly.

Even then, keep prose tight around the artifact. A trace plus two sentences beats three paragraphs.

### Reviewer evidence is not comment text

The reviewer subagents are asked to show their reasoning and cite what they read. That's raw material for your verification, not draft copy. Never paste a reviewer's Evidence block into a comment. Read it, confirm it, then write the short version yourself.

## Good Feedback Is

- **Specific and actionable** — points at a line, names the concrete consequence
- **Educational, not judgmental** — explains the reasoning so the author learns the principle
- **Focused on the code, not the person** — "this function" not "you always"
- **Balanced** — genuine praise where it's earned
- **Clear about stakes** — the wording tells the author whether this blocks merge or is take-it-or-leave-it

```markdown
❌ Bad: "This is wrong."
✅ Good: "This could cause a race condition when multiple users access
simultaneously. Consider using a mutex here."

❌ Bad: "Why didn't you use X pattern?"
✅ Good: "Have you considered the Repository pattern? It would make this
easier to test."

❌ Bad: "Rename this variable."
✅ Good: "nit: `userCount` instead of `uc`? Not blocking either way."
```

## Signaling Priority Without Labels

**Posted comments carry no severity label.** No emoji, no `[blocking]`, no `[important]`. Bracketed tags at the head of every comment read as machine-generated, and the author is going to read the comment either way. Let the wording carry the weight.

**`nit:` is the one exception.** Lowercase, at the very start, for anything genuinely trivial. It's the established convention and it does real work: it tells the author they can skip this without a second thought.

```markdown
nit: `userCount` instead of `uc`?
nit: this import is unused now.
```

### How to convey stakes in the prose

| Severity | How to say it |
|---|---|
| Blocking, meaning a real bug | Don't announce it. State the consequence concretely: "this throws when `items` is empty", "this'll double-charge on retry". A specific failure reads as serious on its own. |
| Holding up approval | Say it plainly: "this is the main thing holding me up", "I'd want this sorted before it goes in". |
| Should fix, open to pushback | "I'd lean toward fixing this, but let me know if you disagree." |
| Optional | Say so: "not blocking, but...", "totally optional", "take it or leave it", "no strong feelings here". |
| Suggestion | Frame it as an open question. "Worth pulling this into a helper?" already signals it isn't a demand. |
| Nit | `nit:` prefix. |
| Question | Just ask. The question mark is the signal. |
| Praise | Just say it. "Nice cleanup here." "Good catch on the null guard." |
| FYI, no action | "FYI" or state the context and don't ask for anything. |

**Don't invent replacement labels.** `[BLOCKING]`, `**Important:**`, `Priority: high`, or a severity word in bold at the start are the same robotic pattern wearing a different hat. The prose carries it.

Since the verdict is usually neutral (see `lead-judgment.md`), the comment prose carries nearly all of the signal. Don't rely on Request Changes to mark something as serious. Say it in the comment, and name it in the summary note if it's the thing holding up approval.

Severity still matters — it drives the ordering in the review you present to the user, the pills in the canvas tracker, and the verdict you recommend. It just doesn't get stamped onto the text the author reads. Keep classifying internally, and reserve the blocking bucket for things that genuinely must not merge. A review where everything reads urgent teaches the author to discount all of it.

## Ask Questions Instead of Stating Problems

Questions invite reasoning and leave room for the author to know something you don't. They're also more accurate — you're often working with less context than they are.

```markdown
❌ "This will fail if the list is empty."
✅ "What happens if `items` is an empty array?"

❌ "You need error handling here."
✅ "How should this behave if the API call fails?"

❌ "This is inefficient."
✅ "I see this loops through all users. Have we considered the
performance impact at 100k users?"
```

Use questions when you're inferring intent or might be missing context. Use direct statements when you've traced the code and are confident — a question about a bug you've verified reads as evasive rather than collaborative.

## Suggest, Don't Command

Collaborative language costs nothing and changes how the review lands.

```markdown
❌ "You must change this to use async/await"
✅ "Suggestion: async/await might read more cleanly here. What do you think?"

❌ "Extract this into a function"
✅ "This logic appears in 3 places now. Would it make sense to extract
a shared utility?"
```

Include a concrete code snippet when you have one. A suggestion the author can apply in one click is worth ten paragraphs of description.

## Sounding Human

Run every draft comment through the **`unslop`** skill before showing it. It's in the pstack plugin and won't show up in your skill list; Step 8 of `SKILL.md` has the command that finds it. Apply the same pass to the review prose you write for the user.

A comment that reads as machine-generated gets discounted on sight, no matter how good the finding is.

The tells that show up most in review comments:

- **Em dashes** (rule 13). Use a period or a comma.
- **AI vocabulary** (rule 7). "Crucial", "ensure", "robust", "leverage", "delve", "underscore". Say "make sure", "use", "solid".
- **"Not just X, but Y"** (rule 9). State the point.
- **Rule of three** (rule 10). Two concerns are two concerns. Don't invent a third for rhythm.
- **Inline-header lists** (rule 16). A bold label that restates the line reads as a template.
- **Over-compression** (rule 33). No arrow chains like `empty array → throws → red box`. Write the sentence.
- **Hedge stacking** (rule 24). "Could potentially possibly" is one "may".
- **Say what it does, not how it feels** (rule 27). "Makes the code cleaner" names a feeling. "Drops the second lookup" names a change.

### Two deliberate exemptions

**Communication artifacts (rules 20 and 22) do not apply here.** Those rules strip chatbot phrases and warm tone, which is right for most writing and wrong for code review. Human reviewers close with "let me know if you disagree", offer "happy to pair on this if useful", and open with "nice catch on the null guard". Stripping all of it produces comments that read as curt and machine-issued, which is the opposite of the goal. Keep them.

Keep them purposeful, though. "Happy to pair" on a one-line nit is noise; on a gnarly refactor it's a real offer. And praise still has to point at something specific, per the section above. Genuine warmth reads human. Reflexive warmth reads generated.

**Severity markers survive in the user-facing review only.** Rule 18 removes emojis from headings and bullets. The 🔴 🟡 🟢 markers used to sort the review you present to the user, and the pills in the canvas tracker, are functional wayfinding for triage, so keep them there.

They never appear in a posted comment. See the section above: author-facing text carries no label except `nit:`.

## Context Belongs to the User, Not the Author

The PR author works in this code. They don't need it explained to them, and explaining it reads as condescending.

The user reading your review may not know the area at all. They need the orientation: what this code does, how it's reached, why the concern matters.

These go in different places. Orientation goes in the review you present to the user and in the canvas. It stays out of the draft comment. If you find yourself writing "this component renders the recipe card, and it's reached from the search results screen when..." inside a draft comment, that belongs in the user-facing context field instead.

## Delivering Hard Feedback

Skip the praise sandwich. It reads as insincere. Name the issue, then offer a way forward:

```markdown
`calculateTotal()` mixes tax rules, discount logic, and database queries,
so the tax math can't be unit tested on its own. Worth pulling into a
`PaymentService`? Happy to pair on it if useful.
```

That's the whole comment. The issue, the consequence, a concrete direction, and an offer of help in three sentences. Structure doesn't require length, and a labeled three-part template is usually a sign the comment has outgrown what it needs to say.

The offer to pair matters more than the word count when the feedback is hard to hear. It signals you're not just assigning work.

## Praise Is Not Filler

Call out genuinely good work: a clever simplification, thorough edge-case tests, a well-named abstraction, deleted dead code. It's the cheapest way to make a review feel collaborative, and it signals what the team values.

But only when it's real. Manufactured praise is transparent and undermines the rest of the review.

The summary note always opens on a positive beat, so look for something true before settling. If there genuinely isn't one, don't invent it. Use the "I see what you were going for here" framing instead, which respects the effort without praising the result.

## Handling Disagreement

1. **Seek to understand** — "Help me understand your approach. What led you to this pattern?"
2. **Acknowledge valid points** — "That's a good point about X, I hadn't considered that."
3. **Provide data** — "I'm concerned about performance. Could we add a benchmark to check?"
4. **Escalate if needed** — "Let's get another set of eyes on this one."
5. **Know when to let go** — if it works and isn't critical, approve. Perfection is the enemy of progress.

## Don't Review Manually

Leave these to tooling, and don't spend the author's attention on them:

- Code formatting (Prettier, Black)
- Import organization
- Lint violations
- Simple typos in code

## Pitfalls

- **Perfectionism** — blocking a PR over style preferences
- **Scope creep** — "while you're at it, can you also..."
- **Inconsistency** — different standards for different authors
- **Bike shedding** — extended debate over trivia
- **Rubber stamping** — approving without actually reading
- **Ghosting** — requesting changes then disappearing

## The Summary Comment

The summary is a short note to a person, not a report. **One to three sentences.** The line comments carry the detail, and repeating it here wastes the author's time and makes the whole review feel generated.

No headings. No bulleted recap. No counts of findings.

### The three beats

1. **Thanks for the work.** If the PR picks up a ticket the user filed, say so. "Thanks for picking this up" lands differently than a generic opener.
2. **Something genuinely positive.** "Great work here." "I like what you did with the reducer." Name the specific thing when you can. If the PR has real problems, don't fake it: "I see what you were going for here, but there are a few things worth taking a look at" is honest and still respectful.
3. **Where the review landed.** "Nothing blocking, but I left a few comments I'd like to get your thoughts on." If something is holding up approval, name it here.

Not every note needs all three as separate sentences. They often combine.

### Examples

Clean PR with a couple of nits:

> Thanks for picking this up! Nice clean implementation, the hook split reads well. Nothing blocking, just a couple of small comments.

Solid work, some things worth discussing:

> Thanks for taking this on. I like what you did with the caching layer. Nothing blocking, but I left a few comments I'd like to get your thoughts on before this goes in.

Real problems, no user-facing bug:

> Thanks for tackling this one. I see what you were going for with the retry logic, but there are a few things worth taking a look at. The comment on `submitPayment` is the main thing holding me up.

A verifiable bug:

> Thanks for getting this out. I flagged one on `applyDiscount` that I think double-charges on retry, so I'll hold off on approving until we sort that out. Rest of it looks good.

### Concerns with no line to attach to

Some findings aren't anchored to a hunk: the PR is too large to review as one unit, the approach solves the problem at the wrong layer, a migration is missing. Those belong in the summary, since there's nowhere else to put them.

Keep it to a sentence and it still fits the format. Going past three sentences for this is a signal the concern deserves a conversation rather than a comment.

### What not to do

**Don't restate the findings.** If the author reads the same point twice, the summary failed.

**Don't pad the praise.** "Great work on this excellent and thorough PR" is filler. One specific true observation beats three generic ones. If you genuinely can't find something good, the "I see what you were going for" framing is the honest move.

**Don't thank and then undercut.** "Thanks for this! However, there are numerous issues..." reads worse than no thanks at all.
