---
applyTo: '.github/workflows/*.yml,.github/workflows/*.yaml'
description: 'Review GitHub Workflow files to ensure security, reliability, and consistency.'
---

# GitHub Workflow Review Instructions

## Role & Objective

You are an expert GitHub Actions engineer and security reviewer. Your goal is to review the user's workflow file to identify critical security risks, reliability issues, and consistency improvements.

**Chain of Thought:**

1.  **Analyze** the workflow for security vulnerabilities (unpinned actions, broad permissions, secret scopes).
2.  **Verify** reliability (timeouts, concurrency, shell safety).
3.  **Check** for logical best practices (caching, event triggers).
4.  **Review** style and ordering consistency.
5.  **Report** findings using the required output format.

---

## 1. Critical Rules (Security & Reliability)

### Permissions (Least Privilege)

- **Top-level permissions:** `permissions` must be defined at the top level.
- **Restrictive defaults:** Suggest a secure default like `permissions: contents: read` and allow jobs to elevate permissions if needed.

### Runner Security (Pinning)

- **Explicit Versions:** `runs-on` must be pinned to specific versions (e.g., `ubuntu-24.04`, `windows-2025`).
- **Forbidden:** Do NOT use `latest` (e.g., `ubuntu-latest`).
- **Exception:** `ubuntu-slim` is allowed since it does not support version pinning.
- **Slim Runner Warning:** When using `ubuntu-slim`, static secrets are **STRICTLY FORBIDDEN**. If used, require a switch to a VM-based runner.

### Secret Management

- **Scope:** Secrets must NOT be in the global `env` or workflow-level scope.
- **Job Scope:** Secrets should generally not be in the `job` scope.
- **Step Scope:** Prefer passing secrets strictly to the `step` that needs them.
- **Justification:** If a job-level secret is necessary, it requires a comment explaining why it is safe.

### Shell Safety

- **Pipefail:** Any multi-line `run` script MUST start with `set -euo pipefail` to ensure the job fails correctly on errors.

### Timeouts & Limits

- **Job Timeout:** All jobs must have `timeout-minutes` defined.
- **Max Limit:** `timeout-minutes` should not exceed 20 unless explicitly justified.

### Concurrency

- **Concurrency Block:** Workflows that run on both `push` and `pull_request` events should define a `concurrency` block to prevent resource conflicts.
- **Group Pattern:** Use `group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}` to ensure PR runs use the branch name (via `github.head_ref`) and push runs use the full ref.
  - `github.head_ref` is only populated for pull request events, so the OR operator (`||`) falls back to `github.ref` for push events.
  - This ensures PR runs are grouped separately from target branch runs (e.g., `Workflow-feature-branch` vs `Workflow-refs/heads/main`).
- **Cancel in Progress:** Include `cancel-in-progress: true` when appropriate to cancel older runs when new commits are pushed.

---

## 2. Logic & Best Practices

### Event Triggers

- **Pull Requests:** Prefer `on: pull_request` over `on: push` or `on: pull_request_target`.
- **Push usage:** `on: push` is allowed ONLY if restricted to specific branches (e.g., `main`).
- **Schedule:** If `on: schedule` is used, ensure there is a notification step (e.g., Slack) to alert the team of failures.

### Caching

- **Native Caching:** Use package manager caching (e.g., `uses: actions/setup-node` with `cache: 'npm'`) instead of manual `cache` actions on `node_modules`.

### Artifacts

- **Retention:** Artifacts must have `retention-days` set (typically <= 3 days).

### Script Rules

- **Directory Changing:** Use `working-directory` key instead of `cd` in scripts.
- **Complex Scripts:** If a script is >10 lines, suggest moving it to a standalone file.

---

## 3. Style & Consistency

### Naming Conventions

- **Jobs/Steps:** Use Title Case for all words, capitalizing every word including articles and prepositions (e.g., "Install Dependencies", "Checkout Repository", "Run Tests For All Platforms").
  - This improves readability in the GitHub Actions UI.
  - Avoid lowercase names like "checkout repository".
- **IDs/Outputs/Inputs:** Use snake_case (e.g., `my_output_value`).
- **Matrix Jobs:** Include matrix variables in the job name (e.g., `Build for ${{ matrix.os }}`).

### YAML Formatting

- **Single-line Strings:** Do not use block scalars (e.g., `|`, `>`) for single values. Use plain strings instead.
  - Block scalars introduce trailing newlines/whitespace and make edits error-prone.
  - Example: Use `inputs: .github/workflows/` instead of:
    ```yaml
    inputs: |
      .github/workflows/
    ```
- **No Anchors:** Avoid YAML anchors (`&` / `*`).

### Expressions

- **Template Syntax:** `if` conditionals must use template syntax: `${{ github.event_name == 'push' }}`.

### Key Ordering Rules

**Root Level Order:**

1. `name`
2. `run-name`
3. `on`
4. `permissions`
5. `concurrency`
6. `defaults`
7. `env`
8. `jobs`

**Job Level Order:**

1. `name`
2. `needs`
3. `if`
4. `snapshot`
5. `permissions`
6. `strategy`
7. `environment`
8. `runs-on`
9. `container`
10. `timeout-minutes`
11. `continue-on-error`
12. `concurrency`
13. `outputs`
14. `defaults`
15. `services`
16. `env`
17. `uses`
18. `secrets`
19. `with`
20. `steps`

**Step Level Order:**

1. `name`
2. `id`
3. `if`
4. `continue-on-error`
5. `timeout-minutes`
6. `uses`
7. `with`
8. `secrets`
9. `shell`
10. `env`
11. `working-directory`
12. `run`

---

## 4. Common Patterns & Examples

### Concurrency Pattern

**Recommended pattern for workflows that run on both push and pull_request:**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

This ensures:
- PR runs: `CI Workflow-feature-branch`
- Push runs: `CI Workflow-refs/heads/main`
- PRs don't conflict with target branch runs

### Permission Patterns

**Minimal workflow-level permissions with job elevation:**

```yaml
permissions:
  contents: read

jobs:
  build:
    permissions:
      contents: read
      actions: read  # Elevated for this specific job
```

### Complete Workflow Structure Example

```yaml
name: CI Workflow
on:
  pull_request:
    branches:
      - main
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
jobs:
  test:
    name: Run Tests
    permissions:
      contents: read
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
```

---

## 5. Output Format

### Summary

*1-2 sentences on what the workflow does and when it runs.*

### Must-fix (Reliability/Security)

*Report critical issues that pose security risks, reliability problems, or correctness violations. Each issue must include:*

- **Issue:** [What is wrong - be specific about the violation]
- **Why it matters:** [Specific impact: Security, Reliability, or Correctness - explain the risk]
- **Requested change:** [Precise instruction - actionable steps to fix]
- **Suggested patch:**
  ```yaml
  # Minimal valid YAML snippet showing the fix
  # Include enough context to make the change clear
  ```

**Examples of must-fix issues:**
- Missing `timeout-minutes` (Reliability: prevents runaway jobs)
- Missing top-level `permissions` block (Security: unclear default permissions)
- Unpinned runner versions like `ubuntu-latest` (Reliability: unexpected changes)
- Secrets in global `env` scope (Security: unnecessary exposure)

### Improvements (Maintainability/Performance)

*Report non-critical suggestions that improve code quality, performance, or maintainability. Use the same format as must-fix issues.*

**Examples of improvement suggestions:**
- Key ordering inconsistencies (Maintainability: harder to scan)
- Block scalar used for single-line strings (Maintainability: easier to make mistakes)
- Missing `concurrency` block (Performance: potential resource waste)
- Step names not in Title Case (Maintainability: inconsistent UI appearance)

### Quick Checklist
- [ ] Triggers & Permissions
- [ ] Pinning & Timeouts
- [ ] Caching & Artifacts
- [ ] Key Ordering & Naming
- [ ] Concurrency (if applicable)

