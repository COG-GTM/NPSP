# Architecture Waivers — Nonprofit Success Pack (NPSP)

| Field | Value |
|-------|-------|
| **Document Owner** | VA DTC Governance |
| **Package Name** | Nonprofit Success Pack |
| **Namespace** | `npsp` |
| **Repository** | COG-GTM/NPSP |
| **Waiver Authority** | VA DTC Architecture Review Board |

---

## Purpose

This document formally records approved deviations from VA DTC Salesforce Governance standards. NPSP is a Salesforce-managed package with a locked API surface, developed and maintained by Salesforce.org since 2014. Several DTC standards were established after NPSP's architecture was finalized, and NPSP's managed-package constraints make retroactive compliance impractical or impossible without breaking subscriber orgs.

Each waiver includes the rule being waived, justification, and risk mitigation measures in place.

---

## Approved Waivers

### NPSP-W001 — MetadataTriggerHandler Required

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W001 |
| **DTC Rule** | All triggers must use the MetadataTriggerHandler framework |
| **NPSP Pattern** | TDTM (Table-Driven Trigger Management) |
| **Status** | ✅ Approved |

**Justification**
NPSP uses TDTM (Table-Driven Trigger Management), a trigger dispatch framework developed in 2014 that predates the DTC-standard MetadataTriggerHandler. TDTM is functionally equivalent:

- **One trigger per object**: 26 logicless triggers, each delegating to `TDTM_Config_API.run()`
- **Configurable handler registration**: `Trigger_Handler__c` custom setting stores handler class, execution order, trigger actions, and active flag
- **Bypass controls**: Global disable, per-object disable, per-user disable via Custom Setting hierarchy
- **Execution ordering**: `Load_Order__c` field controls handler sequence
- **Activation controls**: `Active__c` checkbox enables/disables handlers without deployment

Migrating to MetadataTriggerHandler would require rewriting 26 triggers plus all handler registration logic, and would break all subscriber orgs that have customized their `Trigger_Handler__c` records.

**Risk Mitigation**
TDTM provides equivalent bypass, ordering, and activation controls to MetadataTriggerHandler. The pattern is well-documented, battle-tested across 40,000+ subscriber orgs, and maintained by the NPSP engineering team.

---

### NPSP-W002 — fflib Prohibited

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W002 |
| **DTC Rule** | fflib (apex-common) library is prohibited |
| **NPSP Pattern** | fflib-based Domain, Selector, Service, and Unit of Work architecture |
| **Status** | ✅ Approved |

**Justification**
NPSP uses fflib (apex-common, apex-mocks, apex-extensions) as its core architectural framework for:

- **Domain layer**: Business logic encapsulation via `fflib_SObjects`
- **Selector layer**: Centralized, type-safe SOQL queries
- **Service layer**: Stateless business operation orchestration
- **Unit of Work**: Transactional DML management across multiple SObjects

The fflib library is located in `force-app/infrastructure/` and is a locked dependency of the managed package. Removing it would require rewriting 50+ classes and fundamentally restructuring the application architecture.

**Risk Mitigation**
fflib is an industry-standard Apex architecture library endorsed by Salesforce (originally authored by Andrew Fawcett, Salesforce Technical Architect). It is security-reviewed, widely adopted across the Salesforce ecosystem, and actively maintained. The library enforces separation of concerns and promotes testable, maintainable code.

---

### NPSP-W003 — System.runAs in All Test Classes

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W003 |
| **DTC Rule** | All test methods must execute within `System.runAs()` to validate permission enforcement |
| **NPSP Pattern** | ~57 of 355+ test classes use `System.runAs()` (~16%) |
| **Status** | ✅ Approved |

**Justification**
Only ~16% of NPSP test classes use `System.runAs()`. Managed package tests often run in system context because:

1. **Subscriber org variability**: Tests must pass regardless of the installing org's permission configuration (profiles, permission sets, sharing rules)
2. **Cross-object operations**: BDI, CRLP, and RD2 tests perform DML across multiple objects that may have varying access levels
3. **Custom Setting access**: Many tests configure `Trigger_Handler__c` and other Custom Settings that require system context
4. **Package installation tests**: `STG_InstallScript` tests must run in system context to validate install/upgrade behavior

Adding `System.runAs()` to all 355+ test classes would risk breaking tests in subscriber orgs with varying permission configurations and would not improve production security posture (runtime sharing is enforced by class-level sharing declarations, not test-level `runAs()`).

**Risk Mitigation**
Production security is enforced through:
- Explicit sharing declarations on all Apex classes (`with sharing`, `without sharing`, `inherited sharing`)
- CRUD/FLS checks in UI-facing controller methods
- The Salesforce platform's built-in sharing model for record-level access

---

### NPSP-W004 — Standard Object Field Limits (<5 per Object)

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W004 |
| **DTC Rule** | Packages should add fewer than 5 custom fields to standard objects |
| **NPSP Pattern** | Account: 21 fields, Contact: 27 fields, Opportunity: 44 fields, Lead: 6 fields |
| **Status** | ✅ Approved |

**Justification**
NPSP adds custom fields to standard objects to deliver core nonprofit CRM functionality:

- **Account (21 fields)**: Household member counts, rollup donation totals, naming fields, system flags
- **Contact (27 fields)**: Primary affiliation, soft credit totals, donor level, deceased/do-not-contact flags, batch processing fields
- **Opportunity (44 fields)**: Primary contact, acknowledgment status, tribute fields, GAU allocations, matching gift fields, Elevate payment references
- **Lead (6 fields)**: Data import mapping fields for BDI processing

These fields are part of the managed package's locked API surface and cannot be removed without breaking subscriber orgs. They are essential to NPSP's core value proposition of transforming standard Salesforce into a nonprofit CRM.

**Risk Mitigation**
- All fields are namespaced (`npsp__`), preventing naming collisions with subscriber org fields
- Fields are documented in NPSP product documentation
- Field additions to the managed package undergo Salesforce security review
- No further standard object field additions are planned

---

### NPSP-W005 — Custom Permissions for Feature Toggling

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W005 |
| **DTC Rule** | Feature toggles must use Custom Permissions |
| **NPSP Pattern** | Hierarchy Custom Settings for per-org/profile/user feature configuration |
| **Status** | ✅ Approved |

**Justification**
NPSP uses 12+ hierarchical Custom Settings for feature configuration (e.g., `npe01__Contacts_And_Orgs_Settings__c`, `Data_Import_Settings__c`, `Gift_Entry_Settings__c`, `Customizable_Rollup_Settings__c`). These Custom Settings predate the Custom Permissions API and provide capabilities that Custom Permissions do not:

- **Per-org defaults with profile/user overrides**: Hierarchical Custom Settings support org → profile → user cascading; Custom Permissions are binary (granted or not)
- **Rich configuration data**: Settings store strings, numbers, booleans, and picklist values; Custom Permissions are boolean-only
- **NPSP Settings UI**: The `STG_*` Visualforce pages provide admin UI for managing these settings; migrating would require rebuilding this UI

Migrating to Custom Permissions would break all subscriber orgs that have customized their NPSP settings.

**Risk Mitigation**
Custom Settings provide equivalent per-org/profile/user configuration with additional flexibility (rich data types, cascading defaults). The NPSP Settings UI provides a governed admin interface for configuration changes.

---

### NPSP-W006 — NebulaLogger Required

| Field | Detail |
|-------|--------|
| **Waiver ID** | NPSP-W006 |
| **DTC Rule** | All applications must use NebulaLogger for error logging and telemetry |
| **NPSP Pattern** | Custom `SfdoInstrumentationService` for Splunk telemetry; `Error__c` custom object for application error logging |
| **Status** | ✅ Approved |

**Justification**
NPSP uses a dual error tracking approach:

1. **`Error__c` custom object**: Stores application errors with stack traces, timestamps, and context. Visible to admins within the NPSP Settings UI.
2. **`SfdoInstrumentationService`**: Sends telemetry data to Splunk for centralized monitoring across all NPSP subscriber orgs. Part of the sfdo-base instrumentation package.

Adding NebulaLogger as an additional dependency would:
- Increase the managed package install footprint and complexity
- Create a redundant logging layer alongside the existing `Error__c` and Splunk infrastructure
- Require subscriber orgs to install and maintain an additional package
- Not provide additional value beyond what the existing instrumentation delivers

**Risk Mitigation**
The existing `Error__c` + `SfdoInstrumentationService` architecture provides:
- Per-org error visibility via `Error__c` records and the NPSP Settings error log
- Centralized cross-org telemetry via Splunk
- Structured error capture with stack traces, error types, and contextual metadata
- Equivalent error tracking and alerting capabilities to NebulaLogger

---

## Waiver Summary

| Waiver ID | Rule | NPSP Alternative | Impact of Compliance | Status |
|-----------|------|-------------------|---------------------|--------|
| NPSP-W001 | MetadataTriggerHandler required | TDTM framework | Rewrite 26 triggers + break subscriber orgs | ✅ Approved |
| NPSP-W002 | fflib prohibited | fflib (apex-common/mocks/extensions) | Rewrite 50+ classes | ✅ Approved |
| NPSP-W003 | System.runAs in all test classes | System-context test execution | Break tests in subscriber orgs | ✅ Approved |
| NPSP-W004 | <5 fields per standard object | 98 fields across 4 standard objects | Cannot remove managed fields | ✅ Approved |
| NPSP-W005 | Custom Permissions for toggles | Hierarchy Custom Settings | Break subscriber org configs | ✅ Approved |
| NPSP-W006 | NebulaLogger required | SfdoInstrumentationService + Error__c | Add redundant package dependency | ✅ Approved |

---

## Review Schedule

These waivers shall be reviewed annually or upon significant changes to DTC governance standards. Review should assess whether any waiver can be retired due to NPSP architecture changes or DTC standard updates.

| Next Review Date | Reviewer |
|-----------------|----------|
| Annual from approval date | VA DTC Architecture Review Board |
