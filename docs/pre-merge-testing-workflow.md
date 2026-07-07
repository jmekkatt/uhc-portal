# Pre-Merge Testing Workflow

This workflow guides comprehensive pre-merge testing of pull requests and feature branches to identify functional bugs, regressions, and test coverage gaps before code reaches production.

## When to Use

Use this workflow when:
- Conducting pre-merge review and testing
- Finding all functional issues in a feature branch
- Auditing commits before merge
- Assessing regression impacts of code changes
- Validating test coverage for new features
- User says "pre-merge test", "review for bugs", or "find all functional issues"

## Workflow

### Step 1 — Identify scope

```bash
# List commits introduced by the feature branch (not yet in base)
git log <base-branch>..HEAD --oneline

# Get full diff of all changed files
git diff <base-branch>...HEAD
```

Ask the user for the base branch if not provided. Default to `main`.

### Step 2 — Collect context

For every file touched by the diff:

1. **Read the changed source file** to understand the before/after intent.

2. **Search for matching Playwright specs** (`playwright/e2e/`) and **unit tests** (`*.test.ts`) that cover the changed component or function. Use two passes:
   - **Direct match**: search for specs that import or reference the changed component or function by name.
   - **Route/page match**: identify the page or route the changed component belongs to (e.g. from its file path or parent component), then search `playwright/e2e/` broadly for any spec that navigates to the same page or route — even if it does not directly import the changed file. These indirect specs are high-risk for UI behaviour regressions.
   
   Read all matched specs and page objects (`playwright/page-objects/`) to understand what UI behaviour and assertions are currently expected.

3. **Read the PR description** if the user pastes it, or ask for it.

### Step 3 — Identify functional bugs and regression impacts

Focus only on bugs and regressions introduced by the new commits. Do not report pre-existing issues.

Look for:
- Logic errors, off-by-one, incorrect conditions, missing guard clauses
- Broken UI interactions or state management regressions
- API calls with wrong payloads, missing params, or unhandled error paths
- Race conditions or async/await misuse
- Accessibility or form validation regressions
- Missing or incorrect test coverage for the new behaviour

**Regression impact analysis** — for each bug or risky change, assess:
- Which existing features, pages, or user flows could break as a side effect
- Whether shared utilities, mock data, or page objects were modified in ways that affect other consumers
- Whether the change alters behaviour that existing Playwright or unit tests rely on (even if those tests still pass, the tested behaviour may have shifted)
- The blast radius: is the impact isolated to one component, or does it ripple across multiple views

### Step 4 — Output the bug report

Print the report inline using the template below. Group bugs under severity headings.

---

## Bug Report Template

```markdown
## Pre-Merge Testing Report — <branch-name>

Diff base: `<base-branch>`  |  Commits reviewed: <N>  |  Files changed: <N>

---

### 🔴 Critical  (blocks merge / data loss / security risk)

#### BUG-001 · <Short title>

**Description**  
<One paragraph explaining what is wrong and why it matters.>

**Simulation setup** (if required)  
<Describe how to simulate the conditions needed to reproduce this bug. Include specific steps for:>
- Network conditions (block requests, simulate timeouts, slow connections)
- API mocking (mock error responses, specific status codes, malformed data)
- Browser DevTools modifications (throttling, offline mode, disable cache)
- State setup (specific user roles, cluster states, feature flags)
- Example:
  - "Mock the API to return 503 status with no response body"
  - "Use Chrome DevTools → Network → Throttling → Offline"
  - "Mock apiRequest.post to reject with: `new Error('Network timeout')`"

**Steps to reproduce**  
1. <Step 1>
2. <Step 2>
3. <Step N>

**Expected behaviour**  
<What should happen according to the PR description or existing tests.>

**Actual behaviour**  
<What actually happens with the new code.>

**Code reference**  
```ts
// The problematic change
<snippet>
```

**Regression impact**  
<Which existing features, pages, or user flows could break as a side effect. Note the blast radius (isolated vs cross-cutting) and any downstream consumers affected.>

**Missing test coverage**  
<Note any missing or inadequate test coverage for this bug.>

**Affected file(s)**  
`path/to/file.ts` — line <N>

---

### 🟡 Major  (user-visible bug, degraded UX, but workaround exists)

#### BUG-002 · ...
(same template)

---

### 🟠 Minor  (edge case, cosmetic, low impact)

#### BUG-003 · ...
(same template)

---

### ✅ No issues or regression risks found in
- `path/to/clean-file.ts`
```

---

## Severity Guide

| Severity | Criteria |
|----------|----------|
| 🔴 Critical | Data loss, security hole, crash, blocks core user flow |
| 🟡 Major | Wrong output, broken feature, regression visible to most users |
| 🟠 Minor | Edge case, cosmetic glitch, misleading label, low-traffic path |

## Project-Specific Notes

- Tests live in `playwright/e2e/` (Playwright) and `src/**/*.test.ts` (unit/React Testing Library).
- Page objects live in `playwright/page-objects/`.
- When a bug is detectable by a new or updated test, note it under the bug entry as **"Missing test coverage"**.
- Always cross-reference changed components with existing E2E specs that might test the same UI flows indirectly.
- When shared mock data or utilities are modified, check all consumers for regression impacts — not just the feature under review.

## Common Simulation Techniques

When bugs require specific conditions to reproduce, document the simulation setup clearly:

### Network Conditions
```javascript
// Browser DevTools
Network → Throttling → Offline
Network → Throttling → Slow 3G
Network → Block request URL

// Playwright
await page.route('**/api/clusters_mgmt/**', route => route.abort());
await page.setOffline(true);
```

### API Error Simulation
```javascript
// Unit tests - Mock specific error responses
apiRequestMock.post.mockRejectedValue({
  response: { status: 503, data: { code: 'SERVICE_UNAVAILABLE' } }
});

apiRequestMock.post.mockRejectedValue(new Error('Network timeout'));

// Playwright - Intercept and modify responses
await page.route('**/api/clusters_mgmt/v1/clusters/*/node_pools/*/upgrade_policies', route => 
  route.fulfill({ status: 500, body: 'Internal Server Error' })
);
```

### State/Environment Setup
```javascript
// Feature flags
process.env.FEATURE_FLAG_X = 'true'

// User roles/permissions
// Mock specific cluster states
const clusterMock = { ...defaultCluster, state: 'error' };

// Mock data modifications
mockdata/api/clusters_mgmt/v1/clusters.json
```

### Browser Conditions
```bash
# Chrome DevTools
Application → Storage → Clear site data
Application → Service Workers → Bypass for network
Network → Disable cache
```

### Timing/Race Conditions
```javascript
// Delay API responses
apiRequestMock.post.mockImplementation(() => 
  new Promise(resolve => setTimeout(() => resolve('success'), 5000))
);

// Playwright - slow down actions
await page.click('button', { delay: 100 });
```
