# Security & Permissions Guide

## Sharing Model

### Record-Level Security Hierarchy
1. **Organization-Wide Defaults (OWD)** — Most restrictive baseline
2. **Role Hierarchy** — Opens access upward
3. **Sharing Rules** — Opens access to specific groups/roles
4. **Manual Sharing** — Ad-hoc record sharing
5. **Apex Managed Sharing** — Programmatic sharing

### OWD Settings
- **Private**: Only owner + users above in hierarchy
- **Public Read Only**: Everyone can read, only owner can edit
- **Public Read/Write**: Everyone can read and edit
- **Controlled by Parent**: Detail objects inherit from master

### Apex Sharing
```apex
// with sharing — enforces record-level security (default for most code)
public with sharing class AccountService { }

// without sharing — bypasses record-level security (use sparingly)
public without sharing class DataMigrationService { }

// inherited sharing — inherits from calling class (good for utility classes)
public inherited sharing class QueryHelper { }
```

**Rule**: Always use `with sharing` unless you have a specific reason not to.

## Permission Model

### Hierarchy (most specific wins)
Profile → Permission Set → Permission Set Group → Muting Permission Set

### Best Practice: Minimum-Access Profile + Permission Sets
1. Create a minimal base profile (or use "Minimum Access - Salesforce")
2. Add capabilities via Permission Sets
3. Group Permission Sets into Permission Set Groups
4. Use Muting Permission Sets to remove specific permissions from groups

### Permission Sets via CLI
```bash
# Deploy permission set
sf project deploy start --metadata PermissionSet:My_Permission_Set

# Assign to user
sf org assign permset --name My_Permission_Set --target-org myorg
sf org assign permset --name My_Permission_Set --on-behalf-of user@example.com
```

## CRUD/FLS Enforcement in Code

### SOQL with USER_MODE (API 48.0+, preferred)
```apex
// Enforces both CRUD and FLS
List<Account> accounts = [SELECT Id, Name, Phone FROM Account WITH USER_MODE];

// DML with user mode
Database.insert(records, AccessLevel.USER_MODE);
Database.update(records, AccessLevel.USER_MODE);
Database.delete(records, AccessLevel.USER_MODE);
```

### stripInaccessible (older approach)
```apex
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, Secret_Field__c FROM Account]
);
// Secret_Field__c stripped if user lacks FLS
List<Account> safe = (List<Account>) decision.getRecords();
```

### Schema Describe Checks (manual)
```apex
if (Schema.sObjectType.Account.isAccessible() &&
    Schema.sObjectType.Account.fields.Name.isAccessible()) {
    // Safe to query
}
```

## External Client Apps (Spring '26)

Replaces Connected Apps for new integrations. Migration required for existing apps.

### Key Changes
- Default: Connected App creation now disabled (must explicitly enable)
- New integrations should use External Client App
- Existing Connected Apps continue to work
- ExternalClientApp is a new metadata type for deployment

## Encryption & Data Protection

- Use **Shield Platform Encryption** for data at rest
- Use **Named Credentials** for external auth (never hardcode credentials)
- Use **Custom Metadata Types** for configuration (deployable, not in data)
- **Session ID in outbound messages** retired Feb 16, 2026
- Use My Domain URLs (instance-based URLs no longer redirect in Spring '26)
