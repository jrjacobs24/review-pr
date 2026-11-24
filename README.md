# review-pr

A Cursor Agent Skill for reviewing other people's pull requests.

Several reviewer subagents on different models review the PR against a real local checkout. A lead reviewer filters their findings, and the survivors become short, human-sounding draft comments. Nothing is posted until you approve it.

## What it does

- Checks out the PR branch, either in place or in a separate git worktree. It asks you which one first.
- Runs a multi-model reviewer panel. A verifier then traces each finding through the code before anything reaches you.
- Drafts comments of one to three sentences with no robotic severity labels, plus a short summary note and a recommended verdict. The verdict defaults to a plain comment, and Request Changes is reserved for verifiable user-facing bugs.
- Builds a Cursor Canvas when there are more than three feedback items. The canvas has editable drafts, the summary comment, the verdict, and a preview of exactly what would be posted.
- Never posts a comment or submits a review without your explicit permission. Permission covers one action and expires when that action completes.
- Cleans up its worktree when you send `COMPLETE CODE REVIEW`.

## Requirements

**[pstack](https://github.com/cursor/plugins/tree/main/pstack)** (required). This skill extends pstack and doesn't behave correctly without it:

- The reviewer prompt, rubric, code-quality lens, and lead-judgment framework are adapted from pstack's `interrogate` skill.
- Every draft comment goes through pstack's `unslop` skill so it reads like a person wrote it.
- The reviewer panel comes from the `interrogate reviewers` line in `~/.cursor/rules/pstack-models.mdc`, which `/setup-pstack` writes. The verifier uses its `bug-fix` line.

**[code-review-excellence](https://www.skills.sh/wshobson/agents/code-review-excellence)** (required). This skill's feedback approach and tone guidance build on it.

**[GitHub CLI](https://cli.github.com/)** (`gh`), authenticated against the repos you review.

**A GitHub MCP server** in Cursor, for posting line-specific comments. Summary comments and verdicts go through `gh`.

**Cursor**, with the Agents Window or the CLI if you want to run the skill as a Custom Mode (see Usage).

## Installation

1. Install pstack from Cursor's chat:

   ```
   /add-plugin pstack
   ```

2. Configure pstack's models. This writes `~/.cursor/rules/pstack-models.mdc`:

   ```
   /setup-pstack
   ```

3. Install code-review-excellence:

   ```bash
   npx skills add https://github.com/wshobson/agents --skill code-review-excellence
   ```

4. Clone this repo into your personal Cursor skills directory:

   ```bash
   git clone https://github.com/jrjacobs24/review-pr.git ~/.cursor/skills/review-pr
   ```

5. Check that `gh` is signed in:

   ```bash
   gh auth status
   ```

## Usage

Start a review with a PR link:

```
/review-pr https://github.com/<owner>/<repo>/pull/<number>
```

A review usually runs for several turns, so it works best as a Custom Mode that stays active the whole time. Pick `review-pr` from the `/` menu and press Option+Enter (Mac) or Alt+Enter (Windows) instead of Enter. Cursor offers Custom Modes in the Agents Window and the CLI.

When you're finished, send:

```
COMPLETE CODE REVIEW
```

This removes any review worktree the skill created. It doesn't publish anything, and it asks what to do with any drafts that haven't been posted.

## Credits

- **pstack** by Lauren Tan ([poteto](https://x.com/poteto)), MIT licensed. The reviewer prompt, rubric, code-quality lens, and lead-judgment framework in `references/` are adapted from its `interrogate` skill.
- **code-review-excellence** by Seth Hobson, from [wshobson/agents](https://github.com/wshobson/agents), MIT licensed. `references/feedback-style.md` builds on its feedback approach.

The full license notices are in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## License

MIT. See [LICENSE](LICENSE).
