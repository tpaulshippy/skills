---
name: review-babysit
description: Use when asked to request a code review on a GitHub pull request, babysit a PR, address cubic (cubic-dev-ai) review comments, or loop on feedback until clean.
---

# cubic Review Babysit

Drive a GitHub PR to zero actionable cubic feedback: wait for the automatic
review, address every comment in the PR's worktree, push, wait again, repeat.
Manual `@cubic-dev-ai` comments are a last resort, not part of the loop.

The reviewer is the `cubic-dev-ai` GitHub App, which posts as
`cubic-dev-ai[bot]`. Docs: https://docs.cubic.dev.

## 1. Review triggers (automatic by default — manual is the exception)

Manual trigger commands, for reference only — do not use them in the normal loop:

```bash
gh pr comment <PR> --body "@cubic-dev-ai review this PR"      # full review
gh pr comment <PR> --body "@cubic-dev-ai incremental review"  # since last completed review
```

- A new PR is reviewed automatically within minutes, and later pushes are
  re-reviewed automatically while `reviews.incremental_commits` is on (default
  `true`). Do not trigger a review for commits you pushed — wait for the
  automatic one.
- **NEVER post `@cubic-dev-ai incremental review` after your own push while
  automatic review is on.** The push already queued an automatic incremental
  run covering those exact commits. A manual incremental posted on top either
  races it or lands when HEAD already equals the last reviewed commit, and
  cubic replies `There are no new changes to review since the last completed
  review.` — exactly the wasted comment in the screenshot. If you see that
  reply, you triggered unnecessarily: do not retry, just wait for the next
  push's automatic run.
- **One run per PR at a time.** If you post any `@cubic-dev-ai` comment while
  a run is still `in_progress`/`queued`, cubic replies `A review is already in
  progress. Try again after it finishes.` and starts nothing. Before even
  considering a last-resort manual trigger (section 7), poll the HEAD check
  runs (section 2): if any cubic check has `status` `queued` or `in_progress`,
  wait — do not comment. If you already got this reply, do not retry on a
  timer: poll until that check is `completed`, then re-evaluate against the
  last-resort criteria. Most sightings of this reply mean the push-triggered
  automatic run was still working, not a stuck review.
- `@cubic-dev-ai` on its own means a full review; `@cubic` works as a tag too.
  GitHub's @ mention autocomplete will not list it (Apps are excluded), so
  type the mention by hand — which is what `gh pr comment` does for you.
- Incremental covers only the changes since cubic's last completed review. If
  cubic cannot establish a safe range (force-push with renames or binary
  changes, no completed baseline) it explains why and does NOT fall back to a
  full review — ask for `review this PR` explicitly.
- Automatic review is skipped when the PR is a draft
  (`reviews.check_drafts: false` by default), reviews are disabled for the
  repo (`reviews.enabled: false`), or the PR matches an ignore pattern
  (`reviews.ignore.head_branches` / `base_branches` / `pr_labels` /
  `pr_titles` / `max_changed_lines` in `cubic.yaml` or the dashboard). A manual
  request still reviews all of those; only file ignores
  (`reviews.ignore.files`) also apply to manual reviews.
- `reviews.incremental_commits` (default `true`) re-reviews new pushes to open
  PRs. With it off, trigger every cycle yourself.
- Extra guidance in the same comment applies to that run only:
  `@cubic-dev-ai rerun and focus on auth edge cases`,
  `@cubic-dev-ai review this and use <docs-url>`. Durable guidance belongs in
  `cubic.yaml` (`reviews.custom_instructions`, `reviews.custom_rules`) or the
  dashboard, not in a babysit comment.
- Past ~50,000 reviewable changed lines cubic cannot review the PR at all and
  replies with the line count and the largest files. No trigger gets past that:
  ignore generated/fixture files or split the PR.
- `@cubic-dev-ai ultrareview` is a deeper ~30 minute pass billed at 3x the
  reviewed-line rate. Save it for the final high-risk pass, not every cycle.
- **Never ask cubic to fix anything.** "Fix with cubic" or
  `@cubic-dev-ai fix this` makes cubic push its own commits to the PR branch
  (or open a fix PR). This skill does the fixing, in the worktree, with tests.
- Budget: usage is metered in reviewed lines per billing period. Read section 6
  before putting full reviews in a loop. cubic posts its own nudge once it
  notices the pattern — *"You've manually re-run cubic several times on this
  PR. Each manual re-review checks the full PR again and counts toward your
  usage quota. To preserve your usage limits, we recommend letting cubic
  automatically review new commits."* Treat that comment as a hard stop
  against further triggers: let the automatic incremental review land, and use
  the CLI (section 4, unmetered) for any remaining rechecking.

## 2. Poll for feedback (check run first, reviews second)

The cubic **check run on HEAD is the source of truth** for every push — not
the reviews list:

```bash
HEAD=$(gh pr view <PR> --json headRefOid -q .headRefOid)
gh api repos/{owner}/{repo}/commits/$HEAD/check-runs \
  --jq '.check_runs[] | select(.name | test("cubic"))
        | {name, status, conclusion, started: .started_at,
           completed: .completed_at, title: .output.title,
           summary: .output.summary}'
```

- A completed `success` check with a summary like `0 issues found across
  2 files (changes from recent commits)` **is the clean incremental verdict.**
  cubic posts NO new summary review and NO new inline comments in that case
  (verified on `tpaulshippy/app-builder#38` and `#36`: fix pushes got a
  passing check, zero new reviews). Do not keep waiting for a review whose
  `commit_id` equals HEAD — it will never come. Do not post a manual trigger
  to "confirm" it — that is what produces the `There are no new changes to
  review` reply.
- A `status` of `queued` or `in_progress` means the push-triggered run is
  still working — wait and re-poll every 30–60s. Posting any `@cubic-dev-ai`
  comment in this state is what produces the `A review is already in progress`
  reply. Only judge stalled runs after ~10 minutes with no status change
  (normal runs finish in minutes, `ultrareview` ~30 min): then use the
  **Cancel AI review** button in the Checks UI and re-evaluate, rather than
  stacking another comment.
- A summary like `Found 5 issues across 2 files` means findings were posted —
  only then read the newest cubic review and the inline comments:

```bash
# newest cubic review: verdict, commit, full summary body
gh api repos/{owner}/{repo}/pulls/<PR>/reviews \
  --jq '[.[] | select(.user.login == "cubic-dev-ai[bot]")] | last
        | {state, commit: .commit_id[0:7], submitted: .submitted_at,
           headline: (.body | split("\n")[0]), body}'

# inline comments (path/line/context) and the PR conversation
gh api repos/{owner}/{repo}/pulls/<PR>/comments \
  --jq '[.[] | {user: .user.login, created: .created_at, path, line, body: .body[0:800]}]'
gh api repos/{owner}/{repo}/issues/<PR>/comments \
  --jq '[.[] | {user: .user.login, created: .created_at, body: .body[0:300]}]'
```

The summary body embeds a ready-made work list inside a `<details>` block
titled "Prompt for AI agents (unresolved issues)":

````bash
gh api repos/{owner}/{repo}/pulls/<PR>/reviews \
  --jq '[.[] | select(.user.login == "cubic-dev-ai[bot]")] | last
        | .body | split("```text")[1] | split("```")[0]'
````

It reads as `<file name="..."><violation number="1" location="path:line">P2:
text</violation></file>`. Treat that block as the authoritative work list; the
inline comments carry the same findings plus code context.

- cubic posts a summary review plus inline comments only when a run has
  findings worth reporting. A clean incremental run is check-only: no new
  review, no new comments. So read `last` for the verdict only when the HEAD
  check says issues were found. Verdict headlines look like `N issues found
  across M files`, `0 issues found`, or `All reported issues were
  addressed`. The `(changes from recent commits)` marker for incremental runs
  appears in the check-run summary; a confidence score out of 5 is added when
  merge confidence is enabled.
- **Staleness:** a review whose `commit_id` is older than HEAD is superseded —
  but a superseded review plus a passing cubic check on HEAD means the new
  code is clean, not that a review is missing. Judge pushes by the HEAD check
  run (section above), not by waiting for a review on HEAD. Do not post a
  manual trigger to "freshen" a stale review while automatic review is on;
  polling the check is the freshen step.
- GitHub re-anchors still-valid old comments onto HEAD after a push (same
  `created_at`, same cubic `review-run` id, `isOutdated: false`). A familiar
  comment showing HEAD's SHA is NOT a new finding on your new commits — check
  `created_at` and the `<!-- cubic:review-run=... -->` marker before treating
  anything as new. Truly new findings arrive with a new `review-run` id and a
  `Found N issues` check summary.
- Unresolved threads are the real backlog. cubic auto-resolves threads it
  detects as addressed (`reviews.resolve_threads_when_addressed`, default on)
  and incremental reviews post only new issues instead of repeating old ones:

```bash
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!){
    repository(owner:$owner,name:$repo){
      pullRequest(number:$pr){
        reviewThreads(first:100){
          nodes{ isResolved isOutdated
            comments(first:1){ nodes{ path line author{login} body } } } } } } }' \
  -F owner={owner} -F repo={repo} -F pr=<PR> \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved == false)
        | select(.comments.nodes[0].author.login | test("^cubic"))
        | {path: .comments.nodes[0].path, line: .comments.nodes[0].line,
           outdated: .isOutdated, body: .comments.nodes[0].body[0:400]}'
```

- The GraphQL author login is `cubic-dev-ai`, while REST (`reviews`,
  `pulls/comments`) reports `cubic-dev-ai[bot]`. Match `^cubic` rather than one
  exact string.
- Inline comment bodies open with a hidden
  `<!-- metadata:{"confidence":N} -->` line (confidence out of 10) before the
  `P1:`-prefixed finding. Strip it before quoting a comment back.
- To stop an in-flight run, use the **Cancel AI review** button in the Checks
  UI. Canceled and failed runs consume no reviewed lines.

## 3. Address each comment

- Work ONLY in the PR's separate worktree/branch, never on main.
- Fix the code and add/extend regression tests for every actionable item.
- Reject with reasons in-thread (`POST /repos/{owner}/{repo}/pulls/<PR>/comments/<id>/replies`):
  name the precedent or the incommensurable trade-off. Feedback addressed to
  cubic can become a persistent learning, so state the rule, not just the
  outcome — "done" or "thanks" is discarded. Every rejected item stays open,
  so reply and then resolve the thread yourself.
- Priorities run P1 (highest) to P3. cubic's own auto-approval ladder treats
  lower priorities as non-blocking, so read P3 as advisory: push back on it
  explicitly rather than silently leaving it unaddressed.
- Run the affected test files plus lint; run the repo's migration/model
  consistency check if models changed. Commit, push, then reply to each thread
  summarizing the fix and the test that now covers it.

## 4. Local loop with the cubic CLI (optional, unmetered)

```bash
curl -fsSL https://cubic.dev/install | bash   # macOS/Linux; npx @cubic-dev-ai/cli install -g on Windows
cubic                                       # first run signs in via browser
```

```bash
cubic review                          # uncommitted changes
cubic review -b                       # branch vs auto-detected base
cubic review --base main --prompt "focus on the auth changes"
cubic review --json | jq '.issues[]' # machine-readable; exits 1 when it finds issues
```

- Use it for the inner fix/recheck loop: it is fast, it loads the same
  `cubic.yaml`, custom agents and learnings as the GitHub review, and
  **CLI/local reviews do not draw from the reviewed-line pool**. The GitHub
  review uses a different model and pipeline, so it stays the final pass.
- `cubic review --json` also reports `repository_context` (settings, custom
  agents, learnings actually loaded) — worth checking when a finding looks
  like it ignored repo config.
- Optional stronger models: `cubic auth connect codex` or
  `cubic auth connect claude-code` reuses your ChatGPT/Claude subscription
  instead of cubic's included model. Headless and CI runs need a personal
  `CUBIC_API_KEY` (`cbk_...`) instead of the browser flow.

## 5. Keep CI checks green

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

## 6. Cost and limits

Metered in **reviewed lines**: changed lines cubic actually reviewed in
completed PR reviews. Only diff lines count — surrounding context it reads is
free, and generated files, binaries, vendored code and ignored paths are
discounted automatically.

| Plan | Reviewed lines per seat per billing period |
| --- | --- |
| Trial | 20,000 |
| Team | 40,000 |
| Pro | 80,000 |

- Free plan: 20 AI reviews per month, shared across the repos of your GitHub
  org, resetting on the 1st. Review comments show what remains.
- Public repos are free of charge against a separate public fair-use pool
  aimed at abuse, not at normal OSS maintenance. If an open-source project is
  blocked wrongly, email contact@cubic.dev with org, repo, and PR link.
- On paid plans every active seat adds to one team-wide pool. Check
  **Settings → Usage** for the reset date and a breakdown by review type and
  repository; the quota resets with the billing period, not a rolling window.
- Full reruns re-count the same lines; incremental reviews count only new
  changes; Ultrareviews count 3x; canceled, failed, and merge-before-finish
  runs count nothing. A manual request is never free: a repeated
  `review this PR` pays for the whole PR again, and cubic warns about this once
  it sees a manual re-review pattern on one PR.
- Every review is capped at 50,000 reviewable lines. `max_changed_lines` only
  lowers the automatic trigger, never the ceiling.
- Flex capacity buys 10,000 extra lines for $20, only when a review would
  otherwise pause, bounded by the monthly spend limit in
  **Settings → Subscription**.
- Cut usage instead: ignore generated/test/fixture files, turn off automatic
  incremental reviews, and use the CLI while iterating. Nonprofits and schools
  get 50% off — ask support.

## 7. Push, wait, and loop (no re-request)

- **After every push, do nothing but poll the cubic check on HEAD**
  (section 2). The push-triggered incremental costs only the new lines and
  the check starts within a minute, completing in a few minutes. A
  `completed/success` check is the verdict — a clean one posts no review at
  all, and that silence is the pass signal, not a missing review. Wait at
  least 5 minutes before even considering that automatic review "did not
  run". Posting any `@cubic-dev-ai` comment during this window is what
  produces the `A review is already in progress` or `There are no new changes
  to review` replies — never do it.
- **Manual trigger is a last resort, once per PR at most, not once per
  cycle.** Post `incremental review` only when ALL of these hold: (a) no
  cubic check run exists on HEAD after the wait, (b) no cubic check on HEAD
  is `queued`/`in_progress` (an in-flight run fails the test — wait for it),
  (c) you verified automatic review was actually skipped — PR is a draft, reviews disabled,
  branch/label/title/line ignore matched (section 1), or
  `reviews.incremental_commits` is off — and (d) you pushed new commits since
  cubic's last completed review. A completed check on HEAD — clean or not —
  means review ran: never trigger on top of it.
- Reserve a full `@cubic-dev-ai review this PR` for when the incremental
  baseline was lost (force-push with renames or binary changes, no completed
  review), the change grew a lot, or you want a whole-PR second opinion.
  Append `ultrareview` for that final high-risk pass. Once per PR, not once per
  cycle.
- Thread replies and new pushes do not wake cubic on their own. Replying
  without the tag is treated as discussion and does not authorize cubic to
  edit code. Push your fixes, then wait for the push-triggered incremental
  check when `reviews.incremental_commits` is on — otherwise post a new
  top-level comment with the tag. Judge a push by its HEAD check run
  (completed, `success`), never by waiting for a review on HEAD.
- Stop when the cubic check on HEAD is `completed/success` with `0 issues
  found` (or the newest review says `All reported issues were addressed`)
  and no unresolved cubic thread remains. Report the final state with the
  HEAD SHA and the check-run title/summary as evidence.
- Never merge unless explicitly asked.
