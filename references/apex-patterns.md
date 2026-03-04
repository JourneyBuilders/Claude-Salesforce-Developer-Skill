# Apex Best Practices & Patterns

## Table of Contents
1. [Governor Limits Quick Reference](#governor-limits-quick-reference)
2. [Trigger Framework](#trigger-framework)
3. [Bulkification Patterns](#bulkification-patterns)
4. [Async Apex](#async-apex)
5. [Error Handling](#error-handling)
6. [Testing Patterns](#testing-patterns)
7. [Security Patterns](#security-patterns)
8. [Modern Apex (API 66.0)](#modern-apex-api-660)
9. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Governor Limits Quick Reference

| Limit | Synchronous | Asynchronous |
|-------|-------------|--------------|
| SOQL queries | 100 | 200 |
| SOQL rows retrieved | 50,000 | 50,000 |
| DML statements | 150 | 150 |
| DML rows | 10,000 | 10,000 |
| CPU time | 10,000 ms | 60,000 ms |
| Heap size | 6 MB | 12 MB |
| Callouts | 100 | 100 |
| Callout timeout total | 120 s | 120 s |
| Future calls | 50 | 0 (in future) |
| Queueable jobs | 50 | 1 |
| SOSL queries | 20 | 20 |
| Email invocations | 10 | 10 |

Use `Limits` class methods to monitor consumption at runtime:
```apex
System.debug('SOQL queries: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
System.debug('DML statements: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
System.debug('CPU time: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime());
```

---

## Trigger Framework

**Rule: One trigger per object, all logic in handler/service classes.**

### Recommended: TriggerHandler Base Class Pattern

```apex
public virtual class TriggerHandler {
    @TestVisible private static Boolean isBypassed = false;

    public void run() {
        if (isBypassed) return;

        switch on Trigger.operationType {
            when BEFORE_INSERT  { beforeInsert(); }
            when BEFORE_UPDATE  { beforeUpdate(); }
            when BEFORE_DELETE  { beforeDelete(); }
            when AFTER_INSERT   { afterInsert(); }
            when AFTER_UPDATE   { afterUpdate(); }
            when AFTER_DELETE   { afterDelete(); }
            when AFTER_UNDELETE { afterUndelete(); }
        }
    }

    protected virtual void beforeInsert()  {}
    protected virtual void beforeUpdate()  {}
    protected virtual void beforeDelete()  {}
    protected virtual void afterInsert()   {}
    protected virtual void afterUpdate()   {}
    protected virtual void afterDelete()   {}
    protected virtual void afterUndelete() {}

    public static void bypass()  { isBypassed = true; }
    public static void reset()   { isBypassed = false; }
}
```

### Implementation
```apex
// OpportunityTrigger.trigger
trigger OpportunityTrigger on Opportunity (
    before insert, before update,
    after insert, after update
) {
    new OpportunityTriggerHandler().run();
}

// OpportunityTriggerHandler.cls
public class OpportunityTriggerHandler extends TriggerHandler {

    protected override void beforeInsert() {
        OpportunityService.setDefaults((List<Opportunity>) Trigger.new);
    }

    protected override void afterUpdate() {
        OpportunityService.handleStageChanges(
            (List<Opportunity>) Trigger.new,
            (Map<Id, Opportunity>) Trigger.oldMap
        );
    }
}

// OpportunityService.cls — business logic here
public class OpportunityService {

    public static void setDefaults(List<Opportunity> opps) {
        for (Opportunity opp : opps) {
            if (opp.StageName == null) {
                opp.StageName = 'Prospecting';
            }
        }
    }

    public static void handleStageChanges(
        List<Opportunity> newOpps, Map<Id, Opportunity> oldMap
    ) {
        List<Task> tasksToCreate = new List<Task>();
        for (Opportunity opp : newOpps) {
            Opportunity old = oldMap.get(opp.Id);
            if (opp.StageName != old.StageName && opp.StageName == 'Closed Won') {
                tasksToCreate.add(new Task(
                    WhatId = opp.Id,
                    Subject = 'Follow up on won deal',
                    OwnerId = opp.OwnerId,
                    ActivityDate = Date.today().addDays(3)
                ));
            }
        }
        if (!tasksToCreate.isEmpty()) {
            insert tasksToCreate;
        }
    }
}
```

### Recursion Prevention
```apex
public class RecursionGuard {
    private static Set<Id> processedIds = new Set<Id>();

    public static Boolean isFirstRun(Id recordId) {
        if (processedIds.contains(recordId)) return false;
        processedIds.add(recordId);
        return true;
    }

    @TestVisible
    private static void reset() {
        processedIds.clear();
    }
}
```

---

## Bulkification Patterns

### Pattern 1: Collect → Query → Process → DML
```apex
public static void updateContactAddresses(List<Contact> contacts) {
    // 1. COLLECT: Gather related IDs
    Set<Id> accountIds = new Set<Id>();
    for (Contact c : contacts) {
        if (c.AccountId != null) {
            accountIds.add(c.AccountId);
        }
    }

    // 2. QUERY: Single bulk query
    Map<Id, Account> accountMap = new Map<Id, Account>(
        [SELECT Id, BillingStreet, BillingCity, BillingState, BillingPostalCode
         FROM Account WHERE Id IN :accountIds]
    );

    // 3. PROCESS: Build update list
    List<Contact> toUpdate = new List<Contact>();
    for (Contact c : contacts) {
        Account a = accountMap.get(c.AccountId);
        if (a != null) {
            c.MailingStreet = a.BillingStreet;
            c.MailingCity = a.BillingCity;
            toUpdate.add(c);
        }
    }

    // 4. DML: Single bulk operation (only in after triggers or service methods)
    // In before triggers, field changes on Trigger.new are auto-saved
}
```

### Pattern 2: Map-Based Lookups
```apex
// Use Maps for O(1) lookups instead of nested loops or repeated queries
Map<String, Account> accountByExtId = new Map<String, Account>();
for (Account a : [SELECT Id, External_Id__c FROM Account
                   WHERE External_Id__c IN :externalIds]) {
    accountByExtId.put(a.External_Id__c, a);
}
```

### Pattern 3: Partial DML Success
```apex
Database.SaveResult[] results = Database.insert(records, false); // allOrNone = false
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error err : results[i].getErrors()) {
            System.debug('Error on record ' + records[i].Id + ': ' + err.getMessage());
        }
    }
}
```

---

## Async Apex

### Decision Framework
| Pattern | Use When | Limits |
|---------|----------|--------|
| **Queueable** (preferred) | Async processing, chaining, complex types | 50 per txn, chain 1 |
| **Batch** | >50k records, long-running | 5 concurrent, 250k `start()` |
| **@future** (legacy) | Simple async, primitive types only | 50 per txn |
| **Schedulable** | Recurring jobs | 100 scheduled |
| **Platform Events** | Decoupled, cross-boundary | 150 published per txn |

### Queueable (Preferred over @future)
```apex
public class AccountProcessingJob implements Queueable, Database.AllowsCallouts {
    private List<Id> accountIds;

    public AccountProcessingJob(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
        // Process accounts...
        // Chain another job if needed:
        if (!remainingIds.isEmpty()) {
            System.enqueueJob(new AccountProcessingJob(remainingIds));
        }
    }
}

// Enqueue
System.enqueueJob(new AccountProcessingJob(accountIdList));
```

### Batch Apex
```apex
public class AccountCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {
    private Integer recordsProcessed = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator(
            'SELECT Id, Name FROM Account WHERE LastModifiedDate < LAST_N_YEARS:2'
        );
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        // Process batch chunk (default 200 records)
        for (Account a : scope) {
            a.Description = 'Archived';
        }
        update scope;
        recordsProcessed += scope.size();
    }

    public void finish(Database.BatchableContext bc) {
        System.debug('Processed ' + recordsProcessed + ' records');
    }
}

// Execute: Database.executeBatch(new AccountCleanupBatch(), 200);
```

### Apex Cursors (New in Spring '26 / API 66.0)
```apex
// Create a cursor for large datasets (up to 50M rows)
Cursor cursor = Database.getCursor(
    'SELECT Id, Name FROM Account ORDER BY Name'
);
// Fetch in pages
List<Account> page1 = (List<Account>) cursor.fetch(0, 200);
List<Account> page2 = (List<Account>) cursor.fetch(200, 200);
// PaginationCursor for UI-friendly paging
```

---

## Error Handling

### Custom Exception Pattern
```apex
public class ApplicationException extends Exception {}
public class ValidationException extends ApplicationException {}
public class IntegrationException extends ApplicationException {}

// Usage
public static void processOrder(Order__c order) {
    if (order.Amount__c <= 0) {
        throw new ValidationException('Order amount must be positive');
    }
    try {
        ExternalService.submit(order);
    } catch (CalloutException e) {
        throw new IntegrationException('Failed to submit order: ' + e.getMessage());
    }
}
```

### AddError for Trigger Validation
```apex
// In before triggers — shows error on the record's page
for (Account a : Trigger.new) {
    if (String.isBlank(a.Industry)) {
        a.Industry.addError('Industry is required for all accounts');
    }
}
```

---

## Testing Patterns

### Key Principles
- **Always test bulk** (200+ records to catch non-bulkified code)
- **Use `@TestSetup`** for shared test data (runs once per class, rolled back per method)
- **Assert outcomes**, not just coverage — use `Assert.areEqual()`, `Assert.isTrue()`
- **Use `Test.startTest()` / `Test.stopTest()`** to reset governor limits for the code under test
- **Never rely on org data** — use `@IsTest(SeeAllData=false)` (default)
- **Test negative paths** — verify exceptions, validations, edge cases

### Modern Assert Class (API 59.0+, preferred over System.assert)
```apex
Assert.areEqual(expected, actual, 'message');
Assert.areNotEqual(unexpected, actual, 'message');
Assert.isTrue(condition, 'message');
Assert.isFalse(condition, 'message');
Assert.isNull(value, 'message');
Assert.isNotNull(value, 'message');
Assert.isInstanceOfType(obj, Type.class, 'message');
Assert.fail('Should not reach here');
```

### Testing Async Apex
```apex
@IsTest
static void testQueueable() {
    Account a = new Account(Name = 'Test');
    insert a;

    Test.startTest();
    System.enqueueJob(new MyQueueable(new List<Id>{a.Id}));
    Test.stopTest(); // Forces async execution to complete

    // Assert results after async completes
    Account updated = [SELECT Name FROM Account WHERE Id = :a.Id];
    Assert.areEqual('Processed', updated.Description);
}
```

### Testing Callouts (HttpCalloutMock)
```apex
@IsTest
private class MyCalloutTest {
    private class MockHttpResponse implements HttpCalloutMock {
        public HttpResponse respond(HttpRequest req) {
            HttpResponse res = new HttpResponse();
            res.setHeader('Content-Type', 'application/json');
            res.setBody('{"status": "success"}');
            res.setStatusCode(200);
            return res;
        }
    }

    @IsTest
    static void testCallout() {
        Test.setMock(HttpCalloutMock.class, new MockHttpResponse());
        Test.startTest();
        String result = MyService.callExternalApi();
        Test.stopTest();
        Assert.areEqual('success', result);
    }
}
```

### @isTest Annotation Enhancements (API 66.0)
```apex
@IsTest(testFor=AccountService.class)  // Declare which class this tests
private class AccountServiceTest { ... }

@IsTest(critical=true)  // Always run during RunRelevantTests
private class CriticalBusinessTest { ... }
```

---

## Security Patterns

### CRUD/FLS Enforcement
```apex
// API 66.0+ preferred: WITH USER_MODE in SOQL
List<Account> accounts = [SELECT Id, Name FROM Account WITH USER_MODE];

// For DML
Database.insert(records, AccessLevel.USER_MODE);
Database.update(records, AccessLevel.USER_MODE);

// Older approach: Security.stripInaccessible
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, Phone FROM Account]
);
List<Account> safeAccounts = (List<Account>) decision.getRecords();
```

### @AuraEnabled Security
```apex
public with sharing class AccountController {
    @AuraEnabled(cacheable=true)
    public static List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account WITH USER_MODE LIMIT 50];
    }

    @AuraEnabled
    public static void updateAccount(Account acc) {
        // Validate input
        if (acc == null || acc.Id == null) {
            throw new AuraHandledException('Invalid account');
        }
        Database.update(acc, AccessLevel.USER_MODE);
    }
}
```

---

## Modern Apex (API 66.0)

### Null-Safe Operator (?.)
```apex
String city = account?.BillingAddress?.getCity();
Integer count = myList?.size();
```

### Safe Navigation in SOQL
```apex
// Null-safe field access in results
for (Contact c : [SELECT Id, Account.Name FROM Contact]) {
    String accountName = c.Account?.Name ?? 'No Account';
}
```

### Switch Statements
```apex
switch on opp.StageName {
    when 'Prospecting', 'Qualification' { /* early stage */ }
    when 'Closed Won'   { /* won */ }
    when 'Closed Lost'  { /* lost */ }
    when else           { /* other */ }
}
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| SOQL in a loop | Hits 100 query limit | Query before loop, use Map |
| DML in a loop | Hits 150 DML limit | Collect in List, DML once |
| Hardcoded IDs | Breaks across orgs | Use Custom Metadata/Labels |
| `@future` for complex work | No chaining, primitives only | Use Queueable |
| `with sharing` missing | Security bypass | Always declare sharing |
| No test assertions | Coverage without validation | Assert.areEqual() |
| `SeeAllData=true` | Tests depend on org data | Create test data |
| Logic in triggers | Untestable, no reuse | Use handler/service classes |
| SELECT * (all fields) | Heap/performance issues | Select only needed fields |
| Recursive triggers without guard | Infinite loops | Use static flag/Set |
