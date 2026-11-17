# Templates

Starting shapes for the files the verification layer needs. These formats are this skill's own convention. Change them to fit the repo, but keep the parts marked **required**.

## 1. The `verify` skill

Path: `.claude/skills/verify/SKILL.md`, next to the code it covers.

```markdown
---
name: verify
description: Start <app> and smoke-test it. Use before every commit that touches <paths>.
---

# Verify <app>

## Start
1. `<install command, only if needed>`
2. `<start command>` and wait for `<the line or URL that means it is ready>`

## Smoke test
1. Open `<url or screen>`. Expect `<what is visible>`.
2. `<one action on the most important flow>`. Expect `<result>`.
3. `<one action on the area you changed>`. Expect `<result from the feature map>`.

## Evidence
Save a screenshot or the command output. Report the failing step by number.

## Stop
`<how to shut it down and free the port>`
```

Before trusting it, run it five times in a row. It should give the same result each time and finish quickly enough to run on every commit.

## 2. Feature map entry

Path: `docs/feature-map.md` (or wherever the repo keeps product docs).

```markdown
### Pin a session
- **User goal:** keep an important session at the top of the list
- **Path:** sidebar → session row → menu → Pin
- **Shortcut:** `Ctrl+Shift+P`
- **Selectors:** `[data-testid="session-row"]`, `[data-testid="pin-toggle"]`
- **Expected:** the row moves to the top, shows a pin icon, and stays on top after a restart
```

Write it from the user's side. "Expected" is the rubric an agent grades itself against.

## 3. Reliability checklist item

Path: `docs/reliability-checklist.md`. All three parts are **required**.

```markdown
### Session format changes load old data
- **Trigger (from the diff):** a change to the session or message types, or to the code that reads or writes them
- **Check:** load a file saved before this change. Does every new field have a safe default when it is missing?
- **Why it exists:** <incident or PR link>, where <one sentence on what broke>
```

An item with no incident or PR behind it does not go in the list. To use the list, test each trigger against the diff and run only the matches.

## 4. REVIEW.md

Path: `REVIEW.md` at the repo root. Always read from the default branch.

```markdown
# Review guide for <repo>

## Always flag (Important)
- <rule>. Seen in <PR>, <PR>.

## Flag as a nit
- <rule>

## Do not flag
- <a pattern that looks wrong here but is intended, and why>

## Needs the code owner
- <path or pattern>: <the reason a human must look>
```

Each rule should be visible from a diff alone and name at least two PRs where it mattered. The "Do not flag" section is what stops over-flagging.

## 5. Commit stamp

A git trailer at the end of the commit message. **Required** on every commit, including when nothing was verified.

```
Verified-By: run, e2e, checklist(2)
```

| Word | Means |
| --- | --- |
| `run` | Started the real app and exercised the change |
| `e2e` | An end-to-end test covers the change and passed |
| `property` | A property test covers the change and passed |
| `differential` | Old and new were compared on the same inputs |
| `tests` | The unit or integration suite passed |
| `review` | A code review ran on this exact commit |
| `checklist(n)` | n triggered reliability items were checked |
| `none` | Nothing was verified |

With `none`, add the reason:

```
Verified-By: none
Verification-Skipped: docs-only change, nothing to run
```

Find unverified commits later with:

```bash
git log --grep='^Verified-By: none' --format='%h %ad %s' --date=short
```

## 6. Commit gate

The gate checks, before a commit, whether the session ran a review or a verify step.

| Setting | Value |
| --- | --- |
| Mode | `off`, `observe` or `enforce` |
| Passes when | the session ran a review or verify step, or the commit message carries `Verification-Skipped:` with a reason |
| Escape | after two blocks in one session, the next commit passes and is logged |

Roll out in `observe` first. Read the log, fix the false positives, then switch to `enforce`. Keep the gate's config on the default branch so a change can't turn off its own gate.

## 7. Reviewer eval case

```markdown
### <incident id>
- **Change:** <PR or commit>
- **Answer key:** <one sentence naming the bug>
- **Set:** tuning | test
- **Kind:** incident | clean control
```

Pair every incident case with a clean merged change of similar size. A reviewer that flags the clean control is over-flagging.
