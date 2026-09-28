---
name: review-babysit
description: Use when asked to request a code review on a GitHub pull request, babysit a PR, address cubic (cubic-dev-ai) review comments, or loop on feedback until clean.
---

# cubic Review Babysit

Drive a GitHub PR to zero actionable cubic feedback: request review, address
every comment in the PR's worktree, re-request, repeat.

The reviewer is the `cubic-dev-ai` GitHub App, which posts as
`cubic-dev-ai[bot]`. Docs: https://docs.cubic.dev.

## 1. Request the review (documented method)

```bash
gh pr comment <PR> --body "@cubic-dev-ai review this PR"      # full review
gh pr comment <PR> --body "@cubic-dev-ai incremental review"  # since last completed review
```

- A new PR is reviewed automatically within minutes. Trigger manually when
  automatic review did not run, or to cover commits pushed after it.
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
  before putting full reviews in a loop.

## 2. Poll for feedback

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

- cubic submits a **new** review on every re-review instead of editing the
  previous one, so read `last`, not all of them. Its headline is the verdict:
  `N issues found across M files`, `0 issues found`, or `All reported issues
  were addressed`. Incremental runs add `(changes from recent commits)`, and a
  confidence score out of 5 when merge confidence is enabled.
- **Staleness:** compare each review's `commit_id` with HEAD
  (`gh pr view <PR> --json headRefOid`). A review on an older commit is
  superseded — wait for or trigger a fresh one and judge only that. This is
  normal: incremental reviews only ever cover the newer commits.
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
  runs count nothing.
- Every review is capped at 50,000 reviewable lines. `max_changed_lines` only
  lowers the automatic trigger, never the ceiling.
- Flex capacity buys 10,000 extra lines for $20, only when a review would
  otherwise pause, bounded by the monthly spend limit in
  **Settings → Subscription**.
- Cut usage instead: ignore generated/test/fixture files, turn off automatic
  incremental reviews, and use the CLI while iterating. Nonprofits and schools
  get 50% off — ask support.

## 7. Re-request and loop

- Default to `@cubic-dev-ai incremental review` for the next cycle: the PR is
  usually clean apart from your own new commits, and incremental reviews are
  the cheap ones.
- Escalate to a full `@cubic-dev-ai review this PR` when the incremental
  baseline was lost (force-push with renames or binary changes, no completed
  review), the change grew a lot, or you want a whole-PR second opinion.
  Append `ultrareview` for that final high-risk pass.
- Thread replies and new pushes do not wake cubic on their own. Replying
  without the tag is treated as discussion and does not authorize cubic to
  edit code. Push your fixes, then wait for the push-triggered incremental
  review when `reviews.incremental_commits` is on — otherwise post a new
  top-level comment with the tag. Confirm the fresh review's `commit_id`
  equals HEAD before judging it.
- Stop when the newest review on HEAD reports `0 issues found` or
  `All reported issues were addressed` and no unresolved cubic thread remains.
  Report the final state with the HEAD SHA and the review SHA as evidence.
- Never merge unless explicitly asked.
