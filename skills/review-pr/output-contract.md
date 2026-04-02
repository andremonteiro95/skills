# Output Contract

You MUST respond with ONLY a JSON object matching this exact structure. No prose before or after.

## Schema

```json
{
  "verdict": "approve | request-changes | comment",
  "summary": "2-3 sentence assessment of the PR",
  "description_alignment": {
    "status": "aligned | drift | missing",
    "notes": "Optional explanation of what doesn't match"
  },
  "strengths": [
    "Specific strength with file:line reference where applicable"
  ],
  "findings": [
    {
      "severity": "critical | important | minor",
      "file": "exact/path/to/file.ext",
      "line": 42,
      "line_range": "42-58",
      "title": "Short descriptive title",
      "explanation": "Why this matters — what could go wrong",
      "suggestion": "Concrete fix or approach to resolve this",
      "diff_hunk": "The relevant lines from the diff for context"
    }
  ]
}
```

## Line Number Semantics

The `line` field must be the **line number within the diff hunk**, not the absolute file line number. GitHub's review comment API positions comments relative to the diff. If you use absolute line numbers, comments will appear on the wrong line or fail silently. Only reference lines that appear in the diff — never guess at lines outside it.

## Verdict Rules

- If ANY finding is Critical → verdict MUST be "request-changes"
- If findings are only Important and/or Minor → verdict should be "request-changes" or "comment" (use judgment)
- If no findings or only Minor → verdict can be "approve"
- An empty findings array with verdict "approve" is valid for clean PRs
