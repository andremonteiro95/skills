# PR Review Skills

Two Claude Code skills for AI-powered GitHub PR review. A subagent analyzes each PR's diff, presents severity-rated findings, and walks you through them interactively before submitting a batched GitHub review.

## Skills

### `/pr-inbox` — Review Queue

Discovers all open PRs where you're a requested reviewer (directly or via team), shows a summary table with CI status, and lets you review them one by one.

### `/review-pr <number|url>` — Single PR Review

Reviews a single PR by number or URL. Same interactive walkthrough, just for one PR.

## How It Works

1. **Discovery** — finds PRs using `gh search prs --review-requested=@me` (handles both direct and team-based requests)
2. **Analysis** — dispatches a reviewer subagent per PR that analyzes the diff and returns structured findings rated as Critical / Important / Minor
3. **Summary** — presents a verdict with severity counts and strengths
4. **Walkthrough** — shows each finding with the relevant diff hunk; you accept, reject, or edit each comment
5. **Submission** — batches accepted comments into a single GitHub review with your chosen verdict (approve / request changes / comment)

## Installation

In Claude Code:

```
/plugin marketplace add andremonteiro95/skills
/plugin install pr-reviews@andremonteiro95-skills
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with access to the target repo

## File Structure

```
skills/
  pr-inbox/
    SKILL.md              # Review queue orchestration
  review-pr/
    SKILL.md              # Single PR review flow
    reviewer-prompt.md    # Shared subagent template
```
