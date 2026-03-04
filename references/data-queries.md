# SOQL, SOSL & Data Operations Guide

## Table of Contents
1. [SOQL Fundamentals](#soql-fundamentals)
2. [Query Optimization](#query-optimization)
3. [Relationship Queries](#relationship-queries)
4. [Aggregate Queries](#aggregate-queries)
5. [SOSL Search](#sosl-search)
6. [Bulk API 2.0](#bulk-api-20)
7. [Named Query API (Spring '26)](#named-query-api-spring-26)

---

## SOQL Fundamentals

### Basic Patterns
```sql
-- Standard query
SELECT Id, Name, Industry FROM Account WHERE Industry = 'Technology' LIMIT 100

-- Date literals
SELECT Id FROM Opportunity WHERE CloseDate = THIS_QUARTER
SELECT Id FROM Case WHERE CreatedDate = LAST_N_DAYS:30

-- NULL checks
SELECT Id, Name FROM Contact WHERE Email != null

-- IN clause with bind variables (Apex)
SELECT Id, Name FROM Account WHERE Id IN :accountIds

-- LIKE for pattern matching
SELECT Id, Name FROM Account WHERE Name LIKE 'Acme%'

-- ORDER BY with NULLS handling
SELECT Id, Name FROM Account ORDER BY Name ASC NULLS LAST LIMIT 50

-- OFFSET for pagination
SELECT Id, Name FROM Account ORDER BY Name LIMIT 10 OFFSET 20
```

### User Mode Queries (API 48.0+, strongly recommended)
```apex
// Enforces CRUD/FLS — preferred over manual checks
List<Account> accounts = [SELECT Id, Name FROM Account WITH USER_MODE];

// System mode (only when needed for backend processing)
List<Account> accounts = [SELECT Id, Name FROM Account WITH SYSTEM_MODE];
```

### Dynamic SOQL (use carefully)
```apex
String query = 'SELECT Id, Name FROM Account';
if (String.isNotBlank(industry)) {
    query += ' WHERE Industry = :industry';  // Bind variable — safe from injection
}
List<Account> accounts = Database.query(query);

// With User Mode
List<Account> accounts = Database.query(query, AccessLevel.USER_MODE);
```

**WARNING**: Never concatenate user input directly into SOQL strings. Always use bind variables.

---

## Query Optimization

### Selective Queries (Important for Large Data Volumes)
A query is selective if it returns less than ~10% of total records for the object. The query
optimizer uses indexes to speed up selective queries.

**Standard indexed fields**: Id, Name, OwnerId, CreatedDate, SystemModstamp, RecordType,
lookup/master-detail fields, Email (on Contact/Lead).

**Custom indexed fields**: Mark as External ID or Unique, or request via Salesforce Support.

### Optimization Tips
```sql
-- GOOD: Uses indexed field in WHERE
SELECT Id FROM Account WHERE Id IN :accountIds

-- GOOD: Specific filter + limit
SELECT Id, Name FROM Contact WHERE AccountId = :accId AND Email != null LIMIT 100

-- BAD: Leading wildcard (no index)
SELECT Id FROM Account WHERE Name LIKE '%corp'

-- BAD: Negative filter (not selective)
SELECT Id FROM Account WHERE Industry != 'Technology'

-- BAD: Formula field in WHERE (not indexed)
SELECT Id FROM Account WHERE Formula_Field__c = 'value'
```

### Query Plan Tool
In Developer Console: Query Editor → Check "Use Tooling API" → enter your query →
click "Query Plan" to see cost and selectivity.

### Reduce Query Rows
```apex
// Only select fields you need
// BAD: SELECT * equivalent
// GOOD: Select specific fields
List<Account> accounts = [SELECT Id, Name FROM Account LIMIT 200];

// Use FOR UPDATE only when needed (locks records)
Account a = [SELECT Id, Name FROM Account WHERE Id = :accId FOR UPDATE];
```

---

## Relationship Queries

### Parent-to-Child (Subquery)
```sql
SELECT Id, Name,
    (SELECT Id, LastName, Email FROM Contacts WHERE Email != null)
FROM Account
WHERE Industry = 'Technology'
```
```apex
for (Account a : [SELECT Id, (SELECT LastName FROM Contacts) FROM Account]) {
    for (Contact c : a.Contacts) {
        System.debug(c.LastName);
    }
}
```

### Child-to-Parent (Dot Notation)
```sql
SELECT Id, LastName, Account.Name, Account.Industry
FROM Contact
WHERE Account.Industry = 'Technology'
```

### Polymorphic Relationships (What/Who)
```sql
SELECT Id, Subject,
    TYPEOF What
        WHEN Account THEN Name, Industry
        WHEN Opportunity THEN Name, StageName
    END
FROM Task
```

---

## Aggregate Queries

```sql
-- COUNT
SELECT COUNT() FROM Account WHERE Industry = 'Technology'
SELECT COUNT(Id) cnt FROM Account GROUP BY Industry

-- SUM, AVG, MIN, MAX
SELECT Industry, SUM(AnnualRevenue) totalRev, COUNT(Id) cnt
FROM Account
GROUP BY Industry
HAVING SUM(AnnualRevenue) > 1000000
ORDER BY SUM(AnnualRevenue) DESC

-- GROUP BY ROLLUP
SELECT Industry, COUNT(Id) cnt
FROM Account
GROUP BY ROLLUP(Industry)
```

```apex
// In Apex
List<AggregateResult> results = [
    SELECT AccountId, SUM(Amount) totalAmount
    FROM Opportunity
    WHERE IsClosed = true
    GROUP BY AccountId
];
for (AggregateResult ar : results) {
    Id accountId = (Id) ar.get('AccountId');
    Decimal total = (Decimal) ar.get('totalAmount');
}
```

---

## SOSL Search

```apex
// Basic search
List<List<SObject>> results = [FIND 'Acme*' IN ALL FIELDS
    RETURNING Account(Id, Name), Contact(Id, LastName, Email)];
List<Account> accounts = (List<Account>) results[0];
List<Contact> contacts = (List<Contact>) results[1];

// With filters
List<List<SObject>> results = [FIND 'Cloud Computing' IN ALL FIELDS
    RETURNING Account(Id, Name WHERE Industry = 'Technology' LIMIT 10),
              Contact(Id, LastName WHERE Department = 'Engineering')];
```

### When to Use SOSL vs SOQL
| SOSL | SOQL |
|------|------|
| Text search across multiple objects | Exact field matching |
| Fuzzy/keyword search | Precise filtering, joins |
| Returns results from many objects at once | One object (with relationships) |
| Limit: 2,000 results per object | Limit: 50,000 rows per txn |
| 20 SOSL queries per txn | 100 SOQL queries per txn |

---

## Bulk API 2.0

### Via SF CLI
```bash
# Bulk query
sf data query --query "SELECT Id, Name FROM Account" \
  --target-org myorg --bulk --wait 10

# Bulk upsert from CSV
sf data upsert bulk --sobject Account \
  --file data/accounts.csv \
  --external-id External_Id__c \
  --target-org myorg --wait 10

# Bulk delete
sf data delete bulk --sobject Account \
  --file data/ids-to-delete.csv \
  --target-org myorg --wait 10
```

### Via Apex (REST)
```apex
// Prefer Data Loader, SF CLI, or Bulk API 2.0 REST endpoints for large operations
// In Apex, use Database.insert/update with batch apex for large volumes
```

### Best Practices for Large Data Operations
- Use Bulk API 2.0 for >2,000 records (auto-batches)
- Use Data Loader or SF CLI for one-time data loads
- Use Batch Apex for recurring scheduled operations
- Use `Database.Stateful` in Batch to track state across chunks
- Set appropriate batch size (default 200, reduce for complex processing)

---

## Named Query API (Spring '26)

Define reusable SOQL queries as REST endpoints. Created in Setup → Named Queries.

```
// Endpoint format
GET /services/data/v66.0/named-queries/<QueryApiName>?param1=value1

// Example: Named query "ActiveAccountsByIndustry" with parameter :industry
GET /services/data/v66.0/named-queries/ActiveAccountsByIndustry?industry=Technology
```

Benefits: Reusable, governed, can be used as Agentforce actions (beta),
eliminates need for custom Apex REST endpoints for simple queries.
