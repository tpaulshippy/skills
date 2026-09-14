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
  `@coderabbitai full review` for a complete pass from scratch. Each run
  (auto or manual) uses one PR review from your rolling hourly allowance.
- Free for OSS: every public repo gets reviews free (Team features, no card).
  OSS limits are tight and scale with stars: 1–10 PR reviews per developer
  per hour (rolling, additionally scoped per repo), 100–300 files/review.
  Batch changes and trigger once when ready.
- Eligible repos (>=10 stars, auto-review enabled): CodeRabbit auto
  re-reviews after each push (`auto_incremental_review: true`). Manual
  trigger is mainly needed when auto-review is disabled/paused, the author
  is skipped, or no review appears.
- Small repos (<10 stars): NO automatic reviews ("does not receive automatic
  reviews because it has fewer than 10 stars"). Trigger manually after each
  ready state: `gh pr comment` above, or the Trigger review button in the
  CodeRabbit status comment.
- If no review appears within a few minutes, CodeRabbit is probably not
  installed/enabled for the repo. Say so, fall back to local
  `coderabbit review` CLI + careful self-review, and note it on the PR.
- If a manual trigger answers "Review rate limited": the rolling allowance
  is spent. Wait for refill, avoid repeated triggers (each attempt can
  count), use CLI + self-review meanwhile.
- Auto pauses after several reviewed commits: `@coderabbitai resume`
  restarts it. Incremental triggers only cover new commits since the last
  review — they never re-hash already-reviewed commits.

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
- Run the affected test files plus lint; run the repo's migration/model
  consistency check if models changed.
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

- Eligible repos: CodeRabbit auto re-reviews on push. After every push, poll
  and wait for the fresh incremental review.
- Small repos (<10 stars): no auto-review, so manually trigger once per
  ready state (`gh pr comment <PR> --body "@coderabbitai review"`), then
  poll. Do NOT trigger after every small push — allowance is 1–10/hr.
- Stop when a fresh review on the current HEAD raises no new actionable
  items (or only repeats of already-addressed ones). Report the final
  review state with commit SHAs as evidence.
- Never merge unless explicitly asked.
