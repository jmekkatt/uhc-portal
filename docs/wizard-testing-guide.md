# Wizard Testing Guide (OSD & ROSA)

A guide for **how to approach testing** create-cluster wizard changes in OCM UI — for OSD and ROSA (Classic and Hosted). This is not a wizard functionality walkthrough. It focuses on **what testing options exist**, **what to plan for**, and **where regressions tend to spread**.

---

## Table of Contents

1. [Why Wizard Testing Is Different](#why-wizard-testing-is-different)
2. [Product and Variant Coverage](#product-and-variant-coverage)
3. [Testing Options](#testing-options)
4. [Checklist: Testing a New Wizard Feature](#checklist-testing-a-new-wizard-feature)
5. [Dependencies to Plan For](#dependencies-to-plan-for)
6. [Regression Impact and Dependents](#regression-impact-and-dependents)
7. [Common Testing Pitfalls](#common-testing-pitfalls)

---

## Why Wizard Testing Is Different

Wizard flows are not isolated screens. When you test a wizard — or any new feature inside one — keep these constraints in mind:

- **Sequential journey** — Steps build on earlier choices. A change on one screen can change what appears, what is valid, or what is enabled on a later screen.
- **Shared building blocks** — Many fields and sections are reused across OSD and ROSA. A change in a shared area can regress multiple products at once.
- **Conditional behaviour** — Visibility, validation, and navigation often depend on earlier selections (cloud provider, control plane type, region, networking mode, and so on).
- **Async gating** — Next may stay disabled while credentials are verified, roles or VPCs are loaded, billing status is resolved, or **quota is checked**. Tests must account for loading and failure states, not only the happy path.
- **Review as a dependent** — New or changed fields should be checked on the input step **and** on the review summary before submission.
- **Downstream lifecycle** — Some wizard settings also matter after creation (machine pools, channels, networking). A wizard change may affect cluster details or Day 2 flows even if the wizard itself still works.

Treat the wizard as one connected flow, not a set of independent pages.

---

## Product and Variant Coverage

A single UI change may need re-testing across more than one product path. Use this to decide **how wide** your test scope should be.

| Area | Variants to consider |
| ---- | -------------------- |
| **OSD** | AWS CCS, AWS non-CCS, GCP CCS, GCP non-CCS, Trial, Marketplace, Curated |
| **ROSA** | Classic (standalone control plane), Hosted/HCP (hosted control plane) |
| **Cloud** | AWS vs GCP — different credentials, auth models, and networking behaviour |
| **Auth model** | CCS keys, WIF, service account (GCP), IAM roles and OIDC (ROSA Hosted) |

**Shared vs product-specific**

| Change type | Typical regression scope |
| ----------- | ------------------------ |
| Shared field or section (version, channel, machine pool, networking, review) | OSD **and** ROSA; often multiple cloud variants |
| Product-only screen (OSD billing model, ROSA accounts and roles) | That product only, but check parallel screens on the other product for parity if the pattern is similar |
| Cloud-only behaviour | All flows using that cloud provider |
| Footer / step navigation | All wizard flows using that footer |

If you are unsure, start with the product variant you changed, then add **at least one OSD and one ROSA** path when the change touches shared UI.

---

## Testing Options

Pick the lightest option that gives enough confidence. You do not need full cluster creation for every change.

### 1. Targeted validation (wizard walkthrough, no cluster create)

**Best for:** New or changed fields, validation messages, conditional show/hide, Next/Back behaviour, review summary updates.

**Covers:**

- Invalid and valid input on the affected step
- Error display and clearing
- Whether Next/Back/Cancel behave correctly
- Whether the review step reflects the new or changed value
- Whether earlier choices still produce the right later-step behaviour

**Does not cover:** Whether the cluster is actually provisioned with that setting.

**Note:** Wizard validation is usually sequential — reach the affected step in order, then leave the wizard in a valid state for any follow-on cases.

### 2. Focused feature testing

**Best for:** One capability that spans multiple steps or needs a specific configuration (billing contracts, log forwarding, Y-stream channel, advanced networking, machine pool edge cases).

**Covers:** Deep behaviour in one area without re-running the entire wizard validation suite.

**Consider when:**

- The feature is too large to fold into an existing validation suite
- You need a specific infra or account setup that other tests do not use
- You are verifying Day 2 impact from a wizard choice on an existing cluster

### 3. Full creation path

**Best for:** Changes that affect submission, payload assembly, API integration, or end-to-end provisioning.

**Covers:** Complete wizard → review → create → cluster reaches expected post-create state.

**Requires:** Live credentials, quota, and often pre-provisioned cloud resources.

**Use sparingly** for small UI-only changes; use when the backend or cluster outcome is in scope.

### 4. Component-level testing

**Best for:** Pure validation rules, formatting, and render logic in isolation.

**Does not replace** wizard testing when behaviour depends on step order, API-populated dropdowns, or cross-step conditions.

### Choosing the right option

| If your change… | Start with | Also consider |
| --------------- | ---------- | ------------- |
| Adds or changes a validator or error message | Component + targeted validation | Review step |
| Adds a new optional field | Targeted validation | Review + one creation path if it is submitted |
| Changes shared version/channel/region UI | Targeted validation on OSD and ROSA | Existing validation suites for those products |
| Changes footer or Next enablement | Targeted validation + one creation flow | All products using that footer |
| Changes what is sent on create | Full creation path | Day 2 check on cluster details if applicable |
| Changes networking or VPC selection | Focused feature or creation | Infra-dependent dropdown behaviour |
| Touches billing or quota display | Focused feature | May need mocked or special account state |

---

## Checklist: Testing a New Wizard Feature

Use this when adding or changing wizard functionality.

### Scope

- [ ] Which wizard step(s) are affected?
- [ ] Is the UI **shared** (OSD + ROSA) or **product-specific**?
- [ ] Which **product variants** apply (AWS/GCP, CCS/non-CCS, Classic/Hosted)?
- [ ] Does the change depend on **earlier wizard choices** (region, provider, control plane type)?
- [ ] Does the value appear on the **review** step?
- [ ] Does it affect **cluster creation payload** or only display/validation?
- [ ] Are there **post-create** flows that read the same setting (cluster details, machine pools, upgrades)?

### Behaviour to verify

- [ ] Field/section visible only under the correct conditions
- [ ] Required vs optional behaviour
- [ ] Invalid input → correct error; valid input → error clears
- [ ] Next disabled/enabled at the right times (including during loading)
- [ ] Back preserves entered data; Cancel exits as expected
- [ ] Review shows the correct value
- [ ] Create/submit blocked when the wizard is invalid

### Negative and edge cases

- [ ] Empty state (no API data: no versions, no VPCs, no roles)
- [ ] Loading and slow API responses
- [ ] Switching an upstream choice that should reset or hide the feature
- [ ] Missing credentials or infra — document skip vs fail behaviour
- [ ] **Quota state** — org has quota for the product/path under test; note if option visibility or defaults depend on quota

### Regression on dependents

- [ ] Other steps that use the same shared component
- [ ] Other product variant using the same shared component
- [ ] Review and submission for flows you did not directly change
- [ ] Adjacent non-wizard pages if the same component or data is reused

### What to document for others

- Product variants in scope
- Required accounts, credentials, and pre-provisioned infra
- **Quota requirements** (which product quotas must be present for options to appear or for submit to succeed)
- Scenarios skipped when dependencies are unavailable
- Whether creation or only validation is required to sign off

---

## Dependencies to Plan For

Missing dependencies often look like flaky tests or empty dropdowns. Plan these before you run or sign off wizard testing.

### Environment and access

- Staging org-admin user with **quota** on the organisation (see [Quota and wizard behaviour](#quota-and-wizard-behaviour) below — quota is not only a gate for creation; it shapes what the wizard shows and defaults to)
- Correct **cloud accounts** linked for the variant under test
- **Billing account** setup for Hosted ROSA contract scenarios (often needs more than one account)
- **KMS keys**, **OIDC config**, or **GCP WIF/SA** when encryption or GCP auth is in scope

### Quota and wizard behaviour

Quota is loaded for the organisation when the wizard opens and can **change wizard flow, defaults, and what is enabled** — not just whether cluster creation ultimately succeeds. Account for quota when scoping tests and interpreting results.

| Area | How quota can influence the wizard |
| ---- | ---------------------------------- |
| **OSD billing model (first step)** | Which subscription and infrastructure options are shown or hidden (e.g. trial, standard OSD, marketplace). Default billing model and BYOC vs Red Hat cloud account selection when the user has quota for only one path. |
| **ROSA control plane (Hosted vs Classic)** | Hosted option disabled when the org lacks hosted-product quota; flow may default to Classic or block that choice. |
| **ROSA billing accounts** | List of selectable billing accounts and contract/billing warnings come from quota data. Contracted vs non-contracted behaviour affects warnings and confirmation on Next. |
| **Cluster details** | Quota can affect available options or limits on details (e.g. region or product-related constraints tied to quota). |
| **Machine pool / node count** | Node count limits and validation may be tied to remaining compute quota for the org. |
| **Cloud provider step (OSD GCP)** | Footer may block advancing when required GCP resource quota is not available. |
| **Review and submit** | Insufficient quota at submit time can surface errors even when earlier steps appeared valid. |

**Testing implications:**

- The **same wizard UI** can look different for two testers if their orgs have different quota — do not assume staging behaviour matches your local org without checking quota state.
- Scenarios that need a specific option visible (trial, Hosted, BYOC, marketplace) require an org (or simulated quota response) with that quota granted.
- Billing-contract tests on ROSA Hosted often need **controlled quota/billing state** — staging accounts may not have the right contract mix; tests may skip or use simulated API responses.
- When a option is missing or Next is blocked, confirm **quota** before treating it as a UI defect.
- Quota-related changes regress **billing model, control plane, billing account, node limits, and submit** — widen regression scope beyond the step you changed.

### Pre-provisioned infrastructure

Many networking and creation scenarios need resources that already exist in the cloud account:

- AWS: VPCs, subnets, security groups
- GCP: VPCs, subnets, PSC, shared VPC
- ROSA: IAM roles, installer roles, OIDC for Hosted

If infra is missing, tests may skip, fail at selection controls, or pass only the validation path that does not need live resources.

### Live backend data

Wizard screens depend on staging APIs for versions, channels, linked accounts, roles, VPCs, quota, and billing. Staging data **changes over time** (new versions, renamed channels). Prefer structural checks (options exist, format is valid) over hard-coding specific version strings when possible.

### Ordering within a suite

Wizard cases in one suite usually assume the flow is already on the correct step. A failure early in the suite often causes later cases to fail for unrelated reasons — always triage the **first** failure.

### API state simulation

Some scenarios (billing contract, quota) are hard to reproduce with default staging accounts. Testing may rely on temporarily simulating API responses. Any simulated state must be reset so later cases are not polluted.

---

## Regression Impact and Dependents

### High-impact shared areas

Changes here commonly affect **both OSD and ROSA** and multiple wizard steps:

| Shared area | Dependents to re-test |
| ----------- | --------------------- |
| Cluster details (version, channel, region, encryption) | Details step, review, any step that depends on version/region |
| Machine pool | Machine pool step, review, Day 2 machine pool management |
| Networking (AZ, ingress, CIDR) | Networking substeps, review, VPC step when applicable |
| Upgrade policy | Cluster updates step, review, channel Day 1/Day 2 flows |
| VPC selection | Networking/VPC steps, creation flows using existing VPC |
| Shared form controls | Every step using those inputs |
| Review screen | All flows — summary must stay accurate |

### Navigation and footer

Footer logic decides when Next is enabled. Dependents include:

- Every step that calls async APIs before advance (credential check, role list, VPC load)
- Confirmation dialogs (e.g. billing on Hosted ROSA)
- Silent failures where the wizard appears valid but does not advance

After footer changes, run **validation and at least one creation path** per affected product.

### Product parity

OSD and ROSA often wrap the same shared fields in separate screens. A fix or regression in one product’s details screen may need a **parity check** on the other even if you only changed one file.

### Downstream outside the wizard

| Wizard area | Possible dependents outside wizard |
| ----------- | ----------------------------------- |
| Machine pool defaults | Cluster details machine pools tab |
| Channel / version | Upgrade settings, Day 2 channel change specs |
| Networking / VPC | Cluster networking tab, delete flows |
| Log forwarding | Cluster details log forwarding |
| Billing account | Subscriptions, quota views |

When scoping regression, ask: **who else reads or displays this value after create?**

### Regression scope workflow

1. Map the change to the wizard step(s) and whether the UI is shared or specific.
2. List **downstream steps** in the same wizard (review, later conditional steps).
3. List **other product variants** using the same shared piece.
4. List **post-create** surfaces that use the same setting.
5. Run targeted validation for each affected variant; add creation only when submission or API integration changed.

---

## Common Testing Pitfalls

### Popovers block the next action

Validation errors can open inline help popovers that block the next field. Clear or dismiss them before continuing.

### Upstream choices reset downstream state

Changing version, region, provider, or control plane type often clears or replaces later options. Re-establish prerequisites before testing the feature under change.

### Similar controls, different behaviour

Version and channel are a common example: both look like dropdowns but depend on each other and may use different interaction patterns. Do not assume one test approach fits all.

### Async loading mistaken for failure

Credential verification, role lists, and VPC lists can take a long time. Distinguish **still loading** from **failed to load** before reporting a defect.

### Staging drift

Tests tied to exact version numbers or instance types break when staging is refreshed. Prefer assertions on presence and shape of data where possible.

### Missing infra reported as UI bugs

Empty VPC or role lists are often an environment or account issue, not a UI defect. Confirm dependencies before escalating.

### Quota mistaken for UI bugs

A disabled Hosted tile, hidden trial option, empty billing account list, or blocked Next on the cloud provider step can be **quota-driven**. Verify organisation quota (and billing contract state for Hosted ROSA) before logging a wizard UI defect.

### Cascading failures in sequential suites

Fix the first failing step; later failures are often a consequence of being on the wrong screen.

### Review-only testing is insufficient

A field can look correct on input but be missing or wrong on review. Always include review when the feature is part of the create payload.

### Narrow sign-off

Passing one product variant (e.g. ROSA Classic validation only) is not enough when the change touched **shared** wizard UI — widen scope to OSD or ROSA Hosted as applicable.
