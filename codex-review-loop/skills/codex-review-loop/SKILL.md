---
name: codex-review-loop
description: This skill should be used when the user asks to "review with codex", "codex review loop", "iterative review", "review until approved", "codex審查", "迭代審查", "loop until no issues", or when completing a spec, plan, or code that needs quality review via Codex CLI. Implements a write-review-fix loop where Claude Code produces content and Codex CLI reviews iteratively until approved.
---

# Codex Review Loop

Iterative review workflow: write → Codex review → fix → re-review → repeat until APPROVED.

## Review Types

| Type | Command | When |
|------|---------|------|
| Document (spec/plan) | `codex exec "{prompt}"` | After writing/updating `.md` documents |
| Code (uncommitted) | `codex review --uncommitted` | After code changes (requires git) |
| Code (specific file) | `codex exec "Review {path}..."` | Review any file without git |

## Core Loop

### Step 1: Determine Type and Build Prompt

Select the appropriate review type. For document reviews, load a prompt template from `references/prompt-templates.md` and fill in the file path, criteria, and any prior-round context.

### Step 2: Execute Review

Run Codex with 180s timeout:
```bash
codex exec "{filled_prompt}" 2>&1 | tail -60
```

Parse output for **APPROVED** or listed issues (Critical/Important/Suggestion).

### Step 3: Branch on Result

- **APPROVED** → report results, commit if in git, done.
- **Issues found** → fix each issue, commit if in git, return to Step 2 with updated context.

### Step 4: Update Context for Next Round

Include in the next prompt:
- Round number
- What was fixed
- Instruction to verify fixes AND find new issues

This prevents re-reporting already-fixed issues.

## Safeguards

- **Max 10 rounds** — stop and surface to user if not approved after 10 iterations.
- **Bio safety filter** — if Codex blocks content, retry with rephrased prompt. If still blocked, fall back to `codex review --uncommitted` or inform user to use OpenAI Chat.
- **Non-git context** — skip `codex review --uncommitted` and commit steps. Use `codex exec "Review {path}..."` for all review types.

## Prompt Templates and Criteria

Load detailed prompt templates and review criteria from:
- **`references/prompt-templates.md`** — ready-to-use templates for spec, plan, code, and alignment reviews

## Integration

After approval, commit changes (if git available), push if requested, and notify user with:
- Total rounds
- Issues found/fixed per round
- Final status: APPROVED
