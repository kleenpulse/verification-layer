---
name: verification-layer
description: Prove an agent-written change works before it is committed, and build the checks that make that proof cheap. Runs the change in the real app, picks the right test (end-to-end, property or differential), runs only the reliability checks the diff triggers, stops after three failed fix rounds and hands over to a human, stamps each commit with how it was verified, and turns every miss into a rule. Also sets up the layer itself in a repo - a `verify` skill, a feature map, a diff-triggered reliability checklist, a REVIEW.md, a commit gate - and keeps the change from editing its own judges. Use when the user says "verify this change", "prove it works", "how do I know this works", "is this actually done", "set up verification", "add a verify skill", "write a REVIEW.md", "build a reliability checklist", "add a commit gate", "which tests should I write", "this test is flaky", "the review missed this bug", "turn this miss into a rule", or "I want to run more agents without reading every diff".
license: MIT
---

# Verification Layer: Done Means There Is Evidence

You prove that a change works before it is committed, and you build the checks that make the next proof cheaper. The human's job moves from reading every diff to defining "done" and reading the evidence. Your job is to produce that evidence honestly.

The principles come from [How the Claude Code team thinks about verification](https://claude.dev/blog/how-the-claude-code-team-thinks-about-verification/) (Addy Osmani, Nov 2026). The file formats in `references/templates.md` are this skill's own; the article describes the practices but shows no formats.

## When to Use

- You finished a change and are about to say "done" or commit
- The user asks "does this work?", "prove it", or "verify this"
- A repo has no repeatable way to start and smoke-test the app
- The user asks which tests are worth writing for a feature, a rewrite or a migration
- A review or a test suite missed a bug that reached users, and the user wants it caught next time
- A test is flaky and someone wants to retry or skip it
- The user wants to run several agents at once and can't read every diff

**Do not use for:** reviewing someone else's diff line by line (that is a review, not verification), a one-line doc or comment change where nothing can run, or adding tests to raise a coverage number.

## Core Principle

**A change is done when evidence says so, and a change may never edit its own judge.** "The tests pass" is weak evidence when the same change wrote the tests. Strong evidence is the running app doing what the user asked, checked by something the change could not touch.

## Rules

1. **Run the change.** Start the real app and exercise the changed behaviour. This matters more than unit tests. A passing unit suite with an app that won't start is a failed verification.
2. **Three rounds, then a human.** Run the checks, fix what fails, run again. If a check still fails after three rounds, stop and hand over with the evidence. Don't start a fourth.
3. **Never weaken a judge to pass it.** No editing, skipping or deleting a test, a lint rule, a checklist item or a REVIEW.md rule in the same change it would judge. If a judge is wrong, say so and change it in a separate commit the user approves.
4. **Read judges from the base branch.** Review rules, checklists and gate config come from the default branch, never from the branch under test.
5. **Evidence belongs to one commit.** A verification of an older commit says nothing about a newer one. After any new edit, verify again or report that the evidence is stale.
6. **Cheap checks first, and locally.** Tests, build and lint run before a PR opens, not in a reviewer's time.
7. **Run only the checklist items the diff triggers.** Usually 3 to 10 items match. Running all of them every time teaches everyone to ignore the list.
8. **Stamp every commit with how it was verified, including `none`.** An honest `none` can be searched for later. A missing stamp can't.
9. **A flaky test is a bug.** Don't skip it, add a sleep, or retry until green. Find the root cause. If you can't, delete the test and write down why.
10. **A check that calls for a human must say why.** "Needs review" is not a reason. "Changes the session file format; old transcripts may not load" is.
11. **Every miss becomes a rule, and rules get pruned.** A bug that slipped through is a gap in the layer. Close it with a check, and remove checks that no longer earn their place.

### Which test to write

| Situation | Test | What it proves |
| --- | --- | --- |
| A user-facing flow (if you write only one test, write this) | **End-to-end** | A real user path works from start to finish. A failure shows the step, a trace and a screenshot, not just "timeout". |
| Parsers, encoders, money, state machines, anything with edge cases | **Property** (fast-check, Hypothesis, proptest) | A statement holds for thousands of generated inputs you didn't think of. |
| A rewrite, migration or optimisation of something that already works | **Differential** | Old and new give the same output on the same inputs. |

Properties worth checking:

| Property | Statement |
| --- | --- |
| Round trip | Encode, then decode, returns the original |
| Invariant | Something always holds (a balance is never negative) |
| Idempotency | Doing it twice equals doing it once |
| Equivalence | The fast version matches a slow, obviously correct version |

For differential tests: use random or sampled real inputs, normalise differences that don't matter (whitespace, key order, timestamps), and keep a written list of the differences you approved.

A slow suite gets run less, so coverage drops. Keep the per-edit loop fast and move heavy tests to just before the commit.

## What to Keep (Untouchables)

- Existing tests, lint rules, checklist items, REVIEW.md and gate config: never changed in the commit they judge.
- Command output, exit codes, failing step names and counts: reported exactly as printed.
- The stamp: it says what you really ran. Never write `e2e` for a run you didn't do.
- Screenshots and traces: attached or linked as they are, not described from memory.

## Method

### A. Verify a change (every time)

1. **Say what "done" means.** Write one or two sentences from the user's side: what they do, what they see. If the repo has a feature map, find the entry and use its expected result as the rubric. If you can't state "done", ask before verifying.
2. **Run the cheap checks.** Build, lint, type check, the fast tests. Fix failures. This is round one.
3. **Run the change.** Use the repo's `verify` skill if there is one (look in `.claude/skills/` near the code you changed). If there is none, start the app yourself:
   - Web: drive a browser (Chrome extension, DevTools MCP, Playwright).
   - Mobile: a simulator through a mobile MCP.
   - Desktop: computer use.
   - Backend or CLI: run the service or command locally and call it.

   Exercise the changed path and one path next to it that should not have changed. Capture a screenshot, a trace or the output.
4. **Run the triggered checklist items.** Read the reliability checklist from the base branch. For each item, test its trigger against your diff. Run the check for every match and record the result. List the matched items by name in your report.
5. **Add the missing test.** Use the table above. Confirm the new test fails without your change and passes with it.
6. **Count the rounds.** Each "run checks, fix, run again" is one round. After the third round with a check still failing, stop. Report what fails, what you tried in each round, and your best guess at the cause.
7. **Stamp and report.** Add the stamp to the commit message (format in `references/templates.md`). In your report give: the definition of done, what you ran, the evidence, the checklist items matched, and anything not verified with the reason.

### B. Build the layer (once per repo, then tune)

Do these in order. Each one is useful on its own, so stop wherever the user wants.

1. **A `verify` skill.** Put it in `.claude/skills/verify/`, as close to the code it covers as possible (a monorepo can have one per app). It records how to start the app and smoke-test it. Run it several times in a row before you trust it: it must be fast and give the same result every time.
2. **A feature map.** List features from the user's point of view. For each: the path, keyboard shortcuts, selectors and the expected result. Agents use it as a guide and as a grading rubric.
3. **One end-to-end test for the most important flow.** Then property and differential tests where the table says they fit.
4. **A reliability checklist.** Each item has three parts: a trigger you can read from a diff, a check, and the incident or PR that caused it. No incident, no item. Example: trigger "session storage or message types change", check "does a new field load safely from an old transcript that doesn't have it?".
5. **A REVIEW.md.** It tells the reviewer what to pay attention to in this codebase. Tune this one reviewer over several rounds, where each round builds on what the last one found. Don't stack more reviewers on top. The code owner refines it until it flags what they would flag.
6. **Local enforcement.** A hook that runs the tests when the agent finishes, or a goal that requires a passing suite.
7. **A commit gate.** It blocks a commit when the session ran no review and no verify step. Roll it out in three modes:

   | Mode | Behaviour |
   | --- | --- |
   | `off` | Nothing |
   | `observe` | Logs what it would have blocked. Run this first to find false positives. |
   | `enforce` | Blocks the commit. A commit may carry a written reason for skipping. After two blocks in one session, the next commit goes through so the agent can't get stuck. |

Ask the user before adding hooks, gates or CI changes. They change how everyone commits.

### C. Turn a miss into a rule

Use this when a bug got past the tests or the review.

1. **Find the pattern, not the bug.** Look at human review comments and reverts. A candidate rule must show up in at least two PRs and be visible from a diff alone.
2. **Try to disprove it.** Run three independent attempts to show the rule is wrong, too broad or already covered. If subagents are available, use three. Keep the rule only if it survives.
3. **Check it was catchable.** Replay the PR that caused the incident through three independent bug finders. If two or more find the bug, it was detectable, and a miss on a small diff should become a REVIEW.md rule or a checklist item.
4. **Get a human to approve it.** The code owner decides whether the rule goes in. You propose; they approve.
5. **Grade the reviewer.** Keep a small set of incident cases, each with a one-sentence answer key, plus merged clean changes of similar size as controls against over-flagging. Keep a tuning set and a separate test set. Tag findings `Important`, `Nit` or `Pre-existing`. Stop tuning when more recall would only fit the eval.
6. **Prune.** Test suites, checklists and `.md` rule files only grow unless someone cuts them. On a routine, remove rules that never fire, duplicate another rule, or guard code that no longer exists.

## Worked Example

**Request:** "Add a `pinned` flag to chat sessions so pinned ones sort first."

**Without the skill:**

> Added `pinned` to the session type and updated the sort. Unit tests pass. Done.

**With the skill:**

> **Done means:** a user pins a session in the sidebar and it moves to the top; after a restart it is still on top.
>
> **Ran:** build, lint, 148 unit tests (round 1: 2 failed on the sort comparator, fixed; round 2: all pass). Started the app with the repo's `verify` skill, pinned a session, restarted, confirmed the order. Screenshot attached.
>
> **Checklist items matched (2 of 31):**
> - *Session format changes load old data*: opened a session file saved before this change. It crashed: `pinned` was `undefined` and the comparator threw. Fixed by defaulting to `false` on load. Round 3: passes.
> - *Sort changes keep a stable order*: unpinned sessions keep their previous order. Passes.
>
> **Added:** a property test, "sorting twice equals sorting once", and an end-to-end step for pin, restart, check order. Both fail without the fix.
>
> **Not verified:** the mobile layout. No simulator on this machine.
>
> Stamp: `Verified-By: run, e2e, property, checklist(2)`

The unit tests passed while the app crashed on every existing user's data. Running the change against an old file is what found it, and the checklist item is what said to try.

## Self-Check Before You Deliver

1. Your report has a "done means" line written from the user's side.
2. The app was started and the changed path exercised, or the report says "not run" with the reason.
3. `git diff <base>` shows no edit to an existing test, lint config, checklist, REVIEW.md or gate config. If it does, that edit is called out and in its own commit.
4. No `.skip`, `.only`, `xit`, `sleep`, `retry` or raised timeout was added to make a test pass (search the diff for each).
5. The checklist items you matched are listed by name, with a result for each.
6. Every new test was seen to fail without the change.
7. The round count is three or fewer. If the third round still failed, the report hands over to a human and you did not commit.
8. The commit message has a `Verified-By:` stamp, and each word in it matches something you actually ran.
9. The evidence was gathered on the current `HEAD`, not on an earlier commit.
10. Every new checklist item or rule names the incident or PR behind it.
