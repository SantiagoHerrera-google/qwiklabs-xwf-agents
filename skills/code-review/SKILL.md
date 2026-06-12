---
name: code-review
description: Performs a comprehensive agentic code review of local branch changes. Runs validation linters (RuboCop, Sorbet, i18n-tasks, Haml-Lint) directly to maximize performance and avoid sandbox permission prompts. Performs deep ActiveRecord N+1 query performance optimization analysis. Active on queries like "code-review", "revisión de código", "haz un code review", "ejecuta el code review", "review local changes", or when validating a branch.
---

# Code Review Skill

Follow these instructions step-by-step to perform a robust code review on the current git branch.

## Step 1: Run Pre-Check Validation Commands Directly
Execute individual validation commands directly in the user's terminal workspace using `run_command`. 

> [!IMPORTANT]
> **Zero-Prompt Optimization:** Do NOT execute validation linters wrapped inside a custom local bash script (like `.sh` files). Custom bash scripts are classified as "dangerous commands" in the sandbox, which triggers an interactive security approval dialog for the user on every execution.
>
> Instead, run standard commands (prefixed by `git` or `bundle exec`) directly! Because the user's active workspace permissions already permanently allow the `git` and `bundle` command prefixes, the entire validation pipeline below will execute **100% silently and automatically** with ZERO permission prompts!

1. **Find Merge Base & Branch Diff Metadata:**
   * Get the tracking merge base:
     `git merge-base origin/main HEAD || git merge-base main HEAD`
   * Identify all changed files on this branch, excluding deleted ones (which would otherwise cause linter crashes):
     `git diff --diff-filter=d --name-only {merge_base} HEAD`
   * Filter this file list into arrays of changed Ruby files (`.rb` extension) and changed Haml files (`.haml` extension).

2. **Check A: RuboCop (Optimized with Smart Autofix & Manual Fixes):**
   * If there are changed Ruby files, execute the check:
     `bundle exec rubocop {changed_ruby_files}`
   * **Case A1: If RuboCop fails:**
     * Attempt to automatically repair violations on the changed files:
       `bundle exec rubocop -A {changed_ruby_files}`
     * Recheck to verify if the violations were resolved:
       `bundle exec rubocop {changed_ruby_files}`
     * If it passes on the recheck, record that **"RuboCop"** applied successful autofixes.
     * If it still fails:
       * Analyze the remaining violations.
       * If they are minor (formatting, style, simple fixes), attempt to fix them manually.
       * If they require business logic decisions or important design decisions, **pause execution** and ask the user for guidance to keep moving forward.
       * Recheck after manual fixes. If it passes, proceed.
       * If it still fails and cannot be resolved, collect the remaining violations log, **abort the review immediately**, and proceed directly to **Step 3 (Generate Report)**.
   * **Case A2: If RuboCop passes on the first run:**
     * Proceed to Check B.

3. **Check B: Sorbet Type Checker (Non-Lethal):**
   * Execute the typecheck command:
     `bundle exec srb tc`
   * If it fails, collect the output log, but **do NOT abort the review**. Record the errors to be reported in the final report, and proceed to Check C.

4. **Check C: Translation Keys Verification:**
   * Execute the missing keys check:
     `bundle exec i18n-tasks missing en -t used`
   * If it fails, collect the output log, **abort the review**, and proceed to **Step 3**.

5. **Check D: Haml-Lint (Optimized with Smart Autofix):**
   * If there are changed Haml files, execute the check:
     `bundle exec haml-lint -p {changed_haml_files}`
   * **Case D1: If Haml-Lint fails:**
     * Attempt to automatically repair violations on the changed files:
       `bundle exec haml-lint -A {changed_haml_files}`
     * Recheck to verify if the violations were resolved:
       `bundle exec haml-lint -p {changed_haml_files}`
     * If it passes on the recheck, record that **"Haml-Lint"** applied successful autofixes.
     * If it still fails, collect the remaining violations log, **abort the review immediately**, and proceed directly to **Step 3**.
   * **Case D2: If Haml-Lint passes on the first run:**
     * Proceed to Step 2.

---

## Step 2: Deep Architectural & Performance Analysis (Staff RoR Context)
*(Only execute this step if all pre-check validation commands completed successfully or resolved via autofix)*

Act as a Staff-level Ruby on Rails Engineer conducting a code review. Review only the changes on the current branch against the merge base, focusing on architecture, performance, security, and maintainability (excluding minor formatting or style nits which linters handle).

Perform deep, logical reasoning to analyze the code around the following 5 distinct areas:

1. **Intent Summary:**
   * Synthesize and summarize in 1-2 sentences what the changed code is trying to accomplish conceptually, relying purely on source analysis (not PR metadata).

2. **Architectural & Performance Red Flags:**
   * **Database & Memory Performance:** Scan for N+1 query problems (e.g. iterating `.each`/`.map` over associations without `.includes`/`.preload`), unnecessary DB hits, database locks, or memory bloat risks.
   * **Design Patterns & Responsibility:** Identify single responsibility principle (SRP) violations, bloated controllers/models, or improper separation of concerns.
   * **Simplicity & Readability:** Check if there are cleaner, simpler, more idiomatic ways to implement the logic (e.g. early returns, memoization, native Rails helpers, ActiveRecord shortcuts).

3. **Adversarial Edge Cases:**
   * Think like a chaos engineer. Identify inputs, nil parameter handling, unhandled states, or missing data invariants that would trigger 500 errors, infinite loops, or unpredictable state corruption.

4. **Security & Data Integrity:**
   * **Input Sanitization & Parameters:** Look for unsafe parameter handling (e.g. missing Strong Parameters, direct eval, SQL injections).
   * **Access & Authorization:** Verify if access controls, multi-tenant scopes, or authorization checks (e.g. Pundit/CanCan) are implemented or bypassed.
   * **Concurrency Risks:** Identify potential race conditions, concurrent resource conflicts, or transactional inconsistencies.

5. **The "Blast Radius":**
   * Map dependencies of the modified code to the rest of the application. Assess which downstream services, background workers, databases, or models are at risk of side-effects or breakages from this change.

---

## Step 3: Present Results Directly in the Chat
Respond to the user with the complete Code Review Report directly in the chat window. The response must follow this structure:

# Code Review Report

- **Date:** {Current Local Time}
- **Merge Base:** {Merge base commit/branch used}

## 📊 Summary
- **Pre-checks Status:** {🔴 FAILED | 🟢 PASSED | 🟡 PASSED WITH WARNINGS}
- **Smart Autofixes Applied:** {None | RuboCop | Haml-Lint | RuboCop & Haml-Lint}
- **Critical Red Flags / Edge Cases:** {Yes (Count) | No | Skipped}

## 🔍 Validation Pre-Checks
{List each check and its status: RuboCop, Sorbet, i18n-tasks, Haml-lint. Note if Sorbet failed but was treated as non-lethal}

{IF AUTOFIXES APPLIED:
### 🔧 Smart Autofixes Applied
List each engine that successfully applied autofixes (RuboCop or Haml-Lint). Mention that the formatting updates were automatically written back to the local branch, and prompt the user that they can review the resulting diff.
}

{IF SORBET FAILED:
### ⚠️ Sorbet Warnings
Paste a truncated version of the Sorbet output logs (e.g. first 10-20 lines) if they are very long, and mention that these errors were treated as non-lethal and did not abort the review. Save the full log in a scratch file: `[sorbet_errors.txt](file://{Absolute Path to scratch}/sorbet_errors.txt)` and link to it.
}

{IF FAILED:
### ❌ Failed Linter Logs
Identify which lethal linter (RuboCop, i18n-tasks, Haml-Lint) failed and paste the raw failed output logs from the terminal run here.
> **Abort Warning:** Deep architectural code review was skipped because critical pre-checks failed.
}

{IF PASSED or PASSED WITH WARNINGS:
### 🎯 Intent Summary
{1-2 sentences explaining what the code is trying to accomplish}

### ⚡ Architectural & Performance Red Flags
For each issue detected, provide:
1. **File & Line Number**: Path and specific lines.
2. **Problem Description**: Why this is an issue (N+1 query, memory bloat, bloated class, logic redundancy).
3. **Suggested Fix**: A Git Diff block showing a cleaner/faster implementation (e.g. adding `.includes`, memoizing with `||=`, early return).

### 💥 Adversarial Edge Cases & Failure Modes
For each vulnerability:
1. **Scenario**: What inputs or conditions trigger the bug (e.g., passing nil parameter, user having no active memberships).
2. **Impact**: Exception type or visual/state bug caused.
3. **Remediation**: Ruby/Rails pattern to guard against it.

### 🔒 Security & Data Integrity
For each risk:
1. **Vulnerability**: Describe the security gap (unsafe query, missing authorization, double submit race condition).
2. **Suggested Fix**: Clear code block showing the secure, transaction-wrapped, or scoped resolution.

### 🌐 The "Blast Radius" & Downstream Impact
- **Downstream Services**: {List dependent API calls, external queues affected}
- **Background Jobs**: {Jobs enqueued by these controllers, with risk analysis}
- **Model Side-effects**: {Callbacks triggered, cache invalidations, cascading deletes}
}
