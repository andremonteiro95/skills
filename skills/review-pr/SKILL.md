---
name: review-pr
description: Use when reviewing pull requests — single PR by number/URL, or no args to check your full review inbox. Triggers on "review PR 123", "review my PRs", "check my PR inbox", "what PRs need my review"
---

# Review PR

Review GitHub PRs with AI-powered analysis. Supports single PR review by number/URL, or inbox mode to discover and review all PRs awaiting your review.

## Arguments

- `/review-pr 142` — review PR #142
- `/review-pr https://github.com/org/repo/pull/142` — review by URL
- `/review-pr` — inbox mode: discover all PRs awaiting your review

## Step 1: Parse Input & Determine Mode

**Single PR mode** (argument provided):
Extract the PR number. If a URL, parse the number from it.

**Inbox mode** (no argument):
```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh search prs --review-requested=@me --repo=$REPO --state=open \
  --json number,title,author,url
```

If no PRs found: "No PRs awaiting your review in {REPO}." and stop.

For each discovered PR, fetch enrichment data:
```bash
gh pr checks {NUMBER}
gh pr view {NUMBER} --json additions,deletions,isDraft
```

Show summary table:
```
Found {COUNT} PRs awaiting your review in {REPO}:

 #  | PR                                | Author    | Checks     | +/-
----|-----------------------------------|-----------|------------|--------
 1  | #{NUM} {TITLE}                    | {AUTHOR}  | {STATUS}   | +{A} -{D}
```

Checks emoji: ✅ pass, ❌ fail, ⏳ pending, ➖ none. Mark draft PRs with `[DRAFT]`.

Ask: "Review all, or pick specific numbers? (all / 1,3,5 / none)"

If **none**, stop.

For each selected PR, run Steps 2–8. Dispatch reviewer subagents in **parallel** (Step 5), but walk through results sequentially (Steps 6–7).

## Step 2: Fetch PR Context

```bash
# Metadata
gh pr view {NUMBER} --json title,author,body,additions,deletions,isDraft,baseRefName,headRefName,url

# CI status
gh pr checks {NUMBER}

# Full diff
gh pr diff {NUMBER}
```

### Edge Cases

- **PR not found:** If `gh pr view` fails → "PR #{NUMBER} not found in {REPO}." and stop.
- **Own PR (inbox mode):** Skip silently — exclude from the summary table.
- **Own PR (single PR mode):** "This is your own PR." and stop.

## Step 3: Check for Prior Review

```bash
YOUR_LOGIN=$(gh api user --jq '.login')
gh api repos/{OWNER}/{REPO}/pulls/{NUMBER}/reviews \
  --jq "[.[] | select(.user.login == \"$YOUR_LOGIN\")] | last"
```

If no prior review exists, continue to Step 4.

If a prior review exists, extract `submitted_at` and `state`, then count new commits:
```bash
gh api repos/{OWNER}/{REPO}/pulls/{NUMBER}/commits \
  --jq "[.[] | select(.commit.committer.date > \"{SUBMITTED_AT}\")] | length"
```

Prompt:
```
You reviewed this PR on {DATE} ({VERDICT}).
{N} new commits since then.

(F)ull re-review / (I)ncremental (new commits only) / (S)kip?
```

If **Skip**, move to next PR (inbox) or stop (single).

If **Incremental**, fetch the delta diff instead of the full diff:
```bash
# Get SHA of first new commit's parent
FIRST_NEW_SHA=$(gh api repos/{OWNER}/{REPO}/pulls/{NUMBER}/commits \
  --jq "[.[] | select(.commit.committer.date > \"{SUBMITTED_AT}\")] | first | .sha")
PARENT_SHA=$(gh api repos/{OWNER}/{REPO}/commits/$FIRST_NEW_SHA \
  --jq '.parents[0].sha')

# Get HEAD SHA
HEAD_SHA=$(gh pr view {NUMBER} --json headRefOid --jq '.headRefOid')

# Delta diff
gh api repos/{OWNER}/{REPO}/compare/$PARENT_SHA...$HEAD_SHA \
  --jq '.files[] | {filename, patch}'
```

Use this delta diff in place of the full diff for Steps 4–5.

## Step 4: Diff Management

### Filter Generated Files

Before sending the diff to the reviewer, strip files matching:
```
*.lock, *.min.js, *.min.css, *.snap, *.generated.*,
package-lock.json, yarn.lock, pnpm-lock.yaml,
*.pb.go, *_generated.go, *.g.dart
```

Note skipped files: "Skipped {N} generated/lock files ({list})."

### Size Check

Count lines in the filtered diff.

| Diff size | Strategy |
|-----------|----------|
| < 2,000 lines | Send full diff to one subagent |
| 2,000–8,000 lines | Ask: "Large PR ({N} lines, {M} files). Review all at once or split by directory? (a/s)" |
| > 8,000 lines | "PR too large for single pass ({N} lines). Splitting by directory." |

### Chunked Review

When splitting by directory:
1. Group changed files by their top-level directory
2. Dispatch one subagent per directory group (each gets PR description + its chunk)
3. Merge findings from all subagents, deduplicate by file:line (keep higher severity)
4. Present merged findings in the normal walkthrough

## Step 5: Dispatch Reviewer Subagent(s)

Read the contents of `review-lens.md` and `output-contract.md` from this skill's directory.

Assemble the reviewer prompt by concatenating:

1. Contents of `review-lens.md` (verbatim)
2. Contents of `output-contract.md` (verbatim)
3. PR context block:

```
## PR Context

**PR:** #{PR_NUMBER} — {PR_TITLE}
**Author:** {PR_AUTHOR}
**Base:** {BASE_BRANCH} ← {HEAD_BRANCH}
**CI Status:** {CI_STATUS}
{CI_DETAILS}

## PR Description

{PR_BODY}

## Diff

{PR_DIFF}
```

4. If incremental mode, append:

```
## Incremental Review Note

This is an incremental review. The reviewer previously submitted {VERDICT} on {DATE}. Focus on whether new changes address prior feedback and whether they introduce new issues.
```

Dispatch using the Agent tool. The subagent returns a JSON object.

## Step 6: Present Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR #{NUMBER} — {TITLE} ({AUTHOR})
Checks: {CHECK_EMOJI} {CHECK_STATUS} | +{ADDITIONS} -{DELETIONS}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Verdict: {VERDICT_EMOJI} {VERDICT}

Summary: {SUMMARY}

Description: {ALIGNMENT_EMOJI} {ALIGNMENT_STATUS} {ALIGNMENT_NOTES}

  🔴 {CRITICAL_COUNT} Critical    🟡 {IMPORTANT_COUNT} Important    ⚪ {MINOR_COUNT} Minor

Strengths:
  {STRENGTHS_LIST}

Walk through findings? (y/n/skip)
```

Emoji mapping:
- Checks: ✅ Pass, ❌ Fail, ⏳ Pending, ➖ None
- Verdict: 🟢 Approve, 🟡 Request Changes, 🔵 Comment
- Severity: 🔴 Critical, 🟡 Important, ⚪ Minor
- Alignment: ✅ aligned, ⚠️ drift, ➖ missing

If **skip**, go to Step 8 with no comments.
If **n**, go to Step 8 with no comments.
If **y**, proceed to Step 7.

## Step 7: Finding-by-Finding Walkthrough

Present findings one at a time, ordered by severity (Critical → Important → Minor):

```
[{INDEX}/{TOTAL}] {SEVERITY_EMOJI} {SEVERITY} — {TITLE}
File: {FILE}:{LINE_RANGE}

{DIFF_HUNK}

{EXPLANATION}

Suggestion: {SUGGESTION}

Accept / Reject / Edit comment? (a/r/e)
```

- **Accept (a):** Queue the finding. Comment body = explanation + suggestion.
- **Reject (r):** Drop the finding.
- **Edit (e):** Ask user for revised comment text. Queue the edited version.

## Step 8: Submit Review

```
Review summary for PR #{NUMBER}:
  Accepted: {ACCEPTED_BY_SEVERITY}
  Rejected: {REJECTED_COUNT}

Recommended verdict: {RECOMMENDED_VERDICT}
Submit as: (A)pprove / (R)equest Changes / (C)omment only / (S)kip?
```

Verdict adjustment:
- If user rejected all Critical findings → downgrade from "request-changes" to "comment"
- If user accepted any Critical finding → keep "request-changes"

If **(S)kip**, do not submit.

Otherwise submit:
```bash
gh api repos/{OWNER}/{REPO}/pulls/{NUMBER}/reviews \
  --method POST \
  --field event="{EVENT}" \
  --field body="{REVIEW_BODY}" \
  --field comments='[{COMMENTS_JSON}]'
```

Where:
- `{EVENT}` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`
- `{REVIEW_BODY}` is the subagent's summary (plus "Incremental review (commits since {DATE})." if applicable)
- `{COMMENTS_JSON}` is the array of accepted findings: `[{"path": "{FILE}", "line": {LINE}, "body": "{COMMENT}"}]`

After submission: "Review submitted for PR #{NUMBER} ✅"

## Step 9: Session Summary (Inbox Mode Only)

After all selected PRs are processed:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Review session complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 PR                              | Verdict
---------------------------------|------------------
 #142 Fix auth redirect loop     | ✅ Approved
 #139 Add billing webhook handler| 🟡 Request Changes
 #145 Update README typos        | ⏭️  Skipped
```
