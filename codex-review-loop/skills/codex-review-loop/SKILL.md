---
name: codex-review-loop
description: This skill should be used when the user asks to "review with codex", "codex review loop", "iterative review", "review until approved", "codex審查", "迭代審查", "loop until no issues", or when completing a spec, plan, or code that needs quality review via Codex CLI. Implements a write-review-fix loop where Claude Code produces content and Codex CLI reviews iteratively until approved.
---

# Codex Review Loop

Iterative review workflow: write → Codex review → fix → re-review → repeat until APPROVED.

## Prerequisites

Requires the **openai-codex** marketplace plugin. Run `/codex:setup` to verify.

```bash
CODEX_COMPANION="${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs"
```

> **Note:** `${CLAUDE_PLUGIN_ROOT}` is set by the codex plugin at runtime. If unavailable, the current absolute path is:
> `/home/joe/.claude/plugins/cache/openai-codex/codex/1.0.3/scripts/codex-companion.mjs`

## Review Types

All commands use `--wait` for synchronous execution (required by the loop).

| Type | Command |
|------|---------|
| Code (uncommitted) | `node "$CODEX_COMPANION" review --wait` |
| Code (adversarial) | `node "$CODEX_COMPANION" adversarial-review --wait "focus text"` |
| Code (branch) | `node "$CODEX_COMPANION" review --wait --scope branch` |
| Document / Custom | `node "$CODEX_COMPANION" task --wait "{prompt}"` |

**Note:** `review` and `adversarial-review` have built-in review contracts that produce structured output. The `task` command requires you to append the response contract explicitly in your prompt.

## Response Contract

**Every review command must produce output ending with one of:**
- `APPROVED` — no issues found.
- A list of issues tagged as `Critical`, `Important`, or `Suggestion`.

All prompt templates in `references/prompt-templates.md` enforce this contract. When writing custom prompts, always append:
```
List issues as Critical / Important / Suggestion. If no issues, say APPROVED.
```

## Core Loop

### Step 1: Determine Type and Select Command

| What to review | Command |
|----------------|---------|
| Uncommitted code changes | `review --wait` |
| Uncommitted + specific concern | `adversarial-review --wait "focus"` |
| Full branch diff | `review --wait --scope branch` |
| Document / spec / plan | `task --wait "{prompt}"` (use template) |
| Files outside git repo | `task --wait "{prompt}"` referencing file path |
| Clean worktree, review specific file | `task --wait "{prompt}"` referencing file path |

### Step 2: Execute Review

```bash
node "$CODEX_COMPANION" review --wait 2>&1
```

**Do NOT truncate output with `tail`.** Capture the full output to ensure the `APPROVED` token or highest-severity issue is not lost. If the output is very long, scan for `APPROVED` or `Critical/Important/Suggestion` keywords.

### Step 3: Branch on Result

- **APPROVED** → report results, commit if in git, done.
- **Issues found** → present findings to user first (follow `codex-result-handling` rules: do NOT auto-fix without user confirmation). Once user confirms which issues to fix, apply fixes and return to Step 2.

### Step 4: Update Context for Next Round

For subsequent rounds, use the same command type as the initial review (e.g., `adversarial-review` for code, `task --wait` for documents) with neutral phrasing:

```bash
node "$CODEX_COMPANION" adversarial-review --wait "Round {N} re-review. Attempted fixes for: {brief_list}. Verify whether each is actually resolved. Find any NEW issues. If all clean, say APPROVED." 2>&1
```

**Avoid confirmation bias**: say "attempted fixes for" not "all fixed". Let the reviewer independently verify.

## Safeguards

### Max Rounds
- **Max 10 rounds** — stop and surface to user if not approved after 10 iterations.
- If Codex keeps re-raising the same issue after a reasonable fix attempt, present the disagreement to the user with both perspectives and let them decide.

### Bio Safety Filter
If Codex blocks content, retry with rephrased prompt. If still blocked, fall back to direct CLI.

### Sandbox (bwrap) Issues
If `review --wait` fails with `bwrap` errors, use `task --wait` with the diff passed via a temp file:

```bash
# Write diff to file (avoids shell argument limits on large diffs)
git diff > /tmp/review.diff
# Pass file path, NOT file contents, to avoid argv overflow
node "$CODEX_COMPANION" task --wait "Review the diff at /tmp/review.diff for correctness, bugs, and consistency. List issues as Critical/Important/Suggestion. If no issues, say APPROVED." 2>&1
```

**Do NOT inline large diffs with `$(cat ...)`** — this hits shell argument size limits.

## Prompt Templates

Load from **`references/prompt-templates.md`**. All templates end with the standard response contract.

Consider using the `gpt-5-4-prompting` skill (from the codex plugin) to optimize custom prompts before sending to Codex.

## Integration

After approval, commit changes (if git available), push if requested, and notify user with:
- Total rounds
- Issues found/fixed per round
- Final status: APPROVED
- Codex command used

For complex fixes that are difficult to implement in the loop, consider delegating to `/codex:rescue` which can hand substantial coding tasks to Codex directly.
