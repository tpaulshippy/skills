---
name: review-babysit
description: Use when asked to request a code review on a GitHub pull request, babysit a PR, address review comments, or loop on feedback until clean.
---

# CodeRabbit Review Babysit

Drive a GitHub PR to zero actionable CodeRabbit feedback: request review,
address every comment in the PR's worktree, re-request, repeat.

## 1. Request the review (documented method)

```bash
gh pr comment <PR> --body "@coderabbitai review"
```

- Use `@coderabbitai review` for an incremental review of new changes,
  `@coderabbitai full review` for a complete pass from scratch.
- CodeRabbit auto re-reviews after each push by default
  (`auto_incremental_review: true`), unlike Copilot. Manual trigger is
  mainly needed when auto-review is disabled, the author is skipped, or no
  review appears.
- If no review appears within a few minutes, CodeRabbit is probably not
  installed/enabled for the repo. Say so, fall back to a careful
  self-review, and note it on the PR.

## 2. Poll for feedback

```bash
gh api repos/{owner}/{repo}/pulls/<PR>/reviews \
  --jq '[.[] | {user: .user.login, submitted: .submitted_at, commit: .commit_id[0:7], headline: .body[0:200]}]'
gh api repos/{owner}/{repo}/pulls/<PR>/comments \
  --jq '[.[] | {user: .user.login, created: .created_at, path, body: .body[0:800]}]'
gh api repos/{owner}/{repo}/issues/<PR>/comments \
  --jq '[.[] | {user: .user.login, created: .created_at, body: .body[0:300]}]'
```

- Reviewer login is `coderabbitai` / `coderabbitai[bot]` — filter poll
  results on that. Check each review's `commit_id`: a review on an older
  commit than HEAD is stale — wait for / trigger a fresh review on the
  current HEAD and judge only that.
- CodeRabbit may repeat already-resolved comments on re-review.
  Do not churn on repeats; reply pointing at the existing fix.

## 3. Address each comment

- Work ONLY in the PR's separate worktree/branch, never on main.
- Fix code + add/extend regression tests for every actionable item. If a
  comment is rejected, say why (precedent, incommensurable trade-off) in a
  thread reply and in code comments.
- Run the affected test files plus lint; also verify `makemigrations
  --check` if models changed (needs `CSRF_TRUSTED_ORIGINS=https://example.com`
  in this repo's env).
- Commit, push, then reply to each thread (`POST
  /pulls/<PR>/comments/<id>/replies`) summarizing the fix and test.

## 4. Keep CI checks green

Poll checks on every push — review feedback is not the only gate:

```bash
gh pr checks <PR>
```

- On any failure, fetch the failed job log and fix in the worktree:
  `gh run view <RUN_ID> --job <JOB_ID> --log-failed`, grepping past the
  runner preamble for `FAILED`, `ERROR`, or assertion lines.
- If the PR is behind main, has merge conflicts, or CI fails on merge-related
  causes: `git fetch origin && git merge origin/main` in the worktree,
  resolve, re-run affected suites + lint, push.
- Re-run the affected suites + lint locally, commit, push, and confirm
  `gh pr checks` goes green before asking for re-review.

## 5. Re-request and loop

- CodeRabbit DOES auto re-review on push by default. After every push, poll
  and wait for the fresh incremental review. Only comment
  `gh pr comment <PR> --body "@coderabbitai review"` manually if no fresh
  review appears.
- Stop when a fresh review on the current HEAD raises no new actionable
  items (or only repeats of already-addressed ones). Report the final
  review state with commit SHAs as evidence.
- Never merge unless explicitly asked.
