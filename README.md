# PR Reviews

AI-powered code review for GitHub pull requests, built as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill.

Point it at a PR and it dispatches a reviewer subagent that analyzes the diff, rates findings by severity, and walks you through each one before submitting a batched GitHub review — all without leaving your terminal.

## Installation

```
/plugin marketplace add andremonteiro95/skills
/plugin install pr-reviews@andremonteiro95-skills
```

Requires [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with access to the target repo.

## Usage

- **`/review-pr 142`** — review a single PR by number
- **`/review-pr https://github.com/org/repo/pull/142`** — review by URL
- **`/review-pr`** — inbox mode: discover all PRs awaiting your review, pick which to review

## How It Works

```
Discover PRs ─► Fetch diff ─► Filter noise ─► Dispatch reviewer ─► Present summary
                                                                         │
                                          Submit to GitHub ◄─ Walkthrough ┘
```

1. **Discovery** — finds PRs using `gh search prs --review-requested=@me` (handles both direct and team-based requests)
2. **Diff management** — strips generated files and lockfiles; large PRs (2k+ lines) are chunked by directory so the reviewer stays within context limits
3. **Analysis** — a reviewer subagent analyzes the diff and returns structured findings rated Critical / Important / Minor
4. **Description alignment** — flags when the diff doesn't match what the PR description claims, or vice versa
5. **Walkthrough** — presents each finding with the relevant diff hunk; you accept, reject, or edit each comment before anything is submitted. Clean PRs (0 findings) skip straight to submission.
6. **GitHub suggestion blocks** — when the reviewer can provide exact replacement code, accepted comments include [`suggestion` blocks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/incorporating-feedback-in-your-pull-request) that authors can apply with one click
7. **Incremental re-review** — detects prior reviews and offers to review only new commits instead of the full diff
8. **Submission** — batches accepted comments into a single GitHub review with your chosen verdict

## What's Inside

```
skills/
  review-pr/
    SKILL.md            # The skill — handles single PR and inbox mode
    review-lens.md      # What the reviewer looks for (persona, severity, focus areas)
    output-contract.md  # JSON schema the reviewer must return
```

The reviewer prompt is assembled from two independent files: a **review lens** (what to look for) and an **output contract** (how to format findings). This separation means you can tune what the reviewer focuses on without touching the output format, and vice versa.

## License

MIT
