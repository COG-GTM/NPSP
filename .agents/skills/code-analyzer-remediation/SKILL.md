---
name: npsp-code-analyzer-remediation
description: Patterns and procedures for running Salesforce Code Analyzer on NPSP and remediating violations per VA DTC governance standards. Use when fixing Code Analyzer / PMD / ESLint violations, or when working on security, code quality, or test quality improvements.
---

# NPSP Code Analyzer Remediation

## Quick Start

```bash
cd /home/ubuntu/repos/NPSP

# Ensure Code Analyzer is installed
sf plugins install code-analyzer@latest

# Run full scan (takes ~3-5 min)
sf code-analyzer run --workspace . --view detail --output-file sfca_results.json

# Run on specific files (faster)
sf code-analyzer run --workspace . --view detail --include 'force-app/main/default/classes/MY_CLASS.cls'
```

## Parsing Results

```python
import json
from collections import Counter, defaultdict

with open('sfca_results.json') as f:
    data = json.load(f)

violations = data['violations']  # list of dicts
# Each violation has: rule, engine, severity, tags, locations, message, resources
# severity: 1=Critical, 2=High, 3=Moderate, 4=Low, 5=Info

# Get violations for a specific file
def get_file_violations(filepath):
    return [v for v in violations
            if v['locations'][v['primaryLocationIndex']]['file'] == filepath]
```

## CI Gate

The `.github/workflows/code-analyzer.yml` workflow runs on all PRs and **fails if ANY Sev 1 or Sev 2 violations exist in changed files**. When fixing a violation in a file, fix ALL Sev 1+2 violations in that file.

## NPSP-Specific Context

- **Managed package** with `npsp` namespace — class/field API names are locked post-release
- **TDTM framework** (Table-Driven Trigger Management) — NPSP uses its own trigger framework, not the DTC Trigger Action Framework. This requires a DTC Architecture Waiver.
- **fflib** (`force-app/infrastructure/apex-common/`) — third-party library; changes diverge from upstream. Flag for manual review.
- **Test utilities**: `UTIL_UnitTestData_TEST`, `UTIL_UnitTestData_API` — use these for test data creation
- **System mode**: Many NPSP internal operations intentionally use system mode for cross-org compatibility. Document with comment when preserving system mode.

## VA DTC Severity Mapping

| PMD Sev | CodeScan | VA Severity | Blocks PRs? |
|---------|----------|-------------|-------------|
| 1 | Blocker | Critical | YES |
| 2 | Critical | High | YES |
| 3 | Major | Moderate | NO |
| 4 | Minor | Low | NO |
| 5 | Info | Info | NO |

## Priority Order for Remediation

1. **Security Critical**: ApexSOQLInjection, ApexXSSFromURLParam, ApexOpenRedirect, VfUnescapeEl, VfCsrf
2. **Security High**: ApexCRUDViolation, ApexSharingViolations, OperationWithLimitsInLoop, AvoidHardcodingId
3. **API Version**: AvoidOldSalesforceApiVersions (mechanical, safe to batch)
4. **Test Quality**: @isTest annotations (mechanical), RunAs (VA governance critical), assertions
5. **Code Quality**: Braces, empty catch blocks, debug statements, naming
6. **JS/LWC**: ESLint errors, no-var, SLDS tokens
7. **Formatting**: Whitespace, indentation (lowest priority)

## Key Remediation Patterns

### SOQL Injection → Bind Variables
```apex
// BAD
String q = 'SELECT Id FROM Account WHERE Name = \'' + input + '\'';
// GOOD
Map<String, Object> binds = new Map<String, Object>{'val' => input};
Database.queryWithBinds('SELECT Id FROM Account WHERE Name = :val', binds, AccessLevel.USER_MODE);
```

### CRUD/FLS → USER_MODE
```apex
// SOQL
List<Account> accts = [SELECT Id FROM Account WITH USER_MODE];
// DML
Database.insert(records, AccessLevel.USER_MODE);
// Or stripInaccessible
SObjectAccessDecision d = Security.stripInaccessible(AccessType.READABLE, records);
```

### Sharing → Explicit Declaration
```apex
public with sharing class MyClass { }           // default for user-facing
public inherited sharing class MyHandler { }     // for TDTM handlers
private without sharing class Elevated { }       // inner class, limited scope
```

### XSS → Sanitize URL Params
```apex
Id recordId = (Id) ApexPages.currentPage().getParameters().get('id');
// For strings:
String safe = String.escapeSingleQuotes(param);
```

### VF Escaping
```html
<!-- JS context -->
var x = '{!JSENCODE(value)}';
<!-- URL context -->
<a href="{!URLENCODE(url)}">Link</a>
<!-- Remove escape=false unless HTML rendering is required -->
```

### testMethod → @isTest
```apex
// BAD: static testMethod void test() {
// GOOD:
@isTest
static void test() {
```

### API Version Bump
```bash
# Check target version
cat sfdx-project.json | python3 -c "import json,sys; print(json.load(sys.stdin).get('sourceApiVersion'))"
# Bulk update
find force-app/ -name '*-meta.xml' -exec sed -i 's/<apiVersion>[0-9]*\.[0-9]*</<apiVersion>62.0</g' {} \;
```

## PR Batching Guidelines

| Category | Files per PR | Risk Level |
|----------|-------------|------------|
| Security fixes | 5-10 | High — needs careful review |
| API version bumps | 50-100 | Low — mechanical |
| @isTest annotations | 20-30 | Low — mechanical |
| RunAs additions | 5-10 | Medium — may break tests |
| Braces/formatting | 30-50 | Low — mechanical |
| CRUD/FLS fixes | 5-10 | Medium — behavior change |
| Complexity refactoring | 1-3 | High — domain knowledge needed |

## Running Tests

```bash
# LWC unit tests (always run before committing)
yarn test:unit

# ESLint
yarn lint:lwc

# Apex tests (requires connected org)
echo "$SALESFORCE_AUTH_URL" | sf org login sfdx-url -u - -a npsp-org --set-default
sf apex run test --test-level RunLocalTests --wait 15 --target-org npsp-org
```

## Available Playbooks

- `!npsp_security_fix` — Security vulnerability remediation (Critical + High)
- `!npsp_api_version_fix` — API version bumps
- `!npsp_test_quality_fix` — Test quality improvements
- `!npsp_apex_quality_fix` — Apex code quality
- `!npsp_js_quality_fix` — JavaScript/LWC/SLDS quality
- `!npsp_formatting_fix` — Formatting and hygiene
- `!npsp_scan_report` — Scan and progress report
