# Playwright E2E Test Workflow — Create, Migrate, Run & Debug

This document covers the **process** — the phases you follow when creating, migrating, running, or debugging Playwright E2E tests. For conventions on page objects, fixtures, specs, selectors, and test data, see the [test automation guidelines](Playwright-e2e-test-automation-guidelines.md) and the [FAQ](Playwright-e2e-test-automation-faq.md).

---

## ⚠️ Before You Start

**Read these sections of the guidelines FIRST:**
- [Anti-Patterns to Avoid](./Playwright-e2e-test-automation-guidelines.md#anti-patterns-to-avoid) - Critical violations that will cause rejection
- [Page Object Model](./Playwright-e2e-test-automation-guidelines.md#page-object-model-pom) - How to structure page objects
- [Selector Strategy](./Playwright-e2e-test-automation-guidelines.md#selector-strategy) - Priority order for selectors

---

## Phases

Follow these phases in order. Do not skip phases.

```
Phase 1 — Understand source & context
Phase 2 — Author or update artefacts  (only what the scenario requires)
Phase 3 — Lint check
Phase 4 — Run & debug loop             ← repeat until green
Phase 5 — Standards review              ← fix then re-run if changes made
Phase 6 — Deliver
```

---

## Phase 1 — Understand source & context

Determine the mode first:

| Mode | Starting point |
|------|---------------|
| **Migrate** | Existing Cypress spec — read every `it()` block, page object calls, and test data to reproduce equivalent coverage |
| **Create new** | Feature description or area — infer reasonable scenarios and proceed; only ask for clarification if the feature area is genuinely ambiguous |
| **PR-driven** | GitHub PR — review the diff, identify changed behaviour, update existing specs or add new ones as needed |

### Before writing

**Checklist:**

- [ ] Read the guidelines sections listed at the top of this document
- [ ] Check `playwright/page-objects/base-page.ts` for reusable methods
- [ ] Check if a page object already exists: `ls playwright/page-objects/`
- [ ] Review an existing spec in the same feature area for structural patterns

---

## Phase 2 — Author or update artefacts

Follow the guidelines for all naming, structure, selector, and scoping conventions. Only create or modify the artefacts the scenario actually requires.

---

## Phase 3 — Lint check

Run linting on every new or modified file before executing tests. Fix all reported errors.

---

## Phase 4 — Run & debug loop

### Environment setup

Install Playwright browsers if not already present:

```bash
npx playwright install chromium
```

### Run commands

**Staging (default):**

```bash
CLUSTER_NAME=<cluster> BROWSER=chromium BASE_URL=https://console.dev.redhat.com/openshift/ \
  npx playwright test <spec-path> --headed
```

**Local server:**

```bash
CLUSTER_NAME=<cluster> BROWSER=chromium BASE_URL=https://prod.foo.redhat.com:1337/openshift/ \
  npx playwright test <spec-path> --headed
```

Default to staging when no environment is specified.

### On failure

1. Inspect the failure artefacts (screenshots, error context) — see the *Inspecting Failure Artefacts* section in the guidelines.
2. Fix the root cause and re-run. Repeat until exit code 0.

### When the environment is unavailable

If Phase 4 cannot be completed (e.g., no live cluster, missing auth credentials, environment down):

1. **Do not silently skip** — state explicitly that Phase 4 was not completed and why.
2. Provide the exact run command needed so someone else can execute it.
3. Proceed to Phase 5 (standards review) — it does not depend on a passing run.
4. In the Phase 6 delivery summary, mark Phase 4 as **incomplete** with the prerequisite that must be resolved.

---

## Phase 5 — Standards review

After all tests pass (or after Phase 4 if it was skipped due to environment), audit every new or modified file against the conventions in [`Playwright-e2e-test-automation-guidelines.md`](Playwright-e2e-test-automation-guidelines.md). Pay particular attention to the **Selector Strategy**, **Anti-Patterns to Avoid**, and **Test Data** sections. Do not treat existing legacy code as a reference for selector strategy — follow the documented priority order. Fix any violations, then re-run if changes were made.

### Self-Review Checklist

Run these verification commands to detect common violations (see [Anti-Patterns to Avoid](./Playwright-e2e-test-automation-guidelines.md#anti-patterns-to-avoid) for details):

```bash
# Check for page object violations (should return ZERO)
grep -n "\.page\.\(getBy\|locator\)" playwright/e2e/**/*.spec.ts

# Check for CSS selector violations (should return ZERO)
grep -n "locator('#\|locator('\." playwright/page-objects/*.ts

# Check for wait violations (should return ZERO)
grep -n "waitForTimeout\|networkidle" playwright/e2e/**/*.spec.ts
```

**If any violations are found, refer to the guidelines for the correct approach:**
- Page Object violations → [Page Object Model](./Playwright-e2e-test-automation-guidelines.md#page-object-model-pom)
- Selector violations → [Selector Strategy](./Playwright-e2e-test-automation-guidelines.md#selector-strategy)
- Wait violations → [Anti-Patterns to Avoid](./Playwright-e2e-test-automation-guidelines.md#anti-patterns-to-avoid)

---

## Phase 6 — Deliver

Summarise the run:
- Test results: all N tests passing with exit code 0 (or note if Phase 4 was incomplete and why)
- Files created or modified
- Any selector/role corrections made and why
