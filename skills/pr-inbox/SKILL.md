---
name: pr-inbox
description: Use when reviewing multiple PRs, checking your review queue, or when the user says "review my PRs", "check my PR inbox", or "what PRs need my review"
---

# PR Inbox

Discover all open PRs where you're a requested reviewer (directly or via team), review them with AI-powered analysis, and submit GitHub reviews interactively.

## Step 1: Discover PRs

Get the current repo and find PRs awaiting your review:

```bash
# Get repo
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')

# Find PRs where you're a requested reviewer (handles both direct and team-based)
gh search prs --review-requested=@me --repo=$REPO --state=open \
  --json number,title,author,url
```

If no PRs are found, say: "No PRs awaiting your review in {REPO}." and stop.

## Step 2: Enrich PR Data

For each discovered PR, fetch additional context:

```bash
# CI checks
gh pr checks {NUMBER}

# Size and draft status
gh pr view {NUMBER} --json additions,deletions,isDraft
```

## Step 3: Show Summary Table

Present the PRs in a table:

```
Found {COUNT} PRs awaiting your review in {REPO}:

 #  | PR                                | Author    | Checks     | +/-
----|-----------------------------------|-----------|------------|--------
 1  | #{NUM} {TITLE}                    | {AUTHOR}  | {STATUS}   | +{A} -{D}
 2  | #{NUM} {TITLE}                    | {AUTHOR}  | {STATUS}   | +{A} -{D}
```

Emoji mapping for checks:
- ✅ All checks pass
- ❌ Any check failing
- ⏳ Checks pending or running
- ➖ No checks configured

Mark draft PRs with `[DRAFT]` after the title.

Then ask: "Review all, or pick specific numbers? (all / 1,3,5 / none)"

If the user says **none**, stop.

## Step 4: Dispatch Reviewer Subagents

For each selected PR, dispatch a reviewer subagent **in parallel** using the Agent tool.

For each subagent:
1. Fetch the PR's full context (diff, body, checks) using the commands from review-pr Step 2
2. Read the template at `review-pr/reviewer-prompt.md`
3. Fill in all placeholders
4. Dispatch with the Agent tool

Wait for all subagents to complete.

## Step 5: Walk Through Each PR

For each PR (in the order they appeared in the table), run the interactive walkthrough from the `review-pr` skill:

1. **Present summary** (review-pr Step 4)
2. **Finding-by-finding drill-down** if requested (review-pr Step 5)
3. **Submit review** (review-pr Step 6)

After each PR, if there are more PRs remaining, say:
"Moving to next PR ({REMAINING} remaining)..."

## Step 6: Session Summary

After all PRs are processed, show a summary:

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
