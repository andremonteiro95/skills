# Agent Skills

Developer workflow skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — PR reviews and CLAUDE.md authoring, all from your terminal.

## Installation

```bash
/plugin marketplace add andremonteiro95/skills
/plugin install dev-skills@andremonteiro95-skills
```

Skills update automatically when you update the plugin:

```bash
/plugin update dev-skills
```

## Code Review

- **review-pr** — AI-powered PR review that dispatches a reviewer subagent, rates findings by severity, and walks you through each one before submitting a batched GitHub review.

  ```
  /review-pr 142
  /review-pr https://github.com/org/repo/pull/142
  /review-pr                                        # inbox mode
  ```

  Requires [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with access to the target repo.

  **How it works**

  1. **Discovery** — finds PRs using `gh search prs --review-requested=@me` (handles both direct and team-based requests)
  2. **Diff management** — strips generated files and lockfiles; large PRs (2k+ lines) are chunked by directory so the reviewer stays within context limits
  3. **Analysis** — a reviewer subagent analyzes the diff and returns structured findings rated Critical / Important / Minor
  4. **Description alignment** — flags when the diff doesn't match what the PR description claims, or vice versa
  5. **Walkthrough** — presents each finding with the relevant diff hunk; you accept, reject, or edit before anything is submitted. Clean PRs skip straight to submission.
  6. **Suggestion blocks** — when the reviewer can provide exact replacement code, accepted comments include [`suggestion` blocks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/incorporating-feedback-in-your-pull-request) that authors can apply with one click
  7. **Incremental re-review** — detects prior reviews and offers to review only new commits instead of the full diff
  8. **Submission** — batches accepted comments into a single GitHub review with your chosen verdict

## Writing & Knowledge

- **writing-claude-md** — Create, refactor, or audit CLAUDE.md files. Treats the instruction file as a launchpad — every instruction must earn its place by applying to most tasks in the repo.

  ```
  /writing-claude-md
  ```

  **When to use**

  - New project needs a CLAUDE.md
  - Existing CLAUDE.md is bloated, stale, or agents ignore it
  - Evaluate or health-check instruction file quality

## What's Inside

```
skills/
  review-pr/
    SKILL.md            # PR review — single PR and inbox mode
    review-lens.md      # What the reviewer looks for (persona, severity, focus areas)
    output-contract.md  # JSON schema the reviewer must return
  writing-claude-md/
    SKILL.md            # CLAUDE.md authoring and auditing
    audit-checklist.md  # Quality checklist for instruction files
```

## Contributing

Skills live directly in this repository. To contribute:

1. Fork the repository
2. Create a branch for your skill
3. Submit a PR

## License

MIT
