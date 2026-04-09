---
name: codex-review-loop
description: This skill should be used when the user asks to "review with codex", "codex review loop", "iterative review", "review until approved", "codex審查", "迭代審查", "loop until no issues", or when completing a spec, plan, or code that needs quality review via Codex CLI. Implements a write-review-fix loop where Claude Code produces content and Codex CLI reviews iteratively until approved.
---

# Codex Review Loop

Iterative review workflow: write → Codex review → fix → re-review → repeat until APPROVED.

## Prerequisites

This skill requires the **codex plugin** (`openai/codex-plugin-cc`). Run `/codex:setup` to verify readiness.

The codex-companion runtime is at:
```
CODEX_COMPANION="${CLAUDE_PLUGIN_ROOT:-/home/joe/.claude/plugins/cache/openai-codex/codex/1.0.3}/scripts/codex-companion.mjs"
```

## Review Types

| Type | Command | When |
|------|---------|------|
| Code (uncommitted) | `node "$CODEX_COMPANION" review --wait` | After code changes (default) |
| Code (adversarial) | `node "$CODEX_COMPANION" adversarial-review --wait [focus]` | Deep review with specific focus |
| Code (branch) | `node "$CODEX_COMPANION" review --wait --scope branch` | Review all commits on branch |
| Document / Custom | `node "$CODEX_COMPANION" task "{prompt}"` | Spec, plan, or custom review |

### Fallback (no plugin)

If the codex plugin is not installed, fall back to direct CLI:
```bash
codex exec "{filled_prompt}" 2>&1 | tail -60        # document review
codex review --uncommitted 2>&1 | tail -60           # code review
```

## Core Loop

### Step 1: Determine Type and Select Command

Choose the review command based on what needs review:

- **Uncommitted code changes** → `review --wait`
- **Uncommitted with specific concern** (e.g., data leakage, security) → `adversarial-review --wait "focus text"`
- **Full branch diff** → `review --wait --scope branch`
- **Document / spec / plan** → `task` with a prompt template from `references/prompt-templates.md`

### Step 2: Execute Review

Run the codex-companion command with `--wait` for synchronous results:

```bash
# Code review (standard)
node "$CODEX_COMPANION" review --wait 2>&1 | tail -80

# Adversarial review with focus
node "$CODEX_COMPANION" adversarial-review --wait "Check for data leakage in walk-forward validation" 2>&1 | tail -80

# Custom task review (document / spec)
node "$CODEX_COMPANION" task "Review {file_path} for: 1) Mathematical consistency 2) Completeness 3) Clarity. List issues as Critical/Important/Suggestion. If clean, say APPROVED." 2>&1 | tail -80
```

Parse output for **APPROVED** or listed issues (Critical/Important/Suggestion).

### Step 3: Branch on Result

- **APPROVED** → report results, commit if in git, done.
- **Issues found** → fix each issue, return to Step 2 with updated context.

### Step 4: Update Context for Next Round

For subsequent rounds, use `adversarial-review` or `task` with round context:

```bash
# Adversarial re-review after fixes
node "$CODEX_COMPANION" adversarial-review --wait "Round {N} re-review. Previous issues: {brief_list}. Verify ALL fixed. Find NEW issues. If clean, say APPROVED." 2>&1 | tail -80
```

This prevents re-reporting already-fixed issues.

## Safeguards

- **Max 10 rounds** — stop and surface to user if not approved after 10 iterations.
- **Bio safety filter** — if Codex blocks content, retry with rephrased prompt. If still blocked, fall back to `codex review --uncommitted` or inform user.
- **Sandbox issues** — if `bwrap` errors appear, switch from `review` to `task` with the diff piped in:
  ```bash
  git diff > /tmp/review.diff
  node "$CODEX_COMPANION" task "Review this diff for {criteria}: $(cat /tmp/review.diff)" 2>&1 | tail -80
  ```
- **Background mode** — for long reviews, use `--background` instead of `--wait`, then check with:
  ```bash
  node "$CODEX_COMPANION" status --json
  node "$CODEX_COMPANION" result [job-id]
  ```

## Prompt Templates and Criteria

Load detailed prompt templates from:
- **`references/prompt-templates.md`** — ready-to-use templates for spec, plan, code, and alignment reviews

For GPT-5.4 prompt best practices, also reference the codex plugin's internal skills:
- `codex:gpt-5-4-prompting` — prompt structure guidelines (XML tags, output contracts)

## Integration

After approval, commit changes (if git available), push if requested, and notify user with:
- Total rounds
- Issues found/fixed per round
- Final status: APPROVED
- Codex command used (review / adversarial-review / task)
