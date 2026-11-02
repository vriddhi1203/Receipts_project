# INSTRUCTIONS.md

**How we work on this repo.** Read it once now, come back when you're unsure.

---

## 0. The whole loop, in one block

One issue, start to finish. Everything else in this file is detail.

```bash
# 1. Pick an issue on the board, move it to In progress
# 2. Branch off develop
git checkout develop && git pull
git checkout -b feat/12-pdf-loader

# 3. Work. Commit as you go.
git commit -m "feat(loaders): add PdfLoader for text extraction"

# 4. Before you open the PR, catch up with develop
git fetch origin && git rebase origin/develop

# 5. Push and open a PR into develop
git push -u origin feat/12-pdf-loader

# 6. PR description says what, why, how you tested, and "Closes #12"
# 7. Move the card to In review. I review.
# 8. Squash and merge. Branch is deleted. Issue closes itself. Card → Done.
```

That's the loop. Nine steps, every single time, including for a typo fix.

---

## 1. Branches

Two permanent branches. Neither one is ever committed to directly.

| Branch | What it means | Who writes to it |
|---|---|---|
| `main` | Released, known-good. Every commit here should run. | Only merges from `develop` |
| `develop` | The latest integrated state. Where your work lands. | Only merges from feature branches |

**Why two?** `develop` is where things get integrated and occasionally break.
`main` is the version you'd show someone without checking first. Keeping them
separate means you can experiment freely without ever having a broken repo to
hand to a reviewer.

**Rules**

- Both are protected. No direct pushes, no force pushes, no deletion.
- `develop` should always be runnable. If `make up` is broken on `develop`,
  fixing it is the highest-priority thing in the repo.
- `main` gets updated at the end of each sprint, from `develop`, via a PR.
- Never rebase `main` or `develop`. Rebase is for your own unmerged branches only.

---

## 2. Feature branches

### Naming

```
<type>/<issue-number>-<short-slug>
```

| Type | Use for | Example |
|---|---|---|
| `feat` | new behaviour | `feat/12-pdf-loader` |
| `fix` | something is wrong | `fix/31-dropped-final-chunk` |
| `refactor` | same behaviour, better shape | `refactor/40-extract-embedder-port` |
| `test` | tests only | `test/44-chunker-edge-cases` |
| `docs` | documentation only | `docs/8-corpus-section` |
| `chore` | tooling, config, deps | `chore/5-import-linter` |

Lowercase, hyphens, no spaces. Three or four words in the slug, maximum.

**Why the issue number?** Six weeks from now, `feat/12-pdf-loader` tells you
exactly which conversation produced it. `pdf-stuff` tells you nothing.

### Rules

1. **Always branch off `develop`**, never off `main` and never off another
   feature branch. Branching off a branch that later gets rebased is how you
   end up with a history nobody can read.
2. **One issue, one branch, one PR.** If you find a second thing to fix while
   you're in there, open an issue for it and do it separately.
3. **Every branch is linked to an issue.** No issue, no branch. If the work is
   worth doing, it's worth writing down first.
4. **Rebase onto `develop` often** — at least daily, and always right before
   you open the PR. A ten-minute conflict today is a two-hour conflict next week.

### Rebasing, concretely

```bash
git fetch origin
git rebase origin/develop

# If there are conflicts: fix the files, then
git add <file>
git rebase --continue

# Your history changed, so the push needs a force — use this one:
git push --force-with-lease
```

**`--force-with-lease`, never plain `--force`.** With-lease refuses to push if
someone else has pushed to your branch since you last fetched. Plain `--force`
happily destroys their work. Make it a habit now; on a real team it matters.

---

## 3. Branch protection settings

Set these on GitHub under **Settings → Branches**. Applied to both `main` and
`develop`.

| Setting | On? | Why |
|---|---|---|
| Require a pull request before merging | ✅ | Nothing lands unreviewed |
| Require 1 approval | ✅ | That's me |
| Dismiss stale approvals on new commits | ✅ | An approval covers what I actually read |
| Require status checks to pass | ✅ | `pytest` and `lint-imports` |
| Require branches to be up to date before merging | ✅ | You rebased; CI ran on the real result |
| Require conversation resolution | ✅ | No review comment gets quietly ignored |
| Require linear history | ✅ | We rebase, so history stays a straight line |
| Block force pushes | ✅ | History on shared branches is permanent |
| Block deletions | ✅ | Obvious in hindsight, painful in the moment |

> **Set `develop` as the repository's default branch.** GitHub only auto-closes
> issues when a PR merges into the default branch. If `main` is the default and
> your PRs go into `develop`, your issues will never close themselves and the
> board will drift out of sync with reality. See §6.

---

## 4. Commits

### Format

```
<type>(<scope>): <summary in the imperative>

<optional body: why, not what>
```

```
feat(loaders): add PdfLoader for text extraction
fix(chunking): keep the final chunk when text ends mid-token
refactor(embedding): move batching behind the Embedder port
docs(readme): describe the corpus and why it was chosen
chore(ci): run lint-imports on every push
```

Same `type` values as the branch names.

### Rules

- **Imperative mood.** "add", not "added" or "adds". Read it as *"this commit
  will… add PdfLoader"*.
- **Summary under 72 characters.** No full stop at the end.
- **The body explains why.** The diff already says what changed; it can't say
  what you were thinking.
- **One logical change per commit.** Reformatting 40 files and fixing a bug in
  the same commit makes the bug fix impossible to find.
- **Commit every day you work.** Even three lines. A commit history with daily
  small commits is more convincing than one heroic `initial commit`.

| Don't | Do |
|---|---|
| `update` | `feat(cli): add --version flag` |
| `fixed bug` | `fix(chunking): keep the final chunk when text ends mid-token` |
| `changes as discussed` | `refactor(llm): extract retry logic into its own module` |
| `wip` | Don't push `wip`. Squash it before the PR. |

---

## 5. Pull requests

### Title

Same format as a commit: `feat(loaders): add PdfLoader for text extraction`

### Description

Create `.github/pull_request_template.md` once, and GitHub fills this in for
you every time:

```markdown
## What changed


## Why


## How I tested it


## What I'm unsure about


Closes #
```

**Write the description before you think you're finished.** Half the time,
writing "how I tested it" is what makes you realise you didn't.

The **"what I'm unsure about"** section is not optional and not a weakness.
It's the most useful thing in the PR — it tells me where to look, and it's the
difference between a review that rubber-stamps and one that teaches you
something.

### Rules

- **Small PRs.** Over ~400 changed lines and it's probably two PRs. Big PRs get
  shallow reviews; that's not a discipline problem, it's how attention works.
- **Self-review first.** Open the "Files changed" tab and read your own diff
  before requesting review. You'll catch the debug print.
- **CI must be green.** Don't request review on a red build.
- **Respond to every comment**, even if just "done". Resolve the thread once
  it's addressed.
- **Squash and merge.** One issue becomes one commit on `develop`. The messy
  middle stays in the PR where it belongs.
- **Never merge your own PR.** Wait for the approval.

---

## 6. Issues

Everything starts as an issue. If it isn't an issue, it doesn't exist.

### Types

Use labels, one type per issue:

| Label | For |
|---|---|
| `feat` | New behaviour |
| `bug` | Something is broken |
| `chore` | Tooling, config, dependencies |
| `docs` | Documentation |
| `spike` | Time-boxed investigation — the deliverable is an answer, not code |
| `blocked` | Waiting on something. Say what, in a comment. |

Plus a `week-N` label so the board can be filtered by sprint.

### Writing a good issue

Short is fine. Vague is not.

```markdown
**Why**
One or two lines. What problem does this solve?

**Tasks**
- [ ] ...
- [ ] ...

**Done when**
- [ ] Something observable — a command that runs, output that appears
```

**"Done when" is the important part.** It has to be something you could
demonstrate, not something you'd claim. "Loader works" is a claim.
`receipts ingest handbook.pdf` prints a row count is a demonstration.

### Guidelines

- **Title in the imperative:** "Add the PDF loader", not "PDF loader" or
  "PDF loader needed".
- **One deliverable per issue.** If "done when" has two unrelated bullets,
  it's two issues.
- **Half a day or less.** Bigger than that, split it. You'll feel progress and
  I'll see it.
- **Found something while working? New issue.** Don't fold it into the branch
  you're on.
- **Blocked? Comment on the issue**, don't message me. The comment is a record
  of what you tried, and it's where the answer belongs.

---

## 7. Linking issues to PRs

Put a closing keyword in the **PR description** (not the title, not a comment):

```
Closes #12
```

`Closes`, `Fixes` and `Resolves` all work. Multiple issues need the keyword
repeated: `Closes #12, closes #13`.

When the PR merges, the issue closes itself and the board card moves. That's
the whole point — the board stays accurate without anyone maintaining it.

> **The gotcha that catches everyone:** GitHub only auto-closes issues when the
> PR merges into the **default branch**. Our PRs go into `develop`, so
> `develop` must be the repository's default branch or nothing ever closes.
> Set it in **Settings → General → Default branch**.

To reference an issue *without* closing it — "related to, but doesn't finish" —
just write `#12` with no keyword.

---

## 8. The board

GitHub Projects, one board, filtered by `week-N`. Reviewed every two weeks.

| Column | Means | Card moves here when |
|---|---|---|
| **Backlog** | In this sprint, not started | The issue is created |
| **Todo** | Picked for this week | You decide to do it next |
| **In progress** | Being worked on right now | You create the branch |
| **In review** | PR is open | You open the PR |
| **Blocked** | Stuck, waiting | You've been stuck 45+ minutes |
| **Done** | Merged into `develop` | Automatically, on merge |

### Rules

- **One card in "In progress" at a time.** Two half-finished branches is
  slower than two finished ones, every time.
- **Move the card when the thing happens**, not at the end of the day. A board
  that's updated retrospectively is a board nobody trusts.
- **Blocked is a real column, use it.** Moving a card to Blocked with a comment
  saying what you tried is the correct move. Sitting on a problem for two days
  is the only real failure mode here.
- **No card without an issue.** Notes go in the issue, not the card.

---

## 9. General guidelines

1. **Commit every day you work**, even three lines.
2. **The 45-minute rule.** Stuck for three quarters of an hour? Comment on the
   issue with what you've tried and what you expected. Then keep going on
   something else. This isn't giving up — it's the professional move.
3. **Never commit a secret.** And if you ever do: rotate the key immediately.
   Deleting the commit doesn't help, it's already in the history and possibly
   in someone's clone.
4. **`develop` always runs.** Breaking it is fine. Leaving it broken is not.
5. **Use AI to explain, review and debug — not to produce code you can't walk
   me through line by line.** The review questions are how we find out which
   one happened. Nobody is testing you; it just doesn't work otherwise.
6. **Ask early, ask specifically.** "The embedder doesn't work" is hard to
   help with. "`encode()` returns shape (1, 384) but the repo expects (384,)
   and I can't tell which is wrong" gets an answer in one message.
7. **When you disagree with a review comment, say so.** Explaining why you did
   it your way is a better outcome than silently changing it.

---

## Quick reference

```bash
# start
git checkout develop && git pull
git checkout -b feat/12-pdf-loader

# work
git add -p
git commit -m "feat(loaders): add PdfLoader for text extraction"

# catch up
git fetch origin && git rebase origin/develop
git push --force-with-lease

# ship
git push -u origin feat/12-pdf-loader
# open PR into develop · "Closes #12" in the description · move card to In review
```

**Branch:** `type/issue-slug` off `develop`
**Commit:** `type(scope): imperative summary`
**PR:** what · why · how tested · what I'm unsure about · `Closes #N`
**Stuck 45 min:** comment on the issue
