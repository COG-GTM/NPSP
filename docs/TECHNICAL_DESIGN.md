# Technical Design Document — Nonprofit Success Pack (NPSP)

| Field | Value |
|-------|-------|
| **Document Owner** | VA DTC Governance |
| **Package Name** | Nonprofit Success Pack |
| **Namespace** | `npsp` |
| **API Version** | 53.0 |
| **Repository** | COG-GTM/NPSP |

---

## 1. Trigger Architecture — TDTM (Table-Driven Trigger Management)

### Overview
NPSP uses TDTM (Table-Driven Trigger Management) as its trigger dispatch framework. TDTM predates the DTC-standard MetadataTriggerHandler and provides equivalent functionality: one logicless trigger per object, configurable handler registration, execution ordering, and activation controls.

### How It Works

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   SObject     │     │  Logicless       │     │   TDTM_Config_API   │
│   DML Event   │────▶│  Trigger         │────▶│   .run()            │
│               │     │  (26 triggers)   │     │                     │
└──────────────┘     └──────────────────┘     └──────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌─────────────────────┐
                                              │ Trigger_Handler__c  │
                                              │ Custom Setting       │
                                              │ (handler registry)   │
                                              └──────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌─────────────────────┐
                                              │  Handler Classes     │
                                              │  (e.g., ACCT_*,     │
                                              │   CON_*, OPP_*, etc.)│
                                              └─────────────────────┘
```

### Trigger Pattern
Every trigger in NPSP follows the same logicless pattern:

```apex
trigger TDTM_Account on Account (after delete, after insert, after undelete,
    after update, before delete, before insert, before update) {

    TDTM_Config_API.run(Trigger.isBefore, Trigger.isAfter, Trigger.isInsert,
        Trigger.isUpdate, Trigger.isDelete, Trigger.isUnDelete,
        Trigger.new, Trigger.old, Schema.Sobjecttype.Account);
}
```

### Handler Registration
Handlers are registered in the `Trigger_Handler__c` hierarchical custom setting. Each record specifies:

| Field | Purpose |
|-------|---------|
| `Object__c` | SObject API name the handler fires on |
| `Class__c` | Fully-qualified Apex class name |
| `Load_Order__c` | Execution sequence (lower = earlier) |
| `Trigger_Action__c` | Semicolon-delimited list: `BeforeInsert;AfterInsert;BeforeUpdate;AfterUpdate;BeforeDelete;AfterDelete;AfterUndelete` |
| `Active__c` | Enable/disable without code deployment |
| `Asynchronous__c` | Run handler in a future/queueable context |
| `Filter_Field__c` / `Filter_Value__c` | Field-level filtering before handler executes |

### Covered Objects (26 Triggers)
`Account`, `Address__c`, `Affiliation__c`, `Allocation__c`, `Campaign`, `CampaignMember`, `Contact`, `DataImport__c`, `DataImportBatch__c`, `EngagementPlan__c`, `EngagementPlanTask__c`, `FormTemplate__c`, `GeneralAccountingUnit__c`, `GrantDeadline__c`, `HouseholdObject__c`, `Lead`, `Level__c`, `Opportunity`, `OpportunityContactRole__c`, `AccountSoftCredit__c`, `PartialSoftCredit__c`, `Payment__c`, `RecurringDonation__c`, `Relationship__c`, `Task`, `User`

### TDTM Controls
- **Global bypass**: `TDTM_Config_API.disableAllTriggers()` / `enableAllTriggers()`
- **Per-object bypass**: Deactivate specific handler records
- **Per-user bypass**: Custom Setting hierarchy support
- **Recursion guard**: Built into the TDTM dispatcher

---

## 2. Service Layer Architecture

### fflib Patterns
NPSP uses the fflib library (apex-common, apex-mocks, apex-extensions) located in `force-app/infrastructure/`. The architecture follows four core patterns:

#### Domain Layer
Encapsulates business logic for SObject collections. Domain classes extend `fflib_SObjects` or implement domain-specific interfaces.

```
force-app/infrastructure/apex-common/
force-app/infrastructure/apex-mocks/
force-app/infrastructure/apex-extensions/
```

#### Selector Layer
Centralizes SOQL queries. Selector classes provide type-safe query methods and enforce field-level access patterns.

Example naming convention:
- `AccountSelector.cls`
- `ContactSelector.cls`
- `UserSelector.cls`

#### Service Layer
Orchestrates business operations spanning multiple domains. Service classes are stateless and coordinate between selectors, domains, and external integrations.

#### Unit of Work
`UnitOfWork.cls` manages transactional DML, allowing multiple insert/update/delete operations to be committed atomically. Used heavily in BDI processing and Gift Entry.

### Class Naming Conventions

| Prefix/Suffix | Purpose | Example |
|---------------|---------|---------|
| `*_TDTM` | Trigger handler | `ACCT_AccountMerge_TDTM` |
| `*_BATCH` | Batch Apex class | `BDI_DataImport_BATCH` |
| `*_SCHED` | Schedulable class | `ADDR_Seasonal_SCHED` |
| `*_CTRL` | Controller (VF/Aura) | `ACCT_ViewOverride_CTRL` |
| `*_TEST` | Test class | `ACCT_AccountMerge_TEST` |
| `*_SVC` / `*Service` | Service class | `SfdoInstrumentationService` |
| `UTIL_*` | Utility class | `UTIL_UnitTestData_TEST` |
| `STG_*` | Settings/configuration | `STG_InstallScript` |
| `ERR_*` | Error handling | `ERR_RecordLog` |

### Key Architectural Boundaries

```
┌──────────────────────────────────────────────────┐
│                  UI Controllers                   │
│         (LWC @AuraEnabled, Aura, VF)             │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│               Service Layer                       │
│    (Stateless orchestration, transaction mgmt)    │
└──────────────────────┬───────────────────────────┘
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
┌──────────────┐ ┌──────────┐ ┌─────────────┐
│   Domain     │ │ Selector │ │ Unit of     │
│   Layer      │ │  Layer   │ │ Work        │
│ (Business    │ │ (SOQL)   │ │ (DML)       │
│  Logic)      │ │          │ │             │
└──────────────┘ └──────────┘ └─────────────┘
```

---

## 3. Batch Processing

### BDI Engine (Batch Data Import)
The BDI engine processes `DataImport__c` records in configurable batch sizes:

1. **Staging**: Records loaded into `DataImport__c` (via API, CSV, or Gift Entry UI)
2. **Dry Run**: Optional validation pass that checks for errors without committing DML
3. **Mapping**: Custom Metadata (`Data_Import_Field_Mapping__mdt`, `Data_Import_Object_Mapping__mdt`) drives field-to-field and object-to-object mapping
4. **Processing**: `BDI_DataImport_BATCH` iterates records, creates/matches Accounts, Contacts, Opportunities, Payments, and Allocations
5. **Error Handling**: Per-record error capture in `DataImport__c.Status__c` and `Error__c`

### CRLP Rollup Batches
Customizable Rollups run as scheduled batch jobs:

| Batch Class | Target Object |
|-------------|--------------|
| `CRLP_Account_BATCH` | Account rollups |
| `CRLP_Contact_BATCH` | Contact rollups |
| `CRLP_GAU_BATCH` | GAU rollups |
| `CRLP_RD_BATCH` | Recurring Donation rollups |

Rollup definitions are stored in `Rollup__mdt` with filter criteria in `Filter_Group__mdt` / `Filter_Rule__mdt`.

### Schedulable Jobs

| Job | Purpose |
|-----|---------|
| `ADDR_Seasonal_SCHED` | Swap seasonal addresses on schedule |
| `ADDR_Validator_Batch` | Batch address verification |
| Level evaluation batch | Recalculate donor tier assignments |
| RD2 installment creation | Generate upcoming installment Opportunities |

---

## 4. UI Architecture

### Lightning Web Components (LWC) — 123 Components
Modern UI layer for Gift Entry, Donation History, CRLP configuration, and settings. Located in `force-app/main/default/lwc/`.

Key component families:
- `ge*` — Gift Entry (form renderer, batch wizard, field mapping, Elevate integration)
- `rd2*` — Enhanced Recurring Donations (entry form, status, schedule visualization)
- `crlp*` — Customizable Rollup configuration UI
- `bdi*` — BDI field/object mapping configuration
- `util*` — Shared utility components (page header, illustrations, modals)

### Aura Components — 54 Components
Legacy interactive UI layer. Located in `force-app/main/default/aura/`.

Key component families:
- `BGE_*` — Batch Gift Entry (legacy)
- `CRLP_*` — Rollup configuration (wrapper for LWC)
- `BDI_*` — Data Import mapping management
- `HH_*` — Household address management
- `GE_*` — Gift Entry containers

### Visualforce Pages — 79 Pages
Classic UI for administrative functions and legacy workflows. Located in `force-app/main/default/pages/`.

Key pages:
- `STG_*` — NPSP Settings panels
- `BDI_*` / `BDE_*` — Batch data entry/import
- `ALLO_*` — Allocation management
- `ADDR_*` — Address management utilities
- `PMT_*` — Payment management

### UI Technology Decision Matrix

| Scenario | Technology | Rationale |
|----------|-----------|-----------|
| New feature development | LWC | Modern framework, best performance |
| Existing interactive features | Aura | Maintained but not extended |
| Settings pages, admin tools | Visualforce | Deep integration with Custom Settings |
| Record overrides | Visualforce | Standard controller support |

---

## 5. Testing Strategy

### Apex Testing
- **843 Apex classes** with **355+ test classes** (`*_TEST` suffix)
- Test data factory: `UTIL_UnitTestData_TEST` provides centralized test record creation
- Test classes located in both `force-app/main/default/classes/` and `force-app/test/`
- Test coverage target: ≥ 75% per class

#### Test Class Conventions
```apex
@IsTest
private class FEATURE_Component_TEST {

    @TestSetup
    static void createTestData() {
        // Use UTIL_UnitTestData_TEST for standard test records
        UTIL_UnitTestData_TEST.createMultipleTestContacts(5);
    }

    @IsTest
    static void shouldDoExpectedBehavior() {
        // Arrange
        // Act
        Test.startTest();
        // ... execute logic
        Test.stopTest();
        // Assert
        System.assertEquals(expected, actual, 'Assertion message');
    }
}
```

### LWC Jest Testing
- Jest tests co-located with LWC components (`__tests__/` directories)
- Run via `yarn test:unit`
- Mock data patterns for Apex wire adapters and imperative calls

### Robot Framework (Browser Testing)
- Located in `robot/` directory
- Automated UI tests via CumulusCI
- Page object pattern abstracting Salesforce UI elements
- Requires connected Salesforce org

### CI Pipeline

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `jest.yml` | PR | LWC Jest unit tests |
| `code-analyzer.yml` | PR | Salesforce Code Analyzer (Sev 1/2 gate) |
| `codeowners.yml` | PR | CODEOWNERS validation |
| `compliance.yml` | PR | Compliance checks |

---

## 6. Namespace Considerations

### Managed Package Context (`npsp__`)
When NPSP is installed as a managed package in subscriber orgs:
- All custom objects, fields, classes, and components are prefixed with `npsp__`
- API names become `npsp__DataImport__c`, `npsp__BDI_DataImport_BATCH`, etc.
- Foundation package fields use their own namespaces: `npe01__`, `npo02__`, `npe03__`, `npe4__`, `npe5__`

### Unmanaged Development Context
During development (unpackaged org):
- Namespace prefix is not applied
- CumulusCI handles namespace injection/stripping via `namespace_inject` and `namespace_tokenize` task options
- Token `%%%NAMESPACE%%%` is used in metadata templates for namespace-aware deployment

### Namespace-Aware Patterns in Code
```apex
// Dynamic namespace detection
String nsPrefix = UTIL_Namespace.getNamespace();

// Namespace-safe field references
String fieldName = UTIL_Namespace.StrTokenNSPrefix('npsp__') + 'Amount__c';

// Schema-based references (namespace-agnostic)
Schema.SObjectField amountField = Opportunity.npsp__Amount__c;
```

### Cross-Package References
NPSP references foundation package objects using their namespaces:

| Package | Namespace | Key Objects |
|---------|-----------|-------------|
| Households | `npo02` | `npo02__Household__c`, `npo02__Households_Settings__c` |
| Contacts & Orgs | `npe01` | `npe01__OppPayment__c`, `npe01__Contacts_And_Orgs_Settings__c` |
| Recurring Donations | `npe03` | `npe03__Recurring_Donation__c`, `npe03__Recurring_Donations_Settings__c` |
| Relationships | `npe4` | `npe4__Relationship__c`, `npe4__Relationship_Settings__c` |
| Affiliations | `npe5` | `npe5__Affiliation__c` |

---

## 7. Configuration Architecture

### Custom Settings (Hierarchical)
NPSP uses hierarchical Custom Settings for feature configuration, predating the Custom Permissions API:

| Setting | Purpose |
|---------|---------|
| `npe01__Contacts_And_Orgs_Settings__c` | Contact/Account model settings |
| `npo02__Households_Settings__c` | Household naming and behavior |
| `npe03__Recurring_Donations_Settings__c` | Recurring donation configuration |
| `npe4__Relationship_Settings__c` | Auto-relationship creation rules |
| `Allocations_Settings__c` | Default GAU allocation behavior |
| `Customizable_Rollup_Settings__c` | CRLP engine settings |
| `Data_Import_Settings__c` | BDI processing configuration |
| `Gift_Entry_Settings__c` | Gift Entry feature flags |
| `Error_Settings__c` | Error logging preferences |
| `Household_Naming_Settings__c` | Household name/greeting formats |
| `Package_Settings__c` | Package-wide feature flags |
| `Trigger_Handler__c` | TDTM handler registration |

### Custom Metadata Types
Used for subscriber-deployable configuration:
- `Rollup__mdt` — CRLP rollup definitions
- `Filter_Group__mdt` / `Filter_Rule__mdt` — CRLP filter criteria
- `Data_Import_Field_Mapping__mdt` / `Data_Import_Object_Mapping__mdt` — BDI mappings
- `Data_Import_Field_Mapping_Set__mdt` / `Data_Import_Object_Mapping_Set__mdt` — BDI mapping sets
- `RecurringDonationStatusMapping__mdt` — RD2 status configuration
- `Opportunity_Stage_To_State_Mapping__mdt` — Opportunity rollup state mapping
- `GetStartedChecklistItem__mdt` / `GetStartedChecklistSection__mdt` — Setup assistant
- `Custom_Notification__mdt` — Platform notification definitions

---

## 8. Error Handling

### Error Framework
- `Error__c` custom object stores application errors with stack traces
- `ERR_Handler` provides centralized error capture and logging
- `SfdoInstrumentationService` sends telemetry to Splunk for monitoring
- `@AuraEnabled` methods use try-catch blocks with `AuraHandledException` for UI error surfacing

### Error Flow
```
Exception thrown
       │
       ▼
ERR_Handler.processError()
       │
       ├──▶ Create Error__c record
       │
       ├──▶ SfdoInstrumentationService.log() → Splunk
       │
       └──▶ (If UI context) throw AuraHandledException
```

---

## 9. Build and Deployment

### CumulusCI Orchestration
`cumulusci.yml` defines the complete build, test, and deployment pipeline:

- **Minimum CCI version**: 3.74.0
- **Source format**: SFDX
- **Install script**: `STG_InstallScript` (post-install configuration)
- **Uninstall script**: `STG_UninstallScript` (cleanup on removal)

### Dependency Chain
```
npe01 (Contacts & Orgs)
  └── npo02 (Households) ──includes npe01
npe03 (Recurring Donations)
npe4  (Relationships)
npe5  (Affiliations)
  └── NPSP ──depends on all above
```

### Key CCI Tasks (Custom)
| Task | Description |
|------|-------------|
| `deploy_dev_config` | Post-install config for dev orgs |
| `deploy_trial_config` | Trial org setup with namespace injection |
| `deploy_rd2_config` | Enable Enhanced Recurring Donations |
| `deploy_reports` | Install reports and dashboards |
| `deploy_gift_entry_unmanaged` | Gift Entry metadata for dev |
| `ensure_record_type` | Create default Opportunity record type |
