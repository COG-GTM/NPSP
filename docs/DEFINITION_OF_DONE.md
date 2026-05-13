# Definition of Done — Nonprofit Success Pack (NPSP)

| Field | Value |
|-------|-------|
| **Document Owner** | VA DTC Governance |
| **Package Name** | Nonprofit Success Pack |
| **Namespace** | `npsp` |
| **Repository** | COG-GTM/NPSP |

---

## Purpose

This document defines the minimum quality criteria that must be satisfied before any code change is considered complete and ready for merge into the `main` branch. All items apply to **changed files** in the pull request unless otherwise noted.

---

## Checklist

### Static Analysis
- [ ] Code passes Salesforce Code Analyzer with **0 Severity 1 or Severity 2 violations** in changed files
- [ ] ESLint passes on changed LWC JavaScript files (`yarn lint:lwc`)
- [ ] No `console.*` calls in committed JavaScript files
- [ ] Python tasks pass `flake8` linting (`flake8 tasks/`)

### Security
- [ ] No SOQL injection vulnerabilities — all dynamic queries use bind variables or `String.escapeSingleQuotes()`
- [ ] No hardcoded Salesforce IDs or credentials in source code
- [ ] No secrets, API keys, or passwords committed to the repository
- [ ] Sensitive data is not exposed in debug logs or error messages

### Apex Quality
- [ ] All Apex classes have explicit sharing declarations (`with sharing`, `without sharing`, or `inherited sharing`)
- [ ] `@AuraEnabled` methods include try-catch error handling with `AuraHandledException`
- [ ] Apex test coverage ≥ **75%** for all changed classes
- [ ] Test methods use descriptive names following the `shouldDoExpectedBehavior` pattern
- [ ] Test data created via `UTIL_UnitTestData_TEST` or `@TestSetup` methods — no reliance on org data
- [ ] Bulk-safe patterns used (no SOQL/DML inside loops, collections processed in bulk)
- [ ] Governor limit-conscious design (aggregate queries, batch processing for large data sets)

### LWC Quality
- [ ] All LWC Jest tests pass (`yarn test:unit`)
- [ ] New LWC components include corresponding `__tests__/` Jest test files
- [ ] No direct DOM manipulation — use reactive properties and template directives
- [ ] Accessibility: components follow SLDS patterns and include appropriate ARIA attributes

### Testing
- [ ] All existing tests continue to pass (no regressions)
- [ ] New functionality includes corresponding test coverage
- [ ] Edge cases and error paths are tested
- [ ] Test classes follow the `*_TEST` naming convention

### Code Review
- [ ] PR description follows the repository template
- [ ] PR description clearly explains the **what** and **why** of the change
- [ ] At least one peer review completed and approved
- [ ] All review comments addressed or resolved

### Documentation
- [ ] Public-facing API changes documented
- [ ] Complex business logic includes inline documentation
- [ ] Custom Metadata or Custom Setting changes documented in PR description

### CI Pipeline
- [ ] All GitHub Actions workflows pass:
  - `jest.yml` — LWC unit tests
  - `code-analyzer.yml` — Salesforce Code Analyzer gate
  - `codeowners.yml` — CODEOWNERS validation
  - `compliance.yml` — Compliance checks
- [ ] No new CI failures introduced by the change

---

## Exemptions

Changes that are documentation-only, configuration-only (Custom Metadata/Labels), or cosmetic (whitespace, formatting) may skip the Apex/LWC testing requirements with reviewer approval noted in the PR.

For managed package constraints and DTC rule waivers, see [Architecture Waivers](ARCHITECTURE_WAIVERS.md).
