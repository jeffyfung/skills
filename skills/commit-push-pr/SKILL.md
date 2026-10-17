---
name: commit-push-pr
description: Commit current changes, push the branch, and open a PR against main or a stacked base branch. Use when the user wants to commit their changes, push a branch, or create/open a pull request.
---

# Commit, Push, and Open a PR

Ships work on the current feature branch: commit → push → PR. The confirm gate (step 6) gates the push — never skip it, even if invoked back-to-back in the same turn.

Takes one optional argument: the base branch. Without it, step 5 resolves the base.

## 1. Guard the branch

`git branch --show-current`. If it's `main`/`master`, stop and tell the user to create a feature branch first — never commit or push directly to `main`.

## 2. Scope the commit

`git status` + `git diff`. Stage only files that belong to the current task. Untracked files that look like scratch output, generated data, or local test fixtures (CSVs, `*_results/*`, logs) are not auto-staged — list them and ask the user whether to include, ignore, or gitignore.

## 3. Pre-commit checks

Look up the repo's own format/lint commands — check `AGENTS.md`, `CLAUDE.md`, and `README.md` (in that order) for a documented formatter/linter and run whatever's specified there. Don't default to `dotnet format` or any other tool-specific guess if none of those docs name one — note that no formatter/linter is documented and move on.

If the change touches tested code and a fast targeted test run is available, run it; don't run the full suite just to commit.

## 4. Commit message

Conventional Commits (`feat|fix|test|docs|refactor(scope): summary`), matching the style already in `git log` for this repo. Body only when the "why" isn't obvious from the diff.

## 5. Resolve the base branch

The base is `main` unless the work is **stacked** — built on a feature branch that hasn't landed yet, where basing on `main` would drag the branch below into this diff.

1. **Argument given** — that's the base. Verify it with `git rev-parse --verify origin/<base>`; if the ref doesn't exist, say so and use `main`.
2. **No argument** — hunt one candidate. Any of:
   - The conversation says this work sits on commits made on another branch.
   - The work comes from a ticket that is blocked by, or depends on, another ticket — the branch resolving that blocker is the candidate.
   - `git log --oneline origin/main..HEAD` shows commits this task didn't produce; `git branch -r --contains <sha>` names where they came from.
3. **Validate the candidate.** It must exist on the remote, differ from the current branch, and be absent from `git branch -r --merged origin/main` — a candidate already merged into `main` is dead.
4. **No candidate, or it fails validation** — `main`.

A validated candidate is a suggestion, not a decision: offer it alongside `main` at the gate in step 6 and let the user pick.

## 6. Confirm gate — push

Show the user the commit message, `git diff --stat` (or the commit once made), and the base from step 5 — both options when step 5 found a candidate. Get explicit go-ahead before `git push -u origin <branch>`. Pushing publishes the branch — treat it as one-way. PR creation rides on that same confirmation (see step 7) and needs no separate approval.

## 7. Open the PR

Check first whether a PR already exists for this branch: `gh pr list --head <branch> --state all`.

- **Open PR found** — reuse it, don't open a duplicate. Report its URL.
- **Merged PR found** — the branch was already shipped once; stop and ask the user what they want before creating another PR (new commits since merge are unusual, not the default case).
- **No existing/merged PR** — the push confirmation from step 6 already covers this: draft the body and run `gh pr create --base <base> --head <branch> --title "<title>" --body "<body>"` straight away, no extra approval needed.

**Screenshots for visible UI changes.** If the change alters anything a user can see — a new text field, a changed colour, new layout or copy, a new page or component — capture screenshots of the happy path and the edge cases the change introduces (empty and error states, long/overflowing text, responsive vs desktop where relevant). Attach them to the PR body under a `## Screenshots` section. Skip only when the change is invisible to the user (logic, tests, config, docs, CI).

Before drafting, check the repo for its own PR description standard — `.github/PULL_REQUEST_TEMPLATE.md`, `.github/pull_request_template.md`, `docs/*` or `CONTRIBUTING.md` guidance on PR bodies. A repo-documented template overrides the one below; use it instead.

### PR body template (fallback — repo standard wins if one exists)

```text
## Summary
<what changed and why, 1-6 sentences>

## Changes
- <notable change 1>
- <notable change 2>

## Screenshots
<for visible UI changes: happy path + relevant edge cases; omit for invisible changes>

## Design decisions
<notable tradeoffs or rejected alternatives — only for large functional changes, see below>

## Testing
<commands run / how verified>

## Follow-ups
<deferred work, or "None">
```

When the base isn't `main`, open the Summary by naming the base branch and what has to land first — the reviewer needs to know the diff is stacked and that the PR retargets to `main` once the base merges.

Include design decisions only when the PR introduces new functional behavior of real size (a new module, a non-trivial algorithm, a tradeoff a reviewer would otherwise have to ask about) — omit the section entirely for docs/config/chore/small fixes rather than leaving it empty.

## 8. Report

Give the user the PR URL and the base it targets. State plainly what was committed, pushed, and opened — no hedging.
