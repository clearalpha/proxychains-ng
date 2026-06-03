---
allowed-tools: Read,Glob,Grep,Task,mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh api:*)
description: Review a pull request
---

Perform a code review of this pull request using specialist subagents.

## Available tools

You may ONLY use these tools: `Bash` (for `gh` CLI commands), `Read`, `Glob`, `Grep`, `Task` (to dispatch the specialist reviewer subagents), and the `mcp__github_inline_comment__create_inline_comment` MCP tool. Do NOT attempt to use `WebFetch`, `WebSearch`, or any other network tools — they are not available in this environment and will waste turns.

## jq syntax rules

When writing `--jq` expressions for `gh api` commands, you MUST follow these rules:

1. **Copy the example commands from this document verbatim.** Do not invent new jq expressions. If you need data in a different shape, fetch it with a provided command and reshape the output in your context window.
2. **String slicing** uses `[:N]` not `[0:N]` — e.g., `.body[:150]` not `.body[0:150]`
3. **Always close brackets and parens** — every `[` needs `]`, every `(` needs `)`, every `{` needs `}`
4. **Never embed shell variables in jq strings** — use `jq --arg name "$VAR"` and reference as `$name` inside the expression
5. **jq pipes** (`|`) flow left-to-right: `select()` filters (keeps/drops items), it does not transform; use `.field // ""` for defaults
6. **String interpolation** uses `"\(.field)"` inside double-quoted strings — e.g., `"\(.filename)|\(.status)"`
7. **If a `gh api --jq` command fails**, do NOT retry with a modified jq expression. Instead, run the `gh api` command without `--jq` and process the raw JSON in your context.

## Fast-path: use pre-check metadata when available

The CI workflow's pre-check job may provide metadata in the prompt arguments under the heading "Pre-check metadata (from CI — do NOT re-fetch)". When this metadata is present:

1. **Use it directly** — do NOT re-fetch file lists, per-file metadata, or the HEAD SHA from the GitHub API. The `FILES_IDENTITY` field contains one changed file per line in the format `<filename>|<status>|<additions>|<deletions>`. The `FILE_PATHS` field lists all changed file paths. The `FILES_FINGERPRINT`, `CHANGED_FILES`, and `TOTAL_LINES` fields provide the same aggregate data that API calls would return.
2. **Use `PR_SIZE`** — skip the PR size classification computation. The value is `small`, `medium`, or `large`.
3. **If `REVIEW_SCOPE` is present** — skip the entire "Pre-review: check if review is needed" section. The pre-check already decided whether this should be a `full` or `incremental` review and computed the exact target file set in `REVIEW_TARGET_PATHS` and `REVIEW_TARGET_FILES_IDENTITY`.
4. **If `REVIEW_SCOPE` is `incremental`** — review only the files listed in `REVIEW_TARGET_PATHS`. Prefix the final review comment with the incremental banner using `PREVIOUS_HEAD_SHA` and `PREVIOUS_REVIEWED_AT`.
5. **If `REVIEW_SCOPE` is `incremental` and `REVIEW_TARGET_PATHS` is empty** — post a brief skip comment explaining that there are no newly changed files to review since the previous review, append the fingerprint marker, and stop.
6. **You still need to fetch existing inline comments** (step 1 of "Pre-review: gather context") to avoid posting duplicate findings. This is the one API call that cannot be skipped.

**Turn budget with fast-path:** 1 turn for context (existing inline comments only), remaining turns for direct review and posting.

## Scope constraint

Your primary review scope is code that was **added or modified** in this PR. Do NOT flag style, quality, performance, or documentation issues on unchanged lines.

- **DO**: Flag issues in lines that appear as additions (+) in the diff
- **DO**: Flag issues where a modification introduces a bug in interaction with existing code
- **DO NOT**: Flag MEDIUM/LOW issues in unchanged lines, even if adjacent to changed lines
- **DO NOT**: Suggest improvements to code that was not touched by this PR

**Exception — critical pre-existing issues**: If while reading context you discover a HIGH or CRITICAL severity issue in pre-existing code (security vulnerabilities, data loss risks, broken access control), you SHOULD report it. Place these in a clearly separated section titled "Pre-existing issues (not introduced by this PR)" in the summary comment — never as inline comments on unchanged lines.

Pass this scope constraint verbatim to every subagent you dispatch.

## Turn budget

You have a limited number of turns. Budget them as follows:
- **Context gathering**: 2-3 turns (file list, existing comments, fingerprint check)
- **Subagent dispatch**: 1 turn (they run in parallel)
- **Post-processing and posting**: 2-3 turns

Do NOT spend turns re-reading files, fetching the full diff, or making multiple attempts to post the same comment. If running low on turns, prioritize posting what you have over gathering more findings.

**With pre-check metadata**: Budget shifts to 1 turn for context (existing inline comments only), with remaining turns for direct review and posting. Do not spend turns fetching data already provided in the prompt.

## Pre-review: gather context

The repository is already checked out locally. Use `gh` CLI commands to gather PR context.

> **Fast-path**: If `FILES_IDENTITY` is present in the pre-check metadata, skip steps 2 and 3 below — that data is already available. Only run step 1 to fetch existing inline comments.

1. `gh api repos/<REPO>/pulls/<PR_NUMBER>/comments --paginate --jq '.[] | "\(.path // "general"):\(.line // .original_line // "N/A") \(.body | split("\n")[0][:150])"'` — fetch existing inline review comments (truncated bodies to stay under 256KB limit)
2. `gh pr diff <PR_NUMBER> --name-only` — list changed file paths
3. `gh api repos/<REPO>/pulls/<PR_NUMBER>/files --paginate --jq '.[] | {filename, status, additions, deletions}'` — per-file change metadata (type and size of change, without patch content)

**Do NOT fetch the full diff** with bare `gh pr diff <PR_NUMBER>` (no flags). Large PRs produce output exceeding Claude Code's 256KB read limit, causing silent data loss and wasted turns.

Run each command as a single, standalone `gh` invocation. Do NOT use pipes (`|`), redirects (`>`), or compound operators (`&&`, `;`). These will cause permission errors. The `|` inside `--jq` expressions is a jq filter, not a shell pipe — this is safe. Process the raw output in your context directly.

Compile a list of topics already covered by existing comments. Pass this list to each subagent so they can skip already-addressed findings.

## Pre-review: check if review is needed

Before dispatching subagents, determine whether a review is necessary by checking for a previous review fingerprint.

> **Fast-path**: If `REVIEW_SCOPE` is present in the pre-check metadata, skip this entire section — the pre-check already decided whether the run should be full or incremental and produced the exact review target set.

### Step 1: Fetch current PR state

1. Get the current HEAD SHA: `gh api repos/<REPO>/pulls/<PR_NUMBER> --jq '.head.sha'`
2. You already have the per-file change metadata from step 3 of "Pre-review: gather context". From that data, compute:
   - `files_fingerprint` = `<file_count>|<total_additions>|<total_deletions>` where `file_count` is the number of changed files, `total_additions` is the sum of all additions, and `total_deletions` is the sum of all deletions. Example: `34|3520|6`. Do NOT attempt to compute a SHA-256 hash — you do not have access to `sha256sum`.
   - `files_list` = each file as `<filename>:<additions>:<deletions>:<status>`

### Step 2: Search for previous review marker

Search PR comments for the fingerprint marker left by a previous review:

`gh api repos/<REPO>/issues/<PR_NUMBER>/comments --paginate --jq '.[] | select(.user.type == "Bot") | select(.body | contains("<!-- claude-review-fingerprint")) | {id, body, created_at}'`

**Important:** Always filter by `.user.type == "Bot"` to prevent human users from spoofing fingerprint markers.

If found, parse the most recent marker to extract `head_sha`, `files_fingerprint`, `reviewed_at`, and the per-file `files` list.

### Step 3: Check for human re-review requests

If a previous marker exists, check whether any human (non-bot) commenter posted after the marker's `reviewed_at` timestamp with re-review keywords. Fetch comments after the last review:

`gh api repos/<REPO>/issues/<PR_NUMBER>/comments --paginate --jq '[.[] | select(.created_at > "<reviewed_at>") | select(.user.type != "Bot") | select(.body | test("re-review|review again|please review|@clearalpha-loop"; "i"))] | length'`

If the count is greater than 0, treat this as a human re-review request.

### Step 4: Decision matrix

| Condition | Action |
|-----------|--------|
| No marker found | **Full review** — first time reviewing this PR |
| Human requested re-review | **Full review** — honor the explicit request |
| Same `head_sha` AND same `files_fingerprint` | **Skip** — post a brief comment: "No changes since last review at `<short_sha>`." and stop |
| Different `head_sha` but same `files_fingerprint` | **Skip** — the diff is unchanged (e.g., merge-main). Post: "No meaningful changes since last review (merge commit only)." and stop |
| Different `files_fingerprint` | **Compare the per-file `files:` list** from the previous marker against the current per-file metadata (see below) |

When `files_fingerprint` differs, compare the previous marker's `files:` entries against the current per-file metadata from the API. For each file, compare `filename:additions:deletions:status`. A file is "changed" if it appears in the current list but not the previous, or if its additions/deletions/status differ. Count the changed files and the total line difference. Then:

| Sub-condition | Action |
|---------------|--------|
| >75% of files changed or >75% of lines changed | **Full review** |
| Only some files changed | **Incremental review** — dispatch subagents for changed files only |

For **incremental reviews**: only pass the new/modified files to each subagent. Prefix the review comment with:

> **Incremental Review** — reviewed changes since commit `<previous_short_sha>` (previous review on `<reviewed_at_date>`). Focused on N modified files.

For **skip** decisions: post the skip message as a PR comment and stop. Do not dispatch subagents or post any other output.

## PR size classification

Before dispatching subagents, classify the PR by total changed lines (sum of all additions + deletions across all changed files):

| Size | Threshold | Strategy |
|------|-----------|----------|
| **Small** | ≤200 lines changed AND ≤3 files | **No subagents** — review directly as the orchestrator |
| **Medium** | ≤800 lines AND ≤15 files (but not Small) | **2 subagents** — code-quality-reviewer + security-code-reviewer only |
| **Large** | >800 lines OR >15 files | **All 5 subagents** |

If either threshold is exceeded, use the larger classification (e.g., 1000 lines with 5 files = Large).

**Small PR review**: Read each changed file, examine only the changed lines, and apply combined review criteria (quality, security, performance, test coverage, documentation) in a single pass. Keep your review proportional — a 50-line fix does not need a 2000-word review.

**Medium PR review**: Dispatch only code-quality-reviewer and security-code-reviewer. Handle performance, test-coverage, and documentation concerns yourself during post-processing, but only flag obvious issues.

## Dispatch subagents

**Only dispatch subagents for Medium and Large PRs.** For Small PRs, skip this section entirely and proceed to "Post-review: filter and post".

Use `REVIEW_TARGET_PATHS` and `REVIEW_TARGET_FILES_IDENTITY` when they are present. Otherwise, use the full changed-file set.

From the selected review target set, categorize changed files into these groups:
- **Production source** — non-test, non-migration source files
- **Test files** — files matching `**/tests/**`, `**/test_*`, `**/*_test.*`
- **Migrations** — files matching `**/migration/**`, `**/migrations/**`

Launch only the subagents whose file scope is non-empty. Do NOT launch a reviewer with an empty file set.

For **Medium** PRs, use only:
- **code-quality-reviewer** — production source files only
- **security-code-reviewer** — production source files only (API, auth, service, repository layers)

For **Large** PRs, consider all of these, but skip any reviewer with no relevant files:
- **code-quality-reviewer** — production source files only
- **performance-reviewer** — production source files only (service, repository, API layers)
- **test-coverage-reviewer** — test files AND the production source files they cover
- **documentation-accuracy-reviewer** — production source files with docstrings/docs, plus any markdown files
- **security-code-reviewer** — production source files only (API, auth, service, repository layers)

If every specialist scope is empty after categorization, review directly as the orchestrator instead of launching empty subagents.

Pass each subagent:
- The list of changed file paths relevant to its scope
- Per-file change metadata (status: added/modified/removed, lines added/deleted)
- The list of already-resolved topics
- Instruction to use `Read` to examine individual source files

## Post-review: filter and post

**CRITICAL: Batch all comments before posting.** Do NOT post inline comments one at a time as you process them. Instead: collect ALL findings into a single list, apply ALL filters below, then post the surviving findings.

Once all subagents finish (or you complete your direct review for small PRs), apply these filters IN ORDER:

1. **Deduplication against existing comments**: Drop any finding that matches an already-resolved thread topic
2. **Cross-agent deduplication**: When multiple subagents report the same underlying issue (e.g., both security and performance reviewers flag the same SQL pattern), keep only the single most detailed version and discard the rest
3. **File:line deduplication**: If two or more findings reference the same file and line number (or overlapping line ranges), keep only the single most specific and actionable one
4. **Existing inline comment check**: Before posting any inline comment, verify it does not duplicate an inline comment already on this PR (from the list fetched in pre-review step 1). Compare by file path and line number — if an existing comment covers the same location, skip it
5. **Inline annotations**: Drop any finding where the source code contains an inline comment (prefixed `# Security:`, `# Defensive:`, or `# WP2:`) that explains the decision
6. **Confidence**: Only keep findings you are confident are genuine issues — not speculative, not "nice-to-have" optimizations, not style preferences
7. **Actionability**: Only keep findings with a concrete, specific fix — not vague recommendations

After filtering, separate surviving findings into two groups by severity:

- **Inline comment group** (HIGH or CRITICAL on changed lines): Post these as inline comments on the specific code locations, or as a single top-level comment for general observations.
- **Summary-only group** (MEDIUM, LOW, or INFO): Do NOT create inline comments for these. Instead, collect them into a single summary section appended to the review comment (see format below).
- **Pre-existing issues** (HIGH or CRITICAL on unchanged code): Do NOT post as inline comments. Include in a separate collapsible section in the summary comment (see format below).

### Inline comment formatting rules

- Do NOT use GitHub suggestion code blocks (` ```suggestion `) — they cause encoding issues with the GitHub API
- Use plain text descriptions of the recommended fix instead
- Keep each inline comment concise: 2-3 sentences maximum (issue + fix)

### Summary format for medium/low findings

If there are any MEDIUM/LOW/INFO findings after filtering, append a collapsible section to your review comment:

```
<details>
<summary>Additional observations (not flagged inline)</summary>

| Severity | File | Finding |
|----------|------|---------|
| MEDIUM | `path/to/file.py:42` | Brief description of the issue and suggested fix |
| LOW | `path/to/other.py:17` | Brief description of the issue and suggested fix |

</details>
```

### Summary format for pre-existing issues

If there are any HIGH/CRITICAL findings on pre-existing (unchanged) code, append a separate collapsible section:

```
<details>
<summary>Pre-existing issues (not introduced by this PR)</summary>

| Severity | File | Finding |
|----------|------|---------|
| CRITICAL | `path/to/file.py:100` | Brief description of the pre-existing vulnerability |
| HIGH | `path/to/other.py:50` | Brief description of the pre-existing issue |

</details>
```

Keep feedback concise. If no noteworthy findings survive filtering at any severity, post a brief approving comment.

### Append review fingerprint marker

After posting the review comment (whether full, incremental, or approving), append the following hidden HTML marker to the **end** of the comment body. This marker is invisible in rendered markdown but enables incremental reviews on subsequent triggers.

```
<!-- claude-review-fingerprint
version: 1
head_sha: <FULL_HEAD_SHA>
reviewed_at: <ISO_8601_UTC_TIMESTAMP>
files_fingerprint: <FILE_COUNT>|<TOTAL_ADDITIONS>|<TOTAL_DELETIONS>
files:
- <filename>:<additions>:<deletions>:<status>
- <filename>:<additions>:<deletions>:<status>
-->
```

Use the pre-check metadata values (`HEAD_SHA`, `FILES_FINGERPRINT`, `FILES_IDENTITY`) when available, rather than re-computing them.

Field definitions:
- `head_sha` — the PR HEAD commit SHA at the time of this review (from pre-check metadata or step 1 of the pre-review check)
- `reviewed_at` — current UTC timestamp in ISO-8601 format (e.g., `2026-02-17T15:30:00Z`)
- `files_fingerprint` — `<file_count>|<total_additions>|<total_deletions>` summary of the PR diff (e.g., `34|3520|6`). Do NOT compute a SHA-256 hash.
- `files` — one entry per changed file: `<filename>:<additions>:<deletions>:<status>` where status is `added`, `modified`, `removed`, `renamed`, etc. Derive this directly from `FILES_IDENTITY` when it is present.

Always include this marker, even on skip comments (use the current fingerprint data). This ensures the next trigger has a marker to compare against.
