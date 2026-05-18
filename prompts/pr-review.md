# PR Review — magicfit org (multi-agent, high-signal)

You are reviewing a GitHub pull request. The PR number, owner, and repo are available via `$GITHUB_REF`, `$GITHUB_REPOSITORY`, and the `gh` CLI. You have full `gh` access and can post review comments.

**Hard rules (apply to every subagent you launch):**
- All tools work. Do not test tools or make exploratory calls.
- Only call a tool when required.
- Never include a "checked off by Claude" block, signature, or self-praise in any posted comment.
- This is **non-interactive CI**. Always post your findings if any blocking issues exist (no "ask before posting" branches).

## Step 0 — Setup

Determine the PR number:
- If running on a `pull_request` event, the number is in `$GITHUB_REF` (e.g. `refs/pull/42/merge`) and `$GITHUB_EVENT_PATH` has the full payload.
- If running on an `issue_comment` event (re-review via `@claude`), `$GITHUB_EVENT_PATH` has the issue number which IS the PR number for PR comments.

Extract once:
```bash
PR_NUMBER=$(jq -r '.pull_request.number // .issue.number' "$GITHUB_EVENT_PATH")
REPO="$GITHUB_REPOSITORY"
HEAD_SHA=$(gh pr view "$PR_NUMBER" --json headRefOid -q '.headRefOid')
```

Initialize the blocking-issues marker (will be overwritten in Step 9):
```bash
echo "false" > /tmp/claude-blocking-issues
```

## Step 1 — Triage (cheap; bail early when possible)

Launch a haiku agent to check ALL of:
- PR is closed → bail
- PR is draft → bail
- PR author matches `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, or `claude[bot]` → bail
- PR title contains `[skip-claude-review]` → bail
- Claude has already posted a review comment on the LATEST head SHA (check `gh pr view $PR_NUMBER --comments` for a comment whose body contains `## Claude review` AND whose commit reference matches `$HEAD_SHA`) → bail (avoids duplicate runs on the same SHA when re-triggered by label toggles)

**Do NOT bail based on diff size.** Big PRs are where bugs hide; never degrade on size.

If bailing, write a one-line summary comment via `gh pr comment $PR_NUMBER --body "..."` explaining the skip reason, leave `/tmp/claude-blocking-issues` as `false`, and stop.

## Step 2 — Discover repo-specific rules

Launch a haiku agent to return:
- The path of every `CLAUDE.md` in the repo whose path is an ancestor of any file changed in this PR.
- The path of `REVIEW.md` at the repo root (if it exists). This file contains repo-specific reviewer hints — treat it as authoritative for "what to flag" and "what to skip" in this repo.
- The list of changed files (use `gh pr diff $PR_NUMBER --name-only`).

## Step 3 — Summary

Launch a sonnet agent to read the PR (`gh pr view $PR_NUMBER`, `gh pr diff $PR_NUMBER`) and return:
- A 2–3 sentence summary of intent
- A 1-line classification: `feature` / `bugfix` / `refactor` / `chore` / `docs` / `test` / `migration` / `config`
- A list of changed files grouped by area

This summary is passed verbatim to every downstream subagent so they share intent.

## Step 4 — Parallel review (the 4 reviewers — launched in ONE batch)

Launch FOUR agents IN PARALLEL in a SINGLE message. They MUST run inside one tool-call batch so they share the prompt cache (the shared prefix is `[system + every CLAUDE.md + REVIEW.md + PR diff + summary from Step 3]`).

For each subagent, provide as input: the PR title + description, the Step 3 summary, the full diff, the relevant `CLAUDE.md`(s), and `REVIEW.md` if present.

**Agents 1 + 2 — CLAUDE.md compliance (Sonnet, parallel):**
- Audit changes for CLAUDE.md and REVIEW.md compliance.
- Only consider CLAUDE.md files whose path is an ancestor of the file being evaluated.
- Each agent works independently — do NOT share findings.
- Return: list of issues with `{ file, line, description, rule_quote, confidence_0_100 }`.

**Agents 3 + 4 — Bug / logic / security (Opus 4.7, parallel):**
- Use `claude_args: --model claude-opus-4-7` when launching these subagents.
- Scan for real bugs in the introduced code only. Focus on the diff. Do not flag pre-existing issues.
- Each agent works independently.
- Return: list of issues with `{ file, line, description, why_a_bug, confidence_0_100 }`.

**HIGH-SIGNAL bar (all four agents):**
Flag ONLY:
- Code that will fail to compile / parse (syntax, type, missing imports, unresolved references)
- Code that will definitely produce wrong results (clear logic errors regardless of input)
- Clear, unambiguous CLAUDE.md or REVIEW.md violations where you can quote the rule

**Do NOT flag:**
- Code style or quality preferences
- Issues that depend on specific input/state without evidence
- Subjective suggestions or "improvements"
- Anything a linter or `tsc` would catch (CI runs those separately)
- Pre-existing issues outside the diff
- Issues a senior engineer would consider a pedantic nitpick

If you are not certain, do NOT flag. False positives erode trust.

## Step 5 — De-dupe and rank

Merge the 4 agents' findings into one list. Drop duplicates (same `file` + `line` + similar description → keep highest confidence). Rank by `confidence_0_100` descending.

**Cap: at most 15 issues proceed to validation.** Issues 16+ are summarized in a single "additional unverified findings" bullet list at the end of the summary comment — never posted inline.

## Step 6 — Validation (parallel per issue, capped at 15)

For each of the top 15 issues, launch a validation subagent IN PARALLEL (single batch):
- For bug/logic/security issues: Opus 4.7 subagent. Re-read the relevant code in full context and confirm the issue is real with confidence ≥ 80. The validator MUST quote the offending code and explain exactly why it's wrong.
- For CLAUDE.md / REVIEW.md issues: Sonnet subagent. Confirm the rule applies to the file (path scope), confirm the diff actually violates it, and quote the rule text.

A validator's job is to SUPPRESS issues that don't hold up under scrutiny. Suppress aggressively — better to miss a real issue than post a false positive.

## Step 7 — Filter

Keep only issues that validation confirmed at confidence ≥ 80. Drop the rest.

## Step 8 — Post

If the validated set is **empty**:
- Post one summary comment via `gh pr comment $PR_NUMBER --body "..."`:
  ```
  ## Claude review

  Reviewed at `<HEAD_SHA short>`. No blocking issues found.

  Checked: bugs / logic / security (2× Opus 4.7), CLAUDE.md + REVIEW.md compliance (2× Sonnet), with validation pass on every candidate finding.

  <!-- claude-review-marker: head=<HEAD_SHA> -->
  ```
- Leave `/tmp/claude-blocking-issues` as `false`.
- Stop.

If the validated set is **non-empty**:

**Post inline comments via a single PR review:**
```bash
gh api -X POST "repos/$REPO/pulls/$PR_NUMBER/reviews" \
  -F event="COMMENT" \
  -F commit_id="$HEAD_SHA" \
  -F body="## Claude review

Reviewed at \`<HEAD_SHA short>\`. **<N> blocking issue(s)** flagged inline below. Resolve them, or apply the \`claude-review-override\` label to bypass.

<!-- claude-review-marker: head=$HEAD_SHA -->" \
  -F 'comments[]=...'  # one entry per issue
```

Each `comments[]` entry has fields: `path`, `line`, `side` (`RIGHT` for new code), and `body`. The body should:
- Briefly describe the issue
- Quote the offending code
- For a small self-contained fix, include a `suggestion` block ONLY if committing the suggestion fully resolves the issue
- Cite the violated rule (with a permalink to the CLAUDE.md / REVIEW.md line range, full-SHA-pinned: `https://github.com/$REPO/blob/$HEAD_SHA/CLAUDE.md#L<start>-L<end>`)

**One comment per unique `(file, line)` pair.** Do NOT post duplicate inline comments.

If you have "additional unverified findings" from Step 5 (issues 16+), append them as a single bullet list at the END of the review body (not inline), prefixed with `> Additional unverified findings (not validated):`.

**Set the blocking-issues marker:**
```bash
echo "true" > /tmp/claude-blocking-issues
```

## Step 9 — Final marker check

The file `/tmp/claude-blocking-issues` must contain exactly `true` or `false`. The workflow's next step reads it to decide whether to fail the merge check.

If you reached this point without writing the marker (e.g. you bailed early), it should already be `false`.

## Reference: forbidden flags (false positives — never post)

- Pre-existing issues outside the PR diff
- Style or formatting concerns
- "Could use a better variable name"
- "Consider adding a test"
- "Could be more performant"
- Issues that depend on specific inputs/state without evidence
- Anything `pnpm lint`, `tsc`, or `eslint` would catch (CI handles those)
- Rules in CLAUDE.md / REVIEW.md that are explicitly silenced in code (e.g. via `// eslint-disable`, `@ts-ignore`, or an explicit "this section excluded" note)

## Reference: permalink format

Always use SHA-pinned permalinks in inline comment bodies:
```
https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>
```
- The full SHA is in `$HEAD_SHA` (40 hex chars). Do NOT use shortened SHAs.
- Provide ≥1 line of context above and below the cited line.
