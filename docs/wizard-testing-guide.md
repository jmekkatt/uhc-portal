# Wizard Testing Guide (OSD & ROSA)

A guide for **how to approach testing** create-cluster wizard changes in OCM UI — for OSD and ROSA (Classic and Hosted). This is not a wizard functionality walkthrough. It focuses on **what testing options exist**, **what to plan for**, and **where regressions tend to spread**.

---

## Table of Contents

1. [Why Wizard Testing Is Different](#why-wizard-testing-is-different)
2. [Product and Variant Coverage](#product-and-variant-coverage)
3. [Testing Options](#testing-options)
4. [Checklist: Testing a New Wizard Feature](#checklist-testing-a-new-wizard-feature)
5. [Regression Impact and Dependents](#regression-impact-and-dependents)

---

## Why Wizard Testing Is Different

Wizard flows are not isolated screens. When you test a wizard — or any new feature inside one — keep these constraints in mind:

- **Sequential journey** — Steps build on earlier choices. A change on one screen can change what appears, what is valid, or what is enabled on a later screen.
- **Shared building blocks** — Many fields and sections are reused across OSD and ROSA. A change in a shared area can regress multiple products at once.
- **Conditional behaviour** — Visibility, validation, and navigation often depend on earlier selections (cloud provider, control plane type, region, networking mode, and so on).
- **Async gating** — Next may stay disabled while credentials are verified, roles or VPCs are loaded, billing status is resolved, or **quota is checked**. Tests must account for loading and failure states, not only the happy path.
- **Review as a dependent** — New or changed fields should be checked on the input step **and** on the review summary before submission.
- **Downstream lifecycle** — Most wizard settings also matter after creation (machine pools, channels, networking). A wizard change may affect cluster details or Day 2 flows even if the wizard itself still works.

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

Most changes need **more than one** of these options, but rarely just one. Pick the combination that fits your change.

The majority of wizard tests also require live credentials, quota, and often pre-provisioned cloud resources.

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

**Covers:** Deep behaviour in one capability or configuration area.

**Consider when:**

- The feature spans multiple wizard steps or needs a specific setup
- You need a specific infra or account setups
- You are verifying Day 2 impact from a wizard choice on an existing cluster

### 3. Full creation path

**Best for:** Changes that affect submission, payload assembly, API integration, or end-to-end provisioning.

**Covers:** Complete wizard → review → create → cluster reaches expected post-create state.

**Consider when:**

- Submission, payload assembly, API integration, or end-to-end provisioning is in scope
- You need to confirm the cluster reaches expected post-create state
- Day 2 or cluster details need verification after create

**Note:** Requires live credentials, quota, and pre-provisioned cloud resources. The majority of wizard tests need these.

### 4. Component-level testing

**Best for:** Validators, formatters, and component render behaviour tested in isolation.

**Covers:** Whether a rule, format, or UI element works on its own — without walking the full wizard.

**Consider when:**

- You add or change a validation rule or formatter
- The change lives in a single component with little wizard coupling

**Note:** Pair with targeted validation when behaviour depends on step order, API-populated dropdowns, or earlier wizard choices.

**Example:** A new cluster name restriction —  run targeted validation to confirm the error shows on the cluster details step, clears when fixed, and blocks Next until the wizard is valid.

### 5. Usability testing

**Best for:** Layout, copy, labelling, help text, and whether users can follow the wizard without confusion.

**Covers:** Whether the flow is understandable and easy to complete — labels, grouping, progressive disclosure, error message clarity, and whether the review step reads clearly before submit.

**Consider when:**

- You change wizard copy, step titles, field labels, or help text
- You add, remove, or reorder sections that affect how users scan a step
- You redesign a step or introduce a new pattern users have not seen elsewhere in the wizard

**Example:** A networking step redesign — confirm users can tell public vs private cluster options apart, find advanced settings without getting lost, and understand what the review summary means before clicking Create.


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
