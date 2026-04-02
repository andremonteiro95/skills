# PR Review Skills

A Claude Code skill for AI-powered GitHub PR review. A subagent analyzes each PR's diff, presents severity-rated findings, and walks you through them interactively before submitting a batched GitHub review.

## Skills

### `/review-pr` — PR Review (single PR + inbox mode)

- **`/review-pr <number|url>`** — Reviews a specific PR by number or URL.
- **`/review-pr`** (no args) — Inbox mode: discovers all open PRs where you're a requested reviewer (directly or via team), shows a summary table with CI status, and lets you review them one by one.

## How It Works

1. **Discovery** — finds PRs using `gh search prs --review-requested=@me` (handles both direct and team-based requests)
2. **Diff management** — filters out generated files and lock files before analysis; large PRs are reviewed in chunks to stay within context limits
3. **Analysis** — dispatches a reviewer subagent per PR that analyzes the diff and returns structured findings rated as Critical / Important / Minor
4. **Description alignment** — checks whether the implementation matches what the PR description says it does, flagging any gaps or scope drift
5. **Summary** — presents a verdict with severity counts and strengths
6. **Walkthrough** — shows each finding with the relevant diff hunk; you accept, reject, or edit each comment
7. **Incremental re-review** — if a prior review is detected, offers to review only the commits added since then instead of the full diff
8. **Submission** — batches accepted comments into a single GitHub review with your chosen verdict (approve / request changes / comment)

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
  review-pr/
    SKILL.md              # Unified skill (single PR + inbox mode)
    review-lens.md        # Reviewer persona, focus areas, severity definitions
    output-contract.md    # JSON schema, verdict rules
```
