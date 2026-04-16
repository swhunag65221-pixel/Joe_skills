# Codex Review Prompt Templates

All templates use `--wait` for synchronous execution and end with the **standard response contract**:
```
List issues as Critical / Important / Suggestion. If no issues, say APPROVED.
```

## Canonical Command Forms

```bash
CODEX_COMPANION="/home/joe/.claude/plugins/cache/openai-codex/codex/1.0.3/scripts/codex-companion.mjs"

# Built-in review (no prompt needed — has its own contract)
node "$CODEX_COMPANION" review --wait 2>&1
node "$CODEX_COMPANION" review --wait --scope branch 2>&1

# Adversarial review (focus text, optional)
node "$CODEX_COMPANION" adversarial-review --wait "<focus text>" 2>&1

# Custom task (always needs explicit prompt with response contract)
node "$CODEX_COMPANION" task --wait "<prompt>" 2>&1
```

Always double-quote `"$CODEX_COMPANION"` and prompt strings.

---

## Code Review — Standard (Uncommitted Changes)

```bash
node "$CODEX_COMPANION" review --wait 2>&1
```

No prompt needed — the companion auto-detects changes and applies its built-in review contract.

## Code Review — Adversarial (Focused)

```bash
node "$CODEX_COMPANION" adversarial-review --wait "Focus on data leakage, temporal boundary violations, and normalization using future data" 2>&1
```

## Code Review — Re-review After Fixes

```bash
node "$CODEX_COMPANION" adversarial-review --wait "Round {N} re-review. Attempted fixes for: {brief_list}. Verify whether each is actually resolved. Find any NEW issues. List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Design Spec — First Review

```bash
node "$CODEX_COMPANION" task --wait "You are a spec document reviewer. Review {file_path}.

Check:
1. Mathematical consistency: equations, parameter tables, and text descriptions match
2. Completeness: all sections present, no gaps
3. Clarity: could a developer implement without ambiguity
4. Parameter identifiability: free params feasible for data size
5. Risk coverage: risks well-identified
6. Internal consistency: no contradictions between sections

List issues as Critical / Important / Suggestion. If no issues, say APPROVED. Be strict and concise." 2>&1
```

## Design Spec — Subsequent Round

```bash
node "$CODEX_COMPANION" task --wait "Spec review round {N}. Review {file_path}.

Attempted fixes for {count} issues: {brief_list_of_issues}.
Verify whether each is actually resolved. Find any NEW issues.
List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Implementation Plan — First Review

```bash
node "$CODEX_COMPANION" task --wait "You are a plan document reviewer. Review {plan_path} against its spec at {spec_path}.

Check:
1. Spec coverage: every spec requirement has a corresponding task
2. Task granularity: bite-sized steps (2-5 min each)
3. File paths: exact and consistent with file map
4. Code completeness: complete code, not placeholders
5. Test quality: edge cases covered, expected outputs specified
6. Dependencies: correct dependency graph
7. Missing tasks: anything from spec not covered

List issues as Critical / Important / Suggestion. If no issues, say APPROVED. Be strict and concise." 2>&1
```

## Implementation Plan — Subsequent Round

```bash
node "$CODEX_COMPANION" task --wait "Plan review round {N}. Review {plan_path} against spec at {spec_path}.

Attempted fixes for {count} issues: {brief_list}.
Verify whether each is actually resolved. Find any NEW issues.
List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Implementation Plan — Post Spec Update

```bash
node "$CODEX_COMPANION" task --wait "You are a plan reviewer. Review {plan_path} against its spec {spec_path}.

The spec has been updated through {N} review rounds. Key spec changes that the plan must reflect:
{numbered_list_of_spec_changes}

Check: spec coverage, code completeness, test quality, dependencies, consistency.
List issues as Critical / Important / Suggestion. If no issues, say APPROVED. Be strict and concise." 2>&1
```

## Code File Review (Specific File)

```bash
node "$CODEX_COMPANION" task --wait "Review code quality and correctness of {file_path}. Focus on:
1. Bugs and logic errors
2. Code organization
3. Edge case handling
4. Consistency with project patterns

List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Code Review with Spec Context

```bash
node "$CODEX_COMPANION" task --wait "Review {file_path} for alignment with the design spec at {spec_path}. Check:
1. Implementation matches spec equations
2. Parameter names consistent with spec
3. Edge cases from spec are handled
4. Tests cover spec requirements

List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Spec vs Plan Alignment Check

```bash
node "$CODEX_COMPANION" task --wait "Cross-check {plan_path} against {spec_path}. For each spec section, verify:
1. A corresponding plan task exists
2. The plan code matches spec equations exactly
3. Naming is consistent (functions, parameters, variables)
4. No spec requirements are missing from the plan

List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

## Non-Git / Clean Worktree Review

Reference files by path and let Codex read them directly:

```bash
node "$CODEX_COMPANION" task --wait "Review /path/to/file.py for correctness, bugs, and code quality. List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

**Do NOT inline file contents with `$(cat ...)`** — this is fragile for quotes, backticks, `$()` expansions, and large files.

## Sandbox Workaround (bwrap failures)

When `review --wait` fails due to sandbox issues, write the diff to a file and reference it:

```bash
git diff > /tmp/review.diff
node "$CODEX_COMPANION" task --wait "Review the diff at /tmp/review.diff for correctness, bugs, and consistency. List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

**Do NOT inline large diffs with `$(cat ...)`** — this hits shell argument size limits.

---

## Fallback: Direct Codex CLI (No Plugin)

```bash
# Document / spec review
codex exec "{prompt ending with response contract}" 2>&1

# Uncommitted code review
codex review --uncommitted 2>&1

# Adversarial review (no direct equivalent — save diff to file first)
git diff > /tmp/review.diff
codex exec "Adversarial review of /tmp/review.diff. Focus: {focus}. List issues as Critical / Important / Suggestion. If no issues, say APPROVED." 2>&1
```

**Do NOT inline diffs with `$(git diff)`** — use file path references instead.
