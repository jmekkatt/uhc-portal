# Playwright E2E Agentic Test Creation Workflow

**Purpose:** Process orchestration for AI agents to autonomously create, execute, and fix Playwright E2E tests until passing.

**Target audience:** AI agents (Claude, Cursor, GitHub Copilot, etc.) creating E2E tests without human intervention.

**Success criteria:** Exit code 0 + zero violations detected.

**Important:** This document contains ONLY the process flow. All rules, templates, examples, and selectors are in [`Playwright-e2e-test-automation-guidelines.md`](./Playwright-e2e-test-automation-guidelines.md).

---

## Process Overview

```
Phase 0 → Read guidelines completely
Phase 1 → Understand requirements & scan codebase
Phase 2 → Create/update page object (if needed)
Phase 3 → Register fixture (if new page object)
Phase 4 → Create/update test spec
Phase 5 → Run violation detection
    └─ Violations found? → Fix → Re-run Phase 5
    └─ Zero violations? → Proceed to Phase 6
Phase 6 → Execute tests
    └─ Tests fail? → Debug (Phase 7) → Phase 6
    └─ Tests pass? → Proceed to Phase 8
Phase 8 → Final verification
Phase 9 → Deliver summary
```

---

## Phase 0: Pre-Flight

**Action:** Read the complete guidelines.

**What to read:**
- [`Playwright-e2e-test-automation-guidelines.md`](./Playwright-e2e-test-automation-guidelines.md)
- Focus sections: Page Object Model, Selector Strategy, Anti-Patterns to Avoid

**Outcome:** You understand forbidden patterns, required patterns, and best practices.

**Next:** Phase 1

---

## Phase 1: Understand Requirements

**Actions:**

1. **Identify test type:**
   - New feature → Create new spec
   - Cypress migration → Read source Cypress spec
   - Bug fix → Update existing spec

2. **Scan existing code:**
   - Check if page object exists: `playwright/page-objects/`
   - Check for similar tests: `playwright/e2e/`
   - Review BasePage utilities: `playwright/page-objects/base-page.ts`

3. **Explore application source:**
   - Find component: `src/components/`
   - Check for `data-testid` attributes in source

4. **Identify test resource:**
   - Cluster name? Resource name?
   - Will need environment variable support

**Outcome:** You know what exists, what needs to be created, and test dependencies.

**Decision:**
- Page object exists → Proceed to Phase 4 (may add methods in Phase 2 if needed)
- Page object missing → Proceed to Phase 2

---

## Phase 2: Create/Update Page Object

**Action:** Create new page object OR add methods to existing one.

**Reference:**
- File naming → See guidelines: "Page Object Model" section
- Class structure → See guidelines: "Page Object Patterns" section  
- Method patterns → See guidelines: "Locator Methods" and "Action Methods" sections
- Selector priority → See guidelines: "Selector Strategy" section

**Critical checks:**
- [ ] Class extends BasePage
- [ ] Locator methods return `Locator` (not `Promise`)
- [ ] Action methods are `async`
- [ ] Using getByRole/getByLabel/getByTestId/getByText ONLY
- [ ] NO CSS selectors
- [ ] NO hard-coded waits

**Outcome:** Page object exists with methods for test needs.

**Next:** Phase 3 (if new page object) OR Phase 4 (if existing)

---

## Phase 3: Register Page Object Fixture

**When:** Only if Phase 2 created a NEW page object.

**Action:** Register page object in fixtures.

**Reference:**
- See guidelines: "Using Fixtures" → "Worker-Scoped Fixtures" section

**Steps:**
1. Import page object class
2. Add to WorkerFixtures type
3. Add fixture definition with worker scope

**Outcome:** Page object available as fixture in tests.

**Next:** Phase 4

---

## Phase 4: Create/Update Test Spec

**Action:** Create new test spec OR update existing one.

**Reference:**
- File naming → See guidelines: "Naming Conventions" section
- Test structure → See guidelines: "Test Organization" section
- Tags → See guidelines: "Tagging Strategy" section
- Environment variables → See guidelines: "Best Practices" section

**Critical checks:**
- [ ] Uses `test.describe.serial` if sharing state
- [ ] Has proper tags (@ci, @day2, etc.)
- [ ] Uses environment variable for cluster/resource name
- [ ] Tests use ONLY page object methods
- [ ] NO direct page access in tests
- [ ] Handles unavailable features with test.skip()

**Outcome:** Test spec exists and ready for verification.

**Next:** Phase 5

---

## Phase 5: Violation Detection

**Action:** Run all violation checks. Fix ALL violations before proceeding.

**Reference:**
- Violation types and fixes → See guidelines: "Anti-Patterns to Avoid" section

**Checks to run:**

1. **Page object pattern violation check**
   - Detects: Direct page access in tests
   - Command reference: See guidelines for grep pattern

2. **CSS selector violation check**
   - Detects: CSS selectors in page objects
   - Command reference: See guidelines for grep pattern

3. **Wait violation check**
   - Detects: Hard-coded waits
   - Command reference: See guidelines for grep pattern

**Expected result:** All three checks return ZERO matches.

**Decision:**
- Violations found → Fix violations → Re-run Phase 5
- Zero violations → Proceed to Phase 6

---

## Phase 6: Execute Tests

**Action:** Run the test with environment variable.

**Reference:**
- Run commands → See guidelines: "Running Tests" section

**Expected outcomes:**

**A) Exit code 0**
- All tests pass OR skip gracefully
- Proceed to Phase 8

**B) Exit code 1**
- Tests fail
- Proceed to Phase 7

---

## Phase 7: Debug & Fix

**Action:** Systematic debugging until tests pass.

**Reference:**
- Common errors → See guidelines: "Debugging Tests" section
- Error patterns → See FAQ: "Common Issues" section

**Debug steps:**

1. **Read error output**
   - What failed? Element not found? Timeout? Wrong selector?

2. **Check screenshots**
   - Located in: `playwright-artifacts/results/`
   - Is page loaded? Element visible? Overlay blocking?

3. **Identify root cause**
   - Selector too broad?
   - Element not ready?
   - Feature unavailable for this cluster?

4. **Fix root cause**
   - Reference guidelines for correct pattern

5. **Re-run test**
   - Return to Phase 6

**Outcome:** Tests pass (exit code 0).

**Next:** Phase 8

---

## Phase 8: Final Verification

**Action:** Verify everything before declaring completion.

**Checklist:**

1. **Tests pass**
   - Last test run: exit code 0
   - All tests passed OR skipped gracefully

2. **Zero violations**
   - Re-run all three violation checks from Phase 5
   - All return ZERO matches

3. **Code quality**
   - Page object methods have clear names
   - Tests have descriptive titles
   - Tests handle conditional states
   - Environment variables used

4. **Files tracked**
   - List all files created/modified

**Decision:**
- All checks pass → Proceed to Phase 9
- Any check fails → Return to relevant phase

---

## Phase 9: Deliver Summary

**Action:** Output completion summary.

**Format:**

```
E2E Test Creation Summary
=========================

Test Results:
- X tests passed
- Y tests skipped
- Exit code: 0

Violations:
- Page object: 0
- CSS selectors: 0  
- Hard-coded waits: 0

Files Created/Modified:
- playwright/page-objects/[name]-page.ts
- playwright/fixtures/pages.ts (if new fixture)
- playwright/e2e/[area]/[name].spec.ts

Execution Command:
[command to run test with environment variable]

Status: ✅ Complete - All guidelines followed
```

**Outcome:** Task complete.

---

## Decision Points

### When Phase 2?
- New page object needed → Go to Phase 2
- Page object exists but needs methods → Go to Phase 2
- Page object exists and complete → Skip to Phase 4

### When Phase 3?
- Only if Phase 2 created NEW page object
- If adding methods to existing → Skip Phase 3

### When to loop Phase 5?
- Loop until ALL violation checks return ZERO
- Do NOT proceed to Phase 6 with violations

### When to loop Phase 6-7?
- Loop until exit code 0
- Each failure → Phase 7 → Phase 6

### When Phase 9?
- Only after Phase 8 checklist complete
- All checks must pass

---

## Self-Check Before Completion

**Answer YES to all before declaring complete:**

1. Did I read guidelines before coding? YES/NO
2. Exit code 0? YES/NO
3. Zero violations detected? YES/NO
4. All Phase 8 checks pass? YES/NO
5. Summary delivered? YES/NO

**If any NO → Return to relevant phase.**

---

## Reference Map

**When you need to know HOW:**
- How to name files → Guidelines: "Naming Conventions"
- How to write selectors → Guidelines: "Selector Strategy"
- How to structure page objects → Guidelines: "Page Object Model"
- How to write tests → Guidelines: "Test Organization"
- What to avoid → Guidelines: "Anti-Patterns to Avoid"
- How to fix errors → Guidelines: "Debugging Tests"
- How to run tests → Guidelines: "Running Tests"

**This workflow tells you WHEN to do what. Guidelines tell you HOW to do it.**

---

End of workflow.
