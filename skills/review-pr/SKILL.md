---
name: review-pr
description: Use when reviewing a single pull request by number or URL, or when the user says "review PR 123" or "review this PR"
---

# Review PR

Review a single GitHub PR with AI-powered analysis. Presents a severity-rated summary, walks you through findings one by one, and submits a batched GitHub review with your accepted comments.

## Arguments

This skill accepts a PR number or GitHub PR URL as an argument:
- `/review-pr 142`
- `/review-pr https://github.com/org/repo/pull/142`

If no argument is provided, ask: "Which PR? (number or URL)"

## Step 1: Parse Input and Determine Repo

Extract the PR number from the argument. If a URL is provided, parse the number from it.

Get the repo context:
```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

## Step 2: Fetch PR Context

Run these commands to gather all context for the reviewer:

```bash
# PR metadata
gh pr view {NUMBER} --json title,author,body,additions,deletions,isDraft,baseRefName,headRefName,url

# CI check status
gh pr checks {NUMBER}

# Full diff
gh pr diff {NUMBER}
```

### Edge Cases

Before proceeding, check:

- **PR not found:** If `gh pr view` fails, say "PR #{NUMBER} not found in {REPO}" and stop.
- **Own PR:** If the PR author matches `gh api user --jq '.login'`, warn: "This is your own PR — review anyway? (y/n)". Stop if they say no.
- **Already reviewed:** Run `gh pr view {NUMBER} --json reviews --jq '.reviews[] | select(.author.login == "{YOUR_LOGIN}")'`. If you find an existing review, show its status and ask: "You've already reviewed this PR ({STATUS}). Re-review? (y/n)". Stop if they say no.

## Step 3: Dispatch Reviewer Subagent

Read the template at `review-pr/reviewer-prompt.md` and fill in the placeholders:

| Placeholder | Source |
|-------------|--------|
| `{PR_NUMBER}` | From argument |
| `{PR_TITLE}` | From `gh pr view --json title` |
| `{PR_AUTHOR}` | From `gh pr view --json author` |
| `{BASE_BRANCH}` | From `gh pr view --json baseRefName` |
| `{HEAD_BRANCH}` | From `gh pr view --json headRefName` |
| `{CI_STATUS}` | Summary from `gh pr checks`: "Pass", "Fail", or "Pending" |
| `{CI_DETAILS}` | If failing, list which checks failed |
| `{PR_BODY}` | From `gh pr view --json body` |
| `{PR_DIFF}` | From `gh pr diff` |

Dispatch using the Agent tool. The subagent returns a JSON object with verdict, summary, strengths, and findings.

## Step 4: Present Summary

Show the PR summary to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR #{NUMBER} — {TITLE} ({AUTHOR})
Checks: {CHECK_EMOJI} {CHECK_STATUS} | +{ADDITIONS} -{DELETIONS}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Verdict: {VERDICT_EMOJI} {VERDICT}

Summary: {SUMMARY}

  {CRITICAL_EMOJI} {CRITICAL_COUNT} Critical    {IMPORTANT_EMOJI} {IMPORTANT_COUNT} Important    {MINOR_EMOJI} {MINOR_COUNT} Minor

Strengths:
  {STRENGTHS_LIST}

Walk through findings? (y/n/skip)
```

Emoji mapping:
- Checks: ✅ Pass, ❌ Fail, ⏳ Pending
- Verdict: 🟢 Approve, 🟡 Request Changes, 🔵 Comment
- Severity counts: 🔴 Critical, 🟡 Important, ⚪ Minor

If the user says **n** or **skip**, go to Step 6 (submit with no comments, or skip entirely).
If the user says **y**, proceed to Step 5.

## Step 5: Finding-by-Finding Walkthrough

Present findings one at a time, ordered by severity (Critical first, then Important, then Minor):

```
[{INDEX}/{TOTAL}] {SEVERITY_EMOJI} {SEVERITY} — {TITLE}
File: {FILE}:{LINE_RANGE}

{DIFF_HUNK}

{EXPLANATION}

Suggestion: {SUGGESTION}

Accept / Reject / Edit comment? (a/r/e)
```

For each finding:
- **Accept (a):** Queue the finding for the review. The comment body is: the explanation + suggestion.
- **Reject (r):** Drop the finding. Move to next.
- **Edit (e):** Ask the user to provide their revised comment text. Queue the edited version.

Track accepted findings in a list for submission.

## Step 6: Submit Review

After the walkthrough (or if the user skipped it), show the review summary:

```
Review summary for PR #{NUMBER}:
  Accepted: {ACCEPTED_BY_SEVERITY}
  Rejected: {REJECTED_COUNT}

Recommended verdict: {RECOMMENDED_VERDICT}
Submit as: (A)pprove / (R)equest Changes / (C)omment only / (S)kip?
```

The recommended verdict comes from the subagent's verdict, adjusted:
- If the user rejected all Critical findings, downgrade from "request-changes" to "comment"
- If the user accepted any Critical finding, keep "request-changes"

If the user chooses **(S)kip**, do not submit. Move on.

Otherwise, submit using the GitHub API:

```bash
gh api repos/{OWNER}/{REPO}/pulls/{NUMBER}/reviews \
  --method POST \
  --field event="{EVENT}" \
  --field body="{REVIEW_BODY}" \
  --field comments='[{COMMENTS_JSON}]'
```

Where:
- `{EVENT}` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`
- `{REVIEW_BODY}` is the subagent's summary
- `{COMMENTS_JSON}` is the array of accepted findings mapped to `{"path": "{FILE}", "line": {LINE}, "body": "{COMMENT}"}`

After submission, confirm: "Review submitted for PR #{NUMBER} ✅"
