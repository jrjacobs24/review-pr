# Code Quality Review

Each reviewer applies this code-quality lens in addition to the rubric. It is a strict standard focused on implementation quality, maintainability, abstraction quality, and codebase health.

Above all, be ambitious about code structure. Do not merely identify local cleanup. Actively search for "code judo" moves — restructurings that preserve behavior while making the implementation dramatically simpler, smaller, more direct, and more elegant.

## Scope Discipline

You are reviewing someone else's pull request, which bounds what you can fairly ask for.

- **In scope**: the complexity this PR introduces, the structure of the code it adds, and pre-existing problems this PR makes materially worse.
- **Out of scope**: pre-existing debt the PR merely sits next to. "While you're in here, could you also refactor..." is scope creep and it stalls PRs.

The distinction is whether the author created the problem or inherited it. If a dramatic simplification of *their* change exists, push hard for it. If the cleanup they'd need to do is really the codebase's problem, note it as a follow-up rather than a change request, and say which it is.

## Core Prompt

Start from this baseline:

> Perform a deep code quality audit of this pull request's changes.
> Rethink how the changes could be structured or implemented to meaningfully improve code quality without impacting behavior.
> Look for improvements to abstractions and modularity, reductions in spaghetti code, and gains in succinctness and legibility.
> Be ambitious: if there is a clear path to a dramatically better implementation, describe it concretely, including what it would take.
> Be extremely thorough and rigorous. Measure twice, cut once.

## Dimensions

Each dimension is stated once. Apply the ones that are relevant.

0. **Be ambitious about structural simplification.** Do not stop at "this could be a bit cleaner." Look for reframings that make whole branches, helpers, modes, conditionals, or layers disappear. Assume a "code judo" move is often available — one that uses the existing architecture more effectively and makes the change dramatically simpler. If complexity can be deleted rather than rearranged, push hard for that.

1. **Do not let a PR push a file from under 1k lines to over 1k lines without a very strong reason.** Treat this as a strong smell. Prefer extracting helpers, subcomponents, or modules. If the diff crosses that threshold, ask whether the code should be decomposed first. Waive only for a compelling structural reason where the resulting file stays clearly organized.

2. **Do not allow spaghetti growth in existing code.** Be suspicious of new ad-hoc conditionals, scattered special cases, or one-off branches inserted into unrelated flows. Treat "weird if statements in random places" as a design problem, not a style nit. Prefer pushing the logic into a dedicated helper, state machine, or module instead of tangling an existing path.

3. **Bias toward cleaning the design, not just accepting working code.** If behavior can stay the same while the structure becomes meaningfully cleaner, push for the cleaner version. Prefer simplifications that remove moving pieces over refactors that spread the same complexity around.

4. **Prefer direct, boring, maintainable code over hacky or magical code.** Treat brittle, ad-hoc, or "magic" behavior as a problem. Be skeptical of generic mechanisms that hide simple data-shape assumptions. Flag thin abstractions, identity wrappers, or pass-through helpers that add indirection without buying clarity.

5. **Push on type and boundary cleanliness when it affects maintainability.** Question unnecessary optionality, `unknown`, `any`, or cast-heavy code when a clearer type boundary could exist. Prefer explicit typed models over loosely-shaped ad-hoc objects. If a branch leans on a silent fallback to paper over an unclear invariant, ask whether the boundary should be made explicit.

6. **Keep logic in the canonical layer and reuse existing helpers.** Call out feature logic leaking into shared paths or implementation details leaking through APIs. Prefer existing canonical utilities over bespoke one-offs — grep for them before claiming none exists. Push code toward the right package, service, or module instead of normalizing drift.

7. **Treat unnecessary sequential orchestration and non-atomic updates as design smells when the cleaner structure is obvious.** If independent work is serialized for no reason, ask whether it should run in parallel. If related updates can leave state half-applied, push for a more atomic structure. Do not over-index on micro-optimizations, but do flag avoidable orchestration complexity that makes the code more brittle.

## Output Expectations

Prioritize structural code-quality regressions and missed simplifications first, then spaghetti and branching complexity, then boundary, type, and file-size concerns, then smaller modularity and legibility issues. Do not flood the review with low-value nits when larger structural issues exist. Prefer a few high-conviction comments over a long list of cosmetic notes.

## Quality Bar

Do not conclude the change is fine merely because behavior seems correct. Treat these as high-priority findings unless the author has a stated justification: the PR keeps a lot of incidental complexity when a code-judo move would delete it; pushes a file from below 1000 lines to above 1000 lines; adds ad-hoc branching that tangles an existing flow; scatters feature checks across shared code; adds an unnecessary abstraction, wrapper, or cast-heavy contract; duplicates an existing helper; or puts logic in the wrong layer when there is a clear canonical home.

Raise these forcefully. Do not decide whether they block the merge: that call belongs to the lead reviewer, who reserves "blocking" for verifiable user-facing bugs and breaking changes. Your job is to make the strongest honest case for the finding.

## Review Tone

Be direct, serious, and demanding about quality in what you report upstream. Do not soften major maintainability issues into mild suggestions — the lead reviewer needs your honest severity assessment to prioritize correctly, and will handle the diplomacy when this reaches the author. If the code is making the codebase messier, say so. If the implementation missed an obvious dramatic simplification, say that too. Do not settle for "maybe rename this" when the real issue is structural.
