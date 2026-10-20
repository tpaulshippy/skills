---
name: review-babysit
description: Use when asked to request a code review on a GitHub pull request, babysit a PR, address review comments, or loop on feedback until clean.
---

# Copilot Review Babysit

Drive a GitHub PR to zero actionable Copilot feedback: request review,
address every comment in the PR's worktree, re-request, repeat.

## 0. Resolve the target repo

- Derive `<owner>/<repo>` per
  [`_shared/pr-link-construction.md`](../_shared/pr-link-construction.md)
  (explicit repo, PR URL, or the PR checkout's remote — never the ambient
  cwd). Ask if unknown.
- Pass `--repo <owner>/<repo>` to every `gh pr`/`gh run` command and use
  explicit `repos/<owner>/<repo>/...` paths for `gh api` (no `{owner}/{repo}`
  placeholders).

```bash
R=<owner>/<repo>
gh pr view <PR> --repo $R --json url,title,state,baseRefName
```

## 1. Request the review (documented method) unless it was already requested.

- Skip the request if Copilot has a pending request OR already reviewed the
  current HEAD. Reviews on older commits are stale and do not count.

```bash
HEAD=$(gh pr view <PR> --repo $R --json headRefOid --jq .headRefOid)
# pending: non-empty list => skip
gh api repos/$R/pulls/<PR>/requested_reviewers \
  --jq '[.users[].login | select(ascii_downcase | startswith("copilot"))]'
# reviewed current HEAD: count > 0 => skip
gh api --paginate --slurp repos/$R/pulls/<PR>/reviews \
  | jq "[.[][] | select((.user.login | startswith(\"copilot-pull-request-reviewer\")) and .commit_id == \"$HEAD\")] | length"
```

```bash
gh pr edit <PR> --repo $R --add-reviewer @copilot
```

- This is the official CLI path (GitHub Changelog 2026-03-11). Do NOT use
  `/review` issue comments (no-op) and do NOT guess reviewer logins via the
  raw REST API (`copilot-pull-request-reviewer` returns 422 unless Copilot
  code review is enabled; the `[bot]` variant can silently do nothing).
- If the request errors or no review appears within ~3 minutes, Copilot
  code review is probably not enabled for the repo. Say so, fall back to a
  careful self-review, and note it on the PR.
- Copilot reviews take <3m normally but allow minutes. It leaves `COMMENTED`
  reviews only — never approve/request-changes, never merge-blocking.

## 2. Poll for feedback

```bash
gh api --paginate repos/$R/pulls/<PR>/reviews \
  --jq '.[] | {id, user: .user.login, submitted: .submitted_at, commit: .commit_id, body}'
gh api --paginate repos/$R/pulls/<PR>/comments \
  --jq '.[] | {id, review: .pull_request_review_id, reply_to: .in_reply_to_id, user: .user.login, commit: .commit_id, path, line, body}'
gh api --paginate repos/$R/issues/<PR>/comments \
  --jq '.[] | {id, user: .user.login, created: .created_at, body}'
```

- Keep full bodies and IDs — never truncate. Match inline comments to the
  fresh review via `pull_request_review_id`; use `id` for replies.
- Filter by reviewer login (`copilot-pull-request-reviewer[bot]` for
  reviews, `Copilot` for inline comments) — your own replies also create
  reviews and must not count as Copilot's.

- Check each review's `commit_id`: a review on an older commit than HEAD is
  stale for step 1's skip check — re-request on the current HEAD. Its
  findings are NOT dropped: carry them forward until each is fixed and
  verified or rejected with a rationale.
- Copilot may repeat already-resolved comments on re-review (documented).
  Do not churn on repeats; reply pointing at the existing fix.
- Findings can live only in the review `body` (`### Suppressed comments`,
  "previously missed issues") — read the full body, not just inline
  threads. With no thread to reply to, answer in one PR comment
  (`gh pr comment <PR> --repo $R`).

## 3. Address each comment

- Work ONLY in the PR's separate worktree/branch, never on the base branch.
- Fix code + add/extend regression tests for every actionable item. If a
  comment is rejected, say why (precedent, incommensurable trade-off) in a
  thread reply — Copilot accepts explicit contract changes as resolution.
- Run the affected test files plus lint (e.g. RSpec + RuboCop for Rails,
  Jest + ESLint for React); if migrations changed, confirm `schema.rb` is
  updated.
- Commit, push, then reply to each thread (`POST
  repos/$R/pulls/<PR>/comments/<id>/replies`) summarizing the fix and test.

## 4. Keep CI checks green

Poll checks on every push — review feedback is not the only gate:

```bash
gh pr checks <PR> --repo $R
```

- On any failure, fetch the log by CI provider (check the details URL from
  `gh pr checks <PR> --repo $R --json name,state,link`):
  - GitHub Actions: `gh run view <RUN_ID> --repo $R --job <JOB_ID> --log-failed`.
  - CircleCI: open the job from the details link (see
    `docs/workflow/continuous-integration.md`) or use the `circleci` CLI.
  - Otherwise / no access: ask the user for the failing output.
  Grep past the runner preamble for `FAILED`, `ERROR`, or assertion lines.
- If the PR is behind its base, has merge conflicts, or CI fails on
  merge-related causes: read `baseRefName` (step 0), find the remote that
  points at `$R` (`git remote -v`; not necessarily `origin`), then
  `git fetch <remote> <base> && git merge <remote>/<base>` in the worktree,
  resolve, re-run affected suites + lint, push.
- Re-run the affected suites + lint locally, commit, push, and confirm
  `gh pr checks` goes green before asking for re-review.

## 5. Re-request and loop

- After every push, rerun step 1's checks: the repo may have "Review new
  pushes" enabled, so Copilot can auto re-review. Request manually only
  when there is neither a pending request nor a Copilot review on the new
  HEAD, then poll again.
- A review whose body says Copilot was unable to review (e.g. quota limit)
  is a failure, not a clean pass — don't let it satisfy step 1's check or
  the stop condition; fall back to self-review as in step 1.
- Stop when a fresh review on the current HEAD raises no new actionable
  items (or only repeats of already-addressed ones) AND no earlier finding
  is still outstanding. Report the final
  review state with commit SHAs as evidence.
- Never merge unless explicitly asked.