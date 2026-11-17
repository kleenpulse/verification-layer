# verification-layer

Makes a coding agent prove its change works before it says "done", and builds the checks that make the next proof cheap.

## Install

```bash
npx skills add kleenpulse/verification-layer
```

Works with Claude Code, Codex, Cursor, Copilot, and anything else that reads `SKILL.md`.

## Why this one

"Run the tests" is not verification when the same agent wrote the code and the tests. This skill follows the practices in [How the Claude Code team thinks about verification](https://claude.dev/blog/how-the-claude-code-team-thinks-about-verification/): run the change in the real app, pick the test that is hard to fool, check only what the diff triggers, and keep the judges out of reach of the change they judge. It also covers the part most prompts skip, which is what to do when verification itself misses a bug.

The principles are from the article. The file formats in `references/templates.md` are this skill's own, because the article describes the practices without showing formats.

## What it does

- **Verifies a change:** states what "done" means from the user's side, runs the cheap checks, starts the app and exercises the change, then reports the evidence.
- **Picks the right test:** end-to-end for user flows, property tests for edge cases, differential tests for rewrites and migrations.
- **Runs a diff-triggered reliability checklist:** only the items whose trigger matches the diff, each one tied to a real incident.
- **Stops after three failed fix rounds** and hands over to a human with what it tried.
- **Stamps each commit** with how it was verified, including an honest `none`.
- **Sets up the layer in a repo:** a `verify` skill, a feature map, a checklist, a `REVIEW.md` and a commit gate with `off`, `observe` and `enforce` modes.
- **Turns a miss into a rule:** finds the pattern in at least two PRs, tries three times to disprove it, and asks the code owner to approve it. Then prunes rules that no longer earn their place.

## What it refuses to do

- Say "done" on unit tests alone when the app can be run
- Edit, skip or delete a test, lint rule or review rule in the same change it judges
- Skip a flaky test, add a sleep, or retry until green
- Start a fourth fix round on a check that keeps failing
- Write a stamp for a check it did not run
- Add hooks, gates or CI changes without asking
- Add a checklist item that has no incident or PR behind it

## Usage

- "Verify this change before you commit"
- "How do I know this actually works?"
- "Set up a verify skill and a reliability checklist for this repo"
- "We're rewriting the parser, which tests should we write?"
- "The review missed this bug. Turn it into a rule"
- "This test is flaky, what do we do?"

## License

MIT
