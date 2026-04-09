# Codex Review Prompt Templates

## Quick Reference: Which Command?

| Scenario | Command |
|----------|---------|
| Review uncommitted code | `node "$CODEX_COMPANION" review --wait` |
| Focused adversarial review | `node "$CODEX_COMPANION" adversarial-review --wait "{focus}"` |
| Review full branch | `node "$CODEX_COMPANION" review --wait --scope branch` |
| Review a document/spec | `node "$CODEX_COMPANION" task "{prompt}"` |
| Fix + review (write mode) | `node "$CODEX_COMPANION" task --write "{prompt}"` |

Where `CODEX_COMPANION` is:
```bash
CODEX_COMPANION="/home/joe/.claude/plugins/cache/openai-codex/codex/1.0.3/scripts/codex-companion.mjs"
```

---

## Code Review — Standard (Uncommitted Changes)

```bash
node "$CODEX_COMPANION" review --wait
```

No prompt needed — the companion auto-detects uncommitted changes and applies its built-in review contract.

## Code Review — Adversarial (Focused)

```bash
node "$CODEX_COMPANION" adversarial-review --wait "Focus on data leakage, temporal boundary violations, and normalization using future data"
```

Use adversarial mode when you have a specific concern. The focus text steers the review.

## Code Review — Re-review After Fixes

```bash
node "$CODEX_COMPANION" adversarial-review --wait "Round {N} re-review. Previous round found {count} issues: {brief_list}. All fixed. Verify ALL previous issues are resolved. Find any NEW issues. If clean, say APPROVED."
```

## Design Spec — First Review

```bash
node "$CODEX_COMPANION" task "You are a spec document reviewer. Review {file_path}.

Check:
1. Mathematical consistency: equations, parameter tables, and text descriptions match
2. Completeness: all sections present, no gaps
3. Clarity: could a developer implement without ambiguity
4. Parameter identifiability: free params feasible for data size
5. Risk coverage: risks well-identified
6. Internal consistency: no contradictions between sections

List issues as Critical / Important / Suggestion. If no issues, say APPROVED. Be strict and concise."
```

## Design Spec — Subsequent Round

```bash
node "$CODEX_COMPANION" task "Spec review round {N}. Review {file_path}.

Previous round found {count} issues: {brief_list_of_issues}. All fixed.
Verify ALL previous issues are resolved. Find any NEW issues.
If clean, say APPROVED."
```

## Implementation Plan — First Review

```bash
node "$CODEX_COMPANION" task "You are a plan document reviewer. Review {plan_path} against its spec at {spec_path}.

Check:
1. Spec coverage: every spec requirement has a corresponding task
2. Task granularity: bite-sized steps (2-5 min each)
3. File paths: exact and consistent with file map
4. Code completeness: complete code, not placeholders
5. Test quality: edge cases covered, expected outputs specified
6. Dependencies: correct dependency graph
7. Missing tasks: anything from spec not covered

List issues as Critical / Important / Suggestion. Be strict and concise."
```

## Implementation Plan — Subsequent Round

```bash
node "$CODEX_COMPANION" task "Plan review round {N}. Review {plan_path} against spec at {spec_path}.

Previous round found {count} issues: {brief_list}. All fixed.
Verify ALL previous issues are resolved. Find any NEW issues.
If clean, say APPROVED."
```

## Implementation Plan — Post Spec Update

```bash
node "$CODEX_COMPANION" task "You are a plan reviewer. Review {plan_path} against its spec {spec_path}.

The spec has been updated through {N} review rounds. Key spec changes that the plan must reflect:
{numbered_list_of_spec_changes}

Check: spec coverage, code completeness, test quality, dependencies, consistency.
List issues as Critical/Important/Suggestion. Be strict and concise."
```

## Code File Review (Specific File)

```bash
node "$CODEX_COMPANION" task "Review code quality and correctness of {file_path}. Focus on:
1. Bugs and logic errors
2. Code organization
3. Edge case handling
4. Consistency with project patterns
Keep response concise."
```

## Code Review with Context

```bash
node "$CODEX_COMPANION" task "Review {file_path} for alignment with the design spec at {spec_path}. Check:
1. Implementation matches spec equations
2. Parameter names consistent with spec
3. Edge cases from spec are handled
4. Tests cover spec requirements
List any mismatches."
```

## Spec vs Plan Alignment Check

```bash
node "$CODEX_COMPANION" task "Cross-check {plan_path} against {spec_path}. For each spec section, verify:
1. A corresponding plan task exists
2. The plan code matches spec equations exactly
3. Naming is consistent (functions, parameters, variables)
4. No spec requirements are missing from the plan
List mismatches only."
```

## Diff-Based Review (Sandbox Workaround)

When `review` fails due to `bwrap` sandbox issues, pipe the diff manually:

```bash
git diff > /tmp/review.diff
node "$CODEX_COMPANION" task "Review this code patch for correctness, bugs, and consistency. List issues as Critical/Important/Suggestion. If clean, say APPROVED.

$(cat /tmp/review.diff)"
```

## Fallback: Direct Codex CLI (No Plugin)

If the codex plugin is not installed:

```bash
# Document review
codex exec "{prompt}" 2>&1 | tail -60

# Uncommitted code review
codex review --uncommitted 2>&1 | tail -60
```
