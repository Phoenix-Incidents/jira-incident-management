---
name: pr-review
description: "Full code review of a GitHub PR, posted as ONE review comment headed 'Mergeable' or 'Not ready to merge', with the PR marked 'changes requested' when it is not ready. Use when the user asks to review a PR (URL or number) and post the result, review someone's PR, or re-review a PR after the author pushed changes."
metadata:
  version: 1.0.0
---
# PR Review

A full code review of a GitHub PR. It covers bugs, security, and the repo's own standards. The result goes on the PR as one review comment.

## Inputs

- A PR URL or number. With a bare number, use the repo of the current directory.
- Optional context from the user (ticket, what the PR is meant to do, a paired PR in another repo).

## Step 1: Get the PR

```
gh pr view <N> -R <owner/repo> --json number,title,author,headRefName,baseRefName,headRefOid,mergeable,reviewDecision,additions,deletions,changedFiles,body,commits,reviews,statusCheckRollup
gh pr diff <N> -R <owner/repo> --name-only
```

- If this PR already has a review from us, this is a **re-review**. Read the earlier review and the author's replies first (`gh api repos/<owner>/<repo>/pulls/<N>/reviews` and `.../issues/<N>/comments`). Then follow "Re-reviews" below instead of a full sweep.
- Check out the PR head in a **detached worktree in the scratchpad**. Never touch the user's primary checkout. Never symlink the user's node_modules into it. Use the PR's own repo and its `baseRefName`, not `main`. Run the git commands from a local clone of **that** repo. If there is none, or the current directory is a different repo, clone it into the scratchpad first. If the worktree path already exists from an earlier run, remove it first.
  ```
  git fetch -q origin <baseRefName> pull/<N>/head:pr-<N>-review
  git worktree add -q --detach <scratchpad>/pr<N> pr-<N>-review
  ```
- Diff against the merge base: `git diff $(git merge-base origin/<baseRefName> HEAD) HEAD`.

## Step 2: Load the standards

Read the repo's `CLAUDE.md` and `AGENTS.md` (and any rules they point to). The review checks the PR against **all** of them, not only bugs. That includes architecture, service placement, naming, comment rules, types and casts, file size limits, duplicated types/constants/logic, test rules, and writing style.

## Step 3: Run the checks

Installing and testing runs the PR's code on this machine, with the user's credentials in reach. Only do it when the author is a member of the org or has write access to the repo (`gh api repos/<owner>/<repo>/collaborators/<login>/permission -q .permission` returns `write`, `maintain` or `admin`). For anyone else, skip this step and tell the user why.

In the worktree, install with the repo's package manager (check `packageManager` in package.json) and run type-check, tests and lint. Run this in the background while reviewing. A failing check is a must-fix item only if the PR caused it. If a check fails, run it on the base branch too. Failures that also happen on the base, or that come from the local setup, go in the report to the user, not in the review.

## Step 4: Review

Do what the built-in `code-review` skill does: read changed files fully, find callers and consumers of anything changed, and check correctness, security, performance and data integrity. Add these on top:

- **Repo standards** from Step 2. Every violation the PR adds is a finding.
- **Comments, as a separate check.** List every comment the PR adds (`git diff <base> HEAD | grep -E '^\+[[:space:]]*(//|/\*|\*)'`) and check each one against the repo's comment rules. Typical findings: a comment that repeats the method or variable name, narration of what the code does, mentions of the ticket, PR, wave or caller, roadmap talk ("until X lands"), "legacy" notes about data that never shipped, JSDoc that no longer matches the code, the wrong JSDoc form, lines wrapped short of the repo's width, and em-dashes. Report them as one grouped item with examples, not one item per comment.
- **Regressions for existing users.** Renamed routes, storage keys, persisted shapes or wire contracts that callers (including other repos) still use.
- **Duplication.** Code copied from an existing module instead of shared.
- **Missing tests** for risky paths, and tests that only cover the happy path.
- **Branch hygiene.** Merge commits from main (we rebase, never merge main in). Commits without the ticket key. Unrelated commits that belong in their own PR.
- **PR description.** Out of date or wrong claims. A missing note about deploy order with a paired PR.
- Only flag what the PR introduces, not old problems.

For large PRs (roughly over 1,500 changed lines), split the files into 3 to 4 areas and give each to a parallel read-only subagent. Each one gets the worktree path, the merge base, and the standards files to read. Each returns findings with severity, file:line and a short fix. Add one more subagent that does only the comment check above across the whole diff, because area reviewers focused on bugs skip comments.

**Verify every finding yourself against the code before it goes in the comment.** Drop anything theoretical or unconfirmed.

## Re-reviews

A re-review is not a fresh full review. Code that did not change and passed last time stays passed. Raising new nits on it every round moves the goalposts and the PR never finishes.

1. **Check every earlier item.** Mark each one fixed, still open, or answered. If the author replied with a reason that holds, accept it and drop the item.
2. **Review only what changed** since the commit we last reviewed (the sha in our last review, or the review's `commit_id`). Use `git diff <old sha> <new head>`. If the author rebased, use `git range-diff <old base>..<old sha> <new base>..<new head>`.
3. **Check what the changes touch.** If a fix changed shared code, check its callers too.
4. **Old code gets must-fix items only.** A real bug or security hole in code we already passed still blocks the merge. Say it was missed last time. Do not add should-fix or nice-to-have items on unchanged code.
5. **Run any check the earlier review skipped** across the whole diff, once. For example, an earlier review with no comment check.

Run type-check, tests and lint again on the new head. For a large set of changes, split it across subagents the same way as Step 4.

## Step 5: Write the comment

One comment. Its header is the verdict:

- `## Mergeable`: no must-fix items.
- `## Not ready to merge`: one or more must-fix items.

Then list only the items that need addressing. No praise, no summary of what the PR does, no filler. If there are many items, group them:

```
## Not ready to merge

### Must fix before merge

1. **Short title.** What is wrong, where (`file.ts`, `functionName`), and what to do instead.

### Should fix

- ...

### Nice to have

- ...
```

With only a few items, a single numbered list is fine. A `Mergeable` review may still list should-fix or nice-to-have items.

On a re-review, start with one line: `Re-reviewed at \`<short sha>\`.` Name the earlier items that are now fixed in one sentence. Then list what is still open or new.

Writing rules for the comment:
- Plain words, short sentences, one idea per sentence. Write for a smart coworker.
- No em-dashes. No jargon a reader would have to look up.
- Name files, classes and functions in backticks so the author can find them.
- Each item says what is wrong and what to do. Keep it to two or three sentences.

## Step 6: Post it

Write the body to a file in the scratchpad, then post it as a single review:

```
# Not ready to merge
gh pr review <N> -R <owner/repo> --request-changes --body-file <file>
# Mergeable
gh pr review <N> -R <owner/repo> --approve --body-file <file>
```

If GitHub refuses `--approve` or `--request-changes` (for example on your own PR), post with `--comment` and tell the user.

Confirm with `gh pr view <N> -R <owner/repo> --json reviewDecision -q .reviewDecision`. Then remove the worktree and the temp branch. Do this on every exit, including when a check, the review or a `gh` command failed:

```
git worktree remove --force <scratchpad>/pr<N>
git branch -D pr-<N>-review
```

## Step 7: Report to the user

Tell the user the verdict, the counts per group, the review link, and the result of the checks from Step 3. Keep it short.

## Rules

- Post exactly one review per run. Do not add inline comments or extra comments.
- Do not push, edit code or change the PR branch. This skill only reviews.
- Do not post to Jira or anywhere else.
