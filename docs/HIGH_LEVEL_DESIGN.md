# High-Level Design — Nonprofit Success Pack (NPSP)

| Field | Value |
|-------|-------|
| **Document Owner** | VA DTC Governance |
| **Package Name** | Nonprofit Success Pack |
| **Namespace** | `npsp` |
| **API Version** | 53.0 |
| **Repository** | COG-GTM/NPSP |

---

## 1. System Overview

The Nonprofit Success Pack (NPSP) is a Salesforce-managed package that transforms the Salesforce CRM platform into a purpose-built nonprofit constituent relationship management system. NPSP extends standard Salesforce objects (Account, Contact, Opportunity, Lead) with 64 custom objects, custom settings, and custom metadata types to support fundraising, donor management, household management, and program delivery for nonprofit organizations.

NPSP operates under the `npsp` namespace and is distributed as a second-generation managed package via the Salesforce AppExchange. It is installed into subscriber orgs where it runs alongside standard Salesforce functionality and other managed/unmanaged packages.

---

## 2. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SUBSCRIBER SALESFORCE ORG                        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    NPSP Managed Package (npsp)                   │    │
│  │                                                                   │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │    │
│  │  │  UI Layer   │  │  UI Layer   │  │      UI Layer            │ │    │
│  │  │    (LWC)    │  │   (Aura)    │  │   (Visualforce)          │ │    │
│  │  │ 123 comps   │  │  54 comps   │  │    79 pages              │ │    │
│  │  └──────┬──────┘  └──────┬──────┘  └───────────┬──────────────┘ │    │
│  │         │                │                      │                │    │
│  │         └────────────────┼──────────────────────┘                │    │
│  │                          ▼                                       │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │              TDTM — Trigger Dispatch Framework            │   │    │
│  │  │         (26 logicless triggers → handler registry)        │   │    │
│  │  └──────────────────────────┬───────────────────────────────┘   │    │
│  │                             ▼                                    │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │                  Service / Domain Layer                    │   │    │
│  │  │             (fflib patterns — 843 Apex classes)            │   │    │
│  │  │                                                            │   │    │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐  │   │    │
│  │  │  │   BDI    │ │  Gift    │ │   RD2    │ │    CRLP     │  │   │    │
│  │  │  │  Batch   │ │  Entry   │ │Enhanced  │ │Customizable │  │   │    │
│  │  │  │  Data    │ │          │ │Recurring │ │  Rollups    │  │   │    │
│  │  │  │  Import  │ │          │ │Donations │ │             │  │   │    │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └─────────────┘  │   │    │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐  │   │    │
│  │  │  │Engagement│ │  Levels  │ │  Soft    │ │  Household  │  │   │    │
│  │  │  │  Plans   │ │          │ │ Credits  │ │ Management  │  │   │    │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └─────────────┘  │   │    │
│  │  │  ┌──────────┐ ┌──────────────────────────────────────┐   │   │    │
│  │  │  │   GAU    │ │     Address Verification             │   │   │    │
│  │  │  │Allocation│ │  (SmartyStreets / Cicero / Google)   │   │   │    │
│  │  │  └──────────┘ └──────────────────────────────────────┘   │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  │                             │                                    │    │
│  │                             ▼                                    │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │                    Data Layer                              │   │    │
│  │  │    64 Custom Objects / Settings / Custom Metadata Types    │   │    │
│  │  │    + Extensions to Account, Contact, Opportunity, Lead     │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────┐  ┌──────────────────────────────────────┐   │
│  │   Foundation Packages  │  │    sfdo-base Instrumentation         │   │
│  │  npo02 (Households)    │  │  (Splunk telemetry / error tracking) │   │
│  │  npe03 (Recurring Don) │  └──────────────────────────────────────┘   │
│  │  npe4  (Relationships) │                                             │
│  │  npe5  (Affiliations)  │  ┌──────────────────────────────────────┐   │
│  │  npe01 (Contacts&Orgs) │  │    Elevate Payment Platform          │   │
│  └───────────────────────┘  │  (Credit Card / ACH tokenization)     │   │
│                              └──────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Key Subsystems

### 3.1 Batch Data Import (BDI)
Engine that maps flat staging records (`DataImport__c`) into complex object graphs (Account → Contact → Opportunity → Payment → Allocation). Supports dry-run validation, field-level and object-level Custom Metadata mappings, and batch processing through `BDI_DataImport_BATCH`.

### 3.2 Gift Entry (GE)
Modern LWC-based high-volume donation entry UI. Supports batch gift creation, real-time form rendering from `Form_Template__c`, donation matching against existing Opportunities, and Elevate payment integration for credit card/ACH processing.

### 3.3 Enhanced Recurring Donations (RD2)
Modernized recurring donation framework replacing the legacy npe03 model. Supports installment-based scheduling, pause/resume, change logs (`RecurringDonationChangeLog__c`), status mapping via Custom Metadata (`RecurringDonationStatusMapping__mdt`), and Elevate commitment integration.

### 3.4 Customizable Rollups (CRLP)
Configurable rollup calculation engine using Custom Metadata (`Rollup__mdt`, `Filter_Group__mdt`, `Filter_Rule__mdt`). Calculates aggregate donor statistics (total giving, largest gift, average gift, etc.) across Account, Contact, GAU, and Recurring Donation objects. Runs as schedulable batch jobs.

### 3.5 Engagement Plans
Automated task generation framework. `Engagement_Plan_Template__c` defines sequences of `Engagement_Plan_Task__c` records that are materialized as Tasks when an `Engagement_Plan__c` is created for a Contact/Account/Opportunity.

### 3.6 Levels
Tiered donor categorization engine. `Level__c` records define thresholds based on rollup field values (e.g., Total Gifts > $10,000 = "Major Donor"). Evaluated by a schedulable batch that stamps the level on Contact/Account records.

### 3.7 Soft Credits
Attribution of a gift to contacts who are not the primary donor. Managed through `Partial_Soft_Credit__c` and `Account_Soft_Credit__c` objects with rollup calculations into custom fields on Contact and Account.

### 3.8 Household Management
Account-based household model that groups Contacts at a shared address. Automatic household naming, address management (`Address__c`), seasonal address support, and merge handling. Legacy `npo02__Household__c` object maintained for backward compatibility.

### 3.9 GAU Allocations
Fund accounting system. `General_Accounting_Unit__c` represents funds; `Allocation__c` records distribute Opportunity/Payment amounts across GAUs. Supports default allocations and percentage-based splits.

### 3.10 Address Verification
External address validation via SmartyStreets, Cicero, and Google Geocoding APIs. Configurable through `Address_Verification_Settings__c` with batch and single-record verification modes.

---

## 4. Integration Points

| Integration | Direction | Protocol | Purpose |
|-------------|-----------|----------|---------|
| **Elevate Payment Platform** | Bidirectional | REST API (Named Credential) | Credit card/ACH tokenization, payment commitments, refunds |
| **npo02 (Households)** | Dependency | Managed package | Legacy household object, household settings |
| **npe01 (Contacts & Orgs)** | Dependency | Managed package (via npo02) | Payment object (`npe01__OppPayment__c`), Contact/Org settings |
| **npe03 (Recurring Donations)** | Dependency | Managed package | Recurring donation object and settings |
| **npe4 (Relationships)** | Dependency | Managed package | Contact-to-Contact relationships |
| **npe5 (Affiliations)** | Dependency | Managed package | Contact-to-Account affiliations |
| **sfdo-base** | Instrumentation | Apex callouts | Splunk telemetry via `SfdoInstrumentationService` |
| **Address Verification APIs** | Outbound | REST | SmartyStreets, Cicero, Google Geocoding |

---

## 5. Data Model Summary

### Standard Object Extensions

| Object | Custom Fields Added | Key Fields |
|--------|-------------------|------------|
| **Account** | 21 | `npsp__Number_of_Household_Members__c`, rollup totals, household naming fields |
| **Contact** | 27 | `npsp__Primary_Affiliation__c`, soft credit totals, donor level fields |
| **Opportunity** | 44 | `npsp__Primary_Contact__c`, GAU allocation, acknowledgment, tribute fields |
| **Lead** | 6 | Data import mapping fields |

### Key Custom Objects

| Object | Purpose |
|--------|---------|
| `DataImport__c` | Staging object for BDI batch import records |
| `DataImportBatch__c` | Batch container for Gift Entry and BDI processing |
| `Address__c` | Verified/seasonal addresses linked to Accounts |
| `Allocation__c` | GAU fund allocation on Opportunities/Payments |
| `General_Accounting_Unit__c` | Fund/program accounting categories |
| `npe03__Recurring_Donation__c` | Recurring gift schedules (enhanced via RD2) |
| `RecurringDonationSchedule__c` | Installment schedule detail for RD2 |
| `Engagement_Plan_Template__c` | Task automation templates |
| `Engagement_Plan__c` | Active engagement plan instances |
| `Level__c` | Donor tier definitions |
| `Partial_Soft_Credit__c` | Attribution records for shared gifts |
| `Account_Soft_Credit__c` | Organization-level soft credit attribution |
| `Error__c` | Application error log records |
| `Form_Template__c` | Gift Entry form layout definitions |
| `Batch__c` | Legacy batch data entry containers |

### Custom Metadata Types

| Type | Purpose |
|------|---------|
| `Rollup__mdt` | CRLP rollup definitions |
| `Filter_Group__mdt` / `Filter_Rule__mdt` | CRLP filter criteria |
| `Data_Import_Field_Mapping__mdt` | BDI field-level mappings |
| `Data_Import_Object_Mapping__mdt` | BDI object-level mappings |
| `RecurringDonationStatusMapping__mdt` | RD2 status-to-state mappings |
| `Opportunity_Stage_To_State_Mapping__mdt` | Stage mapping for rollups |

---

## 6. Security Model

### Sharing Approach
- NPSP classes use a mix of `with sharing`, `without sharing`, and `inherited sharing` declarations depending on the context.
- Trigger handlers and batch classes typically use `without sharing` to ensure data processing completes regardless of the running user's record access.
- Controller classes serving UI components generally use `with sharing` to enforce record-level security.

### CRUD/FLS Enforcement
- `@AuraEnabled` controller methods follow standard Apex security patterns.
- BDI and batch processing operate in system context to handle cross-object DML that spans permission boundaries.
- Custom Settings (`Trigger_Handler__c`, `npe01__Contacts_And_Orgs_Settings__c`, etc.) are accessed in system context as they are org configuration, not user data.

### User vs. System Context
- UI-facing operations (Gift Entry, CRLP configuration, Engagement Plan creation) run in user context.
- Background processing (BDI batches, CRLP rollup calculations, RD2 installment creation, scheduled jobs) runs in system context to ensure completeness.

---

## 7. Technology Stack

| Layer | Technology | Details |
|-------|-----------|---------|
| **Backend** | Apex | 843 classes, 259K+ lines of code |
| **UI — Modern** | Lightning Web Components (LWC) | 123 components (Gift Entry, Donation History, CRLP config) |
| **UI — Legacy** | Aura Components | 54 components (BDI mapping, BGE, CRLP containers) |
| **UI — Classic** | Visualforce Pages | 79 pages (batch entry, address management, settings) |
| **Trigger Framework** | TDTM (Table-Driven Trigger Management) | 26 logicless triggers with configurable handler registration |
| **Architecture Patterns** | fflib (apex-common, apex-mocks, apex-extensions) | Domain, Selector, Service, Unit of Work |
| **Build & CI** | CumulusCI, GitHub Actions | Orchestration, packaging, Jest tests, Code Analyzer |
| **Browser Testing** | Robot Framework | Automated UI testing via CumulusCI |
| **Unit Testing (JS)** | Jest (lwc-jest) | LWC component unit tests |
| **Package Management** | Yarn | JavaScript dependency management |
| **Instrumentation** | sfdo-base / SfdoInstrumentationService | Splunk telemetry and error tracking |
