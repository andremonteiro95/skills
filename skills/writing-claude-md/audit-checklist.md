# CLAUDE.md Audit Checklist

Automated checks to run when auditing an existing CLAUDE.md. Run sequentially, collect findings, then present a prioritized report.

## Checks

### 1. Missing Essentials
**What:** Verify the file contains a project description and build/test commands.
**How:** Read the file; check for a sentence describing what the project does and commands for building/testing.
**Severity:** error
**Fix:** Add the missing essentials — see Tier 1 in the main skill.

### 2. Stale File Paths
**What:** File paths referenced in CLAUDE.md may no longer exist.
**How:** Extract all file/directory paths mentioned. Verify each exists in the repo with Glob or ls.
**Severity:** error
**Fix:** Remove the path reference. Describe what the module handles, not where it lives — or remove entirely and let the agent discover.

### 3. Dead References
**What:** Links to docs files (e.g., "see docs/TESTING.md") that don't exist.
**How:** Extract all internal file references. Verify each target exists.
**Severity:** error
**Fix:** Create the referenced file or remove the reference.

### 4. Contradictions
**What:** Rules that conflict with each other.
**How:** Read all instructions and identify pairs that give opposing guidance (e.g., "always use const" vs. "use let when value changes").
**Severity:** warning
**Fix:** Present contradictions to the user. Ask which version to keep. Remove the other.

### 5. Linter-Replaceable Rules
**What:** Style rules that should be enforced by linters/formatters, not prose.
**How:** Flag instructions about code style (const vs let, semicolons, quotes, indentation, import order, naming conventions).
**Severity:** warning
**Fix:** Move to ESLint/Prettier/Biome config. Delete from CLAUDE.md.

### 6. Personal vs. Project
**What:** Personal preferences that belong in `~/.claude/CLAUDE.md`, not the project root.
**How:** Flag instructions about commit message style, verbosity preferences, response formatting, or individual workflow habits.
**Severity:** warning
**Fix:** Move to `~/.claude/CLAUDE.md`. Remove from project CLAUDE.md.

### 7. Bloat Assessment
**What:** File is too large without progressive disclosure structure.
**How:** Count lines and words. Flag if >100 lines or >800 words without references to separate files.
**Severity:** warning
**Fix:** Apply the Tiered Framework from the main skill. Split Tier 2/3 content into separate files.

### 8. Progressive Disclosure Opportunities
**What:** Domain-specific blocks that could be separate files.
**How:** Identify contiguous blocks of >30 lines focused on one domain (TypeScript, testing, API design, etc.).
**Severity:** info
**Fix:** Move to a separate file. Add a reference from root.

### 9. Staleness Signals
**What:** Content that may be outdated.
**How:** Look for mentions of deprecated dependencies, old API patterns, TODO/TBD markers, version-specific instructions for old versions.
**Severity:** info
**Fix:** Verify currency. Update or remove stale content.

### 10. Obvious/Redundant Instructions
**What:** Instructions that provide no value because any competent agent already knows them.
**How:** Flag guidance like "write clean code", "use meaningful names", "follow best practices", "handle errors properly".
**Severity:** info
**Fix:** Delete. These waste tokens without changing agent behavior.

## Output Format

After running all checks, present findings as:

```markdown
## CLAUDE.md Audit Report

### Errors (must fix)
- [findings with specific line references]

### Warnings (should fix)
- [findings with specific line references]

### Info (consider)
- [findings]

### Health Score
X/10 checks passed — [brief assessment]
```

Work through fixes conversationally, starting with errors. Confirm each change with the user before making it.
