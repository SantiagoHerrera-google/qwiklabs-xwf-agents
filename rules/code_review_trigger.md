---
trigger: always_on
---

# Automatic Code Review Triggering Rule

## Objective
Ensure that all code reviews, validations, and branch audits are performed in a consistent, automated, and comprehensive manner using the standard `@code-review` custom skill.

## Triggering Guidelines
1. **Chat Triggers:**
   * Whenever the user asks for a code review or code validation using any language or expression (e.g., `"code-review"`, `"revisión de código"`, `"haz un code review"`, `"ejecuta el code review"`, `"review my code"`, `"audita mi rama"`, `"check my changes"`), you **MUST** automatically and immediately activate the `@code-review` skill.
   * If a user requests a validation check, do not perform individual tool steps (like manually running rubocop or type checkers) in separate terminal instances; instead, call the `@code-review` skill to handle validation unified under a single, structured report.

2. **Branch Modifications / Git Auditing:**
   * If the user asks if the current branch looks good, if tests are clean, or is preparing to submit/push their local branch, you **MUST** suggest running the `@code-review` skill first to ensure zero regressions, formatting issues, typecheck errors, or N+1 queries exist.

3. **Strict Compliance:**
   * Do not bypass the early abort strategy defined in the `@code-review` skill. If pre-checks fail, report the failures immediately as directed by the skill report structure.
