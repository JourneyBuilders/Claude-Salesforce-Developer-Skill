# Debugging in Salesforce

## Debug Logs

### Setting Up Trace Flags

Trace flags tell Salesforce to capture debug logs for a specific entity. Without an active
trace flag, no logs are generated.

**Setup UI:**
1. Setup → Debug Logs
2. Click **New** under User Trace Flags
3. Set **Traced Entity Type**: User, Apex Class, Apex Trigger, Automated Process, or
   Platform Integration
4. Set **Traced Entity Name**: the specific user/class/trigger
5. Set **Start Date** and **Expiration Date** (max 24 hours)
6. Choose or create a **Debug Level**
7. Save

**SF CLI:**
```bash
# Turn on debug logging for replay debugger (auto trace flag, 30 min)
sf apex tail log                              # stream logs in terminal
sf apex tail log --color                      # colorized output
sf apex tail log --color --debug-level MyLevel # custom debug level

# Get existing logs
sf apex log list                              # list available logs
sf apex log get --log-id <logId>              # download specific log
sf apex log get --number 5                    # download last 5 logs
```

### Debug Levels and Categories

A Debug Level defines what gets logged and at what granularity. Levels are cumulative —
higher levels include everything from lower levels.

**Log Levels** (lowest to highest):
NONE → ERROR → WARN → INFO → DEBUG → FINE → FINER → FINEST

**Log Categories:**

| Category | What It Logs | Recommended Level |
|----------|-------------|-------------------|
| **Apex Code** | Apex execution, System.debug output, variable assignments | FINEST for debugging, DEBUG for monitoring |
| **Apex Profiling** | Cumulative limits, method timing, SOQL/DML profiling | INFO (FINEST generates huge logs) |
| **Database** | SOQL queries, DML operations, query plans | INFO or FINE |
| **Callout** | HTTP/SOAP callouts, request/response details | INFO or FINE |
| **Validation** | Validation rules, workflow field updates | INFO |
| **Workflow** | Flows, Process Builder, workflow rules | FINE for flow debugging |
| **Visualforce** | VF page rendering, view state | INFO |
| **System** | System methods, constructor calls | DEBUG |
| **Wave** | CRM Analytics | INFO |
| **NBA** | Next Best Action | INFO |

**Recommended debug level presets:**

```
# Apex-focused debugging (most common)
Apex Code=FINEST, Apex Profiling=INFO, Database=FINE,
Callout=INFO, Validation=INFO, Workflow=INFO, System=DEBUG

# Flow-focused debugging
Apex Code=DEBUG, Apex Profiling=NONE, Database=INFO,
Callout=INFO, Validation=INFO, Workflow=FINER, System=INFO

# Integration/callout debugging
Apex Code=DEBUG, Apex Profiling=INFO, Database=INFO,
Callout=FINE, Validation=NONE, Workflow=NONE, System=DEBUG
```

### Reading Debug Logs

**Log structure:**
```
Line 1: Header — API version + categories with levels
  66.0 APEX_CODE,FINEST;APEX_PROFILING,INFO;DB,FINE;...
Line 2: USER_INFO — user details, timezone
Line 3+: Execution events (timestamp | event type | details)
Last line: EXECUTION_FINISHED
```

**Key event types to search for:**

| Event | What It Shows |
|-------|--------------|
| `USER_DEBUG` | Your System.debug() statements |
| `SOQL_EXECUTE_BEGIN / _END` | SOQL queries with row counts |
| `DML_BEGIN / DML_END` | DML operations with row counts |
| `FATAL_ERROR` | Unhandled exceptions (search this first!) |
| `EXCEPTION_THROWN` | Caught and uncaught exceptions |
| `FLOW_START_INTERVIEW / _END` | Flow execution start/end |
| `FLOW_ELEMENT_BEGIN / _END` | Individual flow element execution |
| `FLOW_ELEMENT_FAULT` | Flow element that threw an error |
| `CALLOUT_REQUEST / _RESPONSE` | HTTP callout details |
| `LIMIT_USAGE_FOR_NS` | Governor limit consumption summary |
| `CODE_UNIT_STARTED / _FINISHED` | Trigger/class execution boundaries |
| `VALIDATION_RULE` | Validation rule evaluation |
| `VALIDATION_FAIL` | Validation rule failure |
| `WF_CRITERIA_BEGIN / _END` | Workflow/Flow criteria evaluation |

**Quick search patterns in logs:**
```bash
# Find errors fast
grep -i "FATAL_ERROR\|EXCEPTION" debug.log

# Find SOQL queries
grep "SOQL_EXECUTE" debug.log

# Find DML operations
grep "DML_BEGIN" debug.log

# Find your debug statements
grep "USER_DEBUG" debug.log

# Find governor limit summary
grep "LIMIT_USAGE_FOR_NS" debug.log

# Find flow errors
grep "FLOW_ELEMENT_FAULT\|FLOW_ELEMENT_ERROR" debug.log
```

### Debug Log Limits

- **Maximum log size**: 20 MB per log (truncated from middle if exceeded)
- **Retention**: system debug logs 24 hours, monitoring logs 7 days
- **Storage**: max 1,000 MB per org; oldest logs overwritten
- **Log generation throttle**: 1,000 MB in 15 minutes → trace flags auto-disabled
- **Max logs per user**: ~20 (older overwritten)
- **Trace flag duration**: max 24 hours

**If logs are being truncated:**
- Lower log levels for categories you're not investigating
- Focus only on relevant categories (set others to NONE or ERROR)
- Use targeted trace flags on specific classes/triggers instead of users

### Developer Console

The Developer Console provides a GUI for viewing and analyzing debug logs.

**Opening:** Click user avatar (top right) → Developer Console

**Key panels:**
- **Logs tab**: double-click a log to open it
- **Execution Overview**: timeline of events, limits consumption
- **Source panel**: view code with line numbers
- **Query Editor**: run SOQL/SOSL queries directly
- **Execute Anonymous Window**: Debug → Open Execute Anonymous Window (Ctrl+E)

**Execute Anonymous for quick debugging:**
```apex
// Quick data investigation
List<Account> accs = [SELECT Id, Name, CreatedDate FROM Account LIMIT 5];
System.debug(JSON.serializePretty(accs));

// Test a specific method
MyClass.myMethod('test param');

// Check governor limit usage
System.debug('SOQL: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
```


## Apex Replay Debugger

The Replay Debugger lets you step through Apex code execution using debug logs in VS Code
— like a traditional debugger but replay-based. Works with all orgs and unmanaged code.
Free, no special licenses needed.

### Setup and Usage

**Prerequisites:**
- VS Code with Salesforce Extension Pack
- Project source code must match what generated the log
- Debug log generated at FINER or FINEST for Apex Code

**Quick workflow (recommended for most cases):**
1. Set breakpoints: click the gutter next to line numbers in .cls or .trigger files
2. Open Command Palette: `Ctrl+Shift+P` (Windows/Linux) or `Cmd+Shift+P` (macOS)
3. Run: `SFDX: Launch Apex Replay Debugger with Current File`
   - Works on Apex test files, Anonymous Apex, or log files
   - Auto-creates trace flag, runs code, captures log, launches debugger

**Manual workflow (for Queueable, triggers, complex scenarios):**
1. Set breakpoints and checkpoints in code
2. Upload checkpoints: `SFDX: Update Checkpoints in Org`
3. Enable logging: `SFDX: Turn On Apex Debug Log for Replay Debugger`
4. Reproduce the issue in your org
5. Retrieve log: `SFDX: Get Apex Debug Logs` → select the log
6. Launch: `SFDX: Launch Apex Replay Debugger with Current File`
7. Use Debug toolbar to step through code

### Breakpoints vs Checkpoints

| Feature | Breakpoints | Checkpoints |
|---------|------------|-------------|
| **Set where** | .cls or .trigger files, gutter click | Same locations |
| **Requires deploy** | No | Yes (re-upload after changes) |
| **Shows variables** | Local variables in scope | Full heap dump — all local, static, and trigger context vars |
| **Limit** | Unlimited | Max 5 per org |
| **Expires** | Never (within session) | 30 minutes after upload |

**Setting checkpoints:**
- Command Palette → `SFDX: Toggle Checkpoint`
- Or right-click gutter → Add Conditional Breakpoint → Expression → type `Checkpoint`

### Debugging Tips

- Debug logs must match your deployed source — deploy before generating logs
- Heap dumps from checkpoints expire after 1 day
- Downloaded logs are in `.sfdx/tools/debug/logs`
- To restart: `SFDX: Launch Apex Replay Debugger with Last Log File`
- Stream logs in real-time: `sf apex tail log --color`


## Apex Interactive Debugger

A real-time debugger that pauses execution at breakpoints (not replay). Requires a
**Performance**, **Enterprise**, or **Unlimited** edition sandbox or scratch org, plus the
**Debug Apex** system permission.

**Setup:**
1. Create a Permission Set with System Permission → **Debug Apex**
2. Assign to your user
3. In VS Code, set breakpoints
4. Command Palette → `SFDX: Launch Apex Debugger Session`
5. Perform the action in your org — execution pauses at breakpoints

**When to use Interactive vs Replay:**

| Scenario | Use |
|----------|-----|
| Most daily debugging | Replay Debugger |
| Real-time variable inspection needed | Interactive Debugger |
| Production org | Replay Debugger (Interactive not available) |
| Scratch org / sandbox development | Either |
| Long-running or async process | Replay Debugger |


## Flow Debugging

### Flow Builder Debug Mode

Click **Debug** in Flow Builder toolbar to run a flow interactively with debug output.

**What you can do:**
- Choose a record or input variables to simulate
- See which path the flow takes (highlighted elements)
- Inspect variable values at each step
- See which elements execute and in what order
- Look for the stop sign icon — indicates an error occurred

**Limitations of Flow Debug:**
- Max 200 tests per flow
- Flow tests only available for record-triggered and Data Cloud-triggered flows
- Cannot test flows that run on record deletion
- Cannot test asynchronous paths
- Flow tests don't count toward code coverage requirements

**Scheduled Path testing tip:** set the schedule to run 1 minute later for quick testing.
Monitor pending actions in the Time-Based Workflow screen.

### Fault Paths (Fault Connectors)

Always add fault paths to data elements (Get Records, Create Records, Update Records,
Delete Records, Action elements). Without fault paths, users see the unhelpful
"An unhandled fault has occurred" message.

**Setting up fault paths:**
1. Hover over a data element in Flow Builder → click the down arrow
2. Select **Add Fault Path**
3. Connect to a Screen element (screen flows) or an Action element (auto-launched flows)
4. Use `{!$Flow.FaultMessage}` to display the actual error to users or log it

**Fault path patterns:**

For **screen flows** — show a user-friendly error screen:
- Add a Screen element on the fault path
- Add a Display Text component with `{!$Flow.FaultMessage}`
- Add a "Go Back" or "Try Again" button

For **auto-launched / record-triggered flows** — notify admins:
- Send an email alert with the flow name, error message, record ID, and user
- Post to a Chatter group or Slack channel via webhook
- Create a custom error log record (Application_Log__c or similar)
- Use an Invocable Apex method to call a logging framework like Nebula Logger

**Common causes of unhandled faults:**
- Flow tries to create/edit a record the running user can't access
- Required field not set on a create/update element
- Invalid ID format (wrong number of characters)
- Invalid email address in a send email action
- Blocked by a validation rule
- Governor limit exceeded (too many SOQL queries, etc.)

**Prevention patterns:**
- Use Decision elements to check data before DML elements
- Use Get Records to verify a record exists before updating
- Check Collection sizes before Loop elements
- Validate user input on screens before proceeding

### Flow Debugging with Debug Logs

Set a trace flag with the **Workflow** category at FINE or FINER to see flow events:
- `FLOW_START_INTERVIEW` / `FLOW_START_INTERVIEW_END`
- `FLOW_ELEMENT_BEGIN` / `FLOW_ELEMENT_END`
- `FLOW_ELEMENT_FAULT` — the element that failed
- `FLOW_VALUE_ASSIGNMENT` — variable assignments
- `FLOW_ACTIONCALL_DETAIL` — invocable action inputs/outputs

### Failed Flow Interviews

Setup → Paused and Failed Flow Interviews shows failed flow executions with details
including the failing element and error message. Fault emails sent to the flow creator or
admin also contain a direct link to the failed interview.


## LWC Debugging

### Enable Debug Mode

By default, LWC code is minified and optimized. Enable Debug Mode to see readable source:

1. Setup → Debug Mode (or search "Debug Mode" in Quick Find)
2. Check the box for your user
3. Reload the page (you'll see a debug mode banner at the top)

**Warning:** Debug mode slows page loads significantly. Only enable for users actively
debugging. Disable when done.

### Chrome DevTools

LWC debugging is optimized for Chrome DevTools. Open with:
- F12, or right-click → Inspect
- Windows/Linux: `Ctrl+Shift+I`
- macOS: `Cmd+Option+I`

**Finding your component code:**
- **Without LWS**: Sources panel → `lightning/n/modules/c/` (c namespace)
- **With LWS enabled** (default): Sources panel → `lws/` folder at bottom of Page pane
- If you can't find your code, you likely need to enable Debug Mode

**Essential DevTools setup:**
1. **Enable Custom Formatters**: Settings → Preferences → Console → Enable custom
   formatters. This unwraps Proxy objects so you see real values instead of `Proxy {}`.
   Note: may not work with LWS enabled — disable LWS temporarily if needed.
2. **Set up Ignore List**: Settings → Ignore List → Add regex:
   `/components/.*.js$` — hides framework code, pauses only on your exceptions

### Debugging Techniques

**Setting breakpoints:**
- Click line numbers in Sources panel to set line-of-code breakpoints
- Right-click a breakpoint → Edit Breakpoint → add a condition
  (e.g., `data !== undefined` for wired methods that fire twice)
- Use Event Listener Breakpoints in the Debugging pane for DOM events

**Handling Proxy objects:**
LWC wraps `@api`, `@wire`, and reactive properties in JavaScript Proxies (for
read-only enforcement). If you see `Proxy {}` in the console:
- Enable Custom Formatters (above)
- Or use `JSON.parse(JSON.stringify(myVar))` to unwrap in console
- Or right-click a value → "Store as global variable" → inspect `temp1`

**console.log tips:**
- Place in `connectedCallback()` or `renderedCallback()`, not `constructor()`
- For `@wire` methods, check both `data` and `error`:
  ```javascript
  @wire(getRecords)
  wiredResult({ data, error }) {
      if (data) { console.log('Data:', JSON.parse(JSON.stringify(data))); }
      if (error) { console.error('Error:', error); }
  }
  ```
- If console.log not showing: check browser console filter (set to "All levels"),
  clear cache (DevTools → Network → Disable cache), hard reload

**Pause on exceptions:**
Sources panel → enable "Pause on caught exceptions" (⏸ icon) to stop at the exact line
that throws an error.

### LWC Jest Testing for Debugging

Run Jest tests locally in VS Code for component-level debugging without deploying:
```bash
# Run all LWC tests
sf lightning lwc test run

# Run a specific test file
npx lwc-jest -- --testPathPattern=myComponent

# Run with VS Code debugger attached
# In launch.json, add a Jest debug configuration
```

### Common LWC Debugging Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Code shows minified in Sources | Debug Mode not enabled | Setup → Debug Mode → enable for user |
| Can't find component in Sources | LWS changes folder structure | Look in `lws/` folder instead of `lightning/n/` |
| Proxy {} instead of real values | Custom formatters not enabled | DevTools Settings → Enable custom formatters |
| console.log not appearing | Wrong lifecycle hook or filter | Use connectedCallback(), check console filters |
| Component not rendering | Conditional rendering issue | Check `if:true` / `lwc:if` conditions, check target configs |
| Wire method returns undefined | Cache, missing field, perms | Check `@wire` params, FLS, inspect Network tab for API calls |


## Governor Limits Debugging

### Key Limits Quick Reference

| Limit | Synchronous | Asynchronous | Error Message |
|-------|-------------|-------------|---------------|
| SOQL queries | 100 | 200 | `Too many SOQL queries: 101` |
| SOQL query rows | 50,000 | 50,000 | `Too many query rows: 50001` |
| DML statements | 150 | 150 | `Too many DML statements: 151` |
| DML rows | 10,000 | 10,000 | `Too many DML rows` |
| CPU time | 10,000 ms | 60,000 ms | `Apex CPU time limit exceeded` |
| Heap size | 6 MB | 12 MB | `Apex heap size too large` |
| Callouts | 100 | 100 | `Too many callouts` |
| SOSL queries | 20 | 20 | `Too many SOSL queries` |
| Future calls | 50 | 0 | `Too many future calls` |
| Queueable jobs | 50 | 1 | `Too many queueable jobs added to the queue` |
| Email invocations | 10 | 10 | `Too many Email Invocations` |

### Using the Limits Class

Apex provides the `Limits` class to programmatically check consumption at any point:

```apex
// Add this at key points in your code during debugging
System.debug('=== Limits Check ===');
System.debug('SOQL: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
System.debug('Query Rows: ' + Limits.getQueryRows() + '/' + Limits.getLimitQueryRows());
System.debug('DML: ' + Limits.getDMLStatements() + '/' + Limits.getLimitDMLStatements());
System.debug('DML Rows: ' + Limits.getDmlRows() + '/' + Limits.getLimitDmlRows());
System.debug('CPU: ' + Limits.getCpuTime() + 'ms/' + Limits.getLimitCpuTime() + 'ms');
System.debug('Heap: ' + Limits.getHeapSize() + '/' + Limits.getLimitHeapSize());
System.debug('Callouts: ' + Limits.getCallouts() + '/' + Limits.getLimitCallouts());
```

**Proactive limit checking:**
```apex
// Check if approaching a limit before expensive operation
if (Limits.getQueries() >= Limits.getLimitQueries() - 10) {
    // Near SOQL limit — abort, queue async, or skip
    System.debug(LoggingLevel.WARN, 'Approaching SOQL limit!');
}
```

### LIMIT_USAGE_FOR_NS in Debug Logs

At the end of every transaction, the debug log contains a `LIMIT_USAGE_FOR_NS` section
showing consumption vs maximum for every governor limit. Search for this section to quickly
see which limits are under pressure.

### Common Limit Violations and Fixes

**Too many SOQL queries (101)**
- Cause: SOQL inside a loop, or multiple triggers/flows/automations each running queries
- Fix: move queries outside loops, use Maps for lookups, consolidate triggers
```apex
// BAD — query per record
for (Contact c : contacts) {
    Account a = [SELECT Name FROM Account WHERE Id = :c.AccountId];
}

// GOOD — single query, map lookup
Map<Id, Account> accMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
);
for (Contact c : contacts) {
    Account a = accMap.get(c.AccountId);
}
```

**Too many DML statements (151)**
- Cause: DML inside a loop
- Fix: collect records in a List, perform one bulk DML
```apex
// BAD — DML per record
for (Account a : accounts) { update a; }

// GOOD — single bulk DML
update accounts;
```

**CPU time limit exceeded**
- Cause: deeply nested loops, inefficient algorithms, recursive triggers
- Fix: optimize logic, reduce iterations, offload to async (Queueable/Batch)
- Tip: database time is NOT counted in CPU time — use aggregate SOQL when possible

**Heap size too large**
- Cause: loading too many records into memory, large JSON payloads, base64 content
- Fix: query only needed fields, use SOQL for-loops, process in batches
```apex
// SOQL for-loop — processes 200 records at a time, doesn't load all into heap
for (Account a : [SELECT Id, Name FROM Account WHERE Industry = 'Tech']) {
    // process each batch of 200
}
```

**Mixed DML error**
- Cause: DML on setup objects (User, PermissionSetAssignment) and non-setup objects
  in the same transaction
- Fix: separate into different transactions using `@future` or Queueable

### Debugging Recursive Triggers

Recursive triggers are a common source of limit violations. Symptoms: unexpected multiple
executions, doubled DML counts, CPU timeouts.

**Detection pattern:**
```apex
// Add to trigger handler to detect recursion
System.debug('=== Trigger Entry: ' + Trigger.operationType
    + ' | Records: ' + Trigger.new?.size()
    + ' | SOQL: ' + Limits.getQueries()
    + ' | DML: ' + Limits.getDMLStatements());
```

**Prevention — static flag pattern:**
```apex
public class TriggerGuard {
    private static Set<String> executedContexts = new Set<String>();

    public static Boolean hasRun(String context) {
        return executedContexts.contains(context);
    }
    public static void setRun(String context) {
        executedContexts.add(context);
    }
}

// In trigger handler:
if (TriggerGuard.hasRun('AccountAfterUpdate')) return;
TriggerGuard.setRun('AccountAfterUpdate');
```


## Logging Frameworks

Native debug logs are transient (24 hours), per-user, size-limited, and only support
`System.debug()` in Apex. For production observability, use a logging framework.

### Nebula Logger (Recommended)

The most popular open-source logging framework for Salesforce. Built 100% natively,
uses Platform Events (`LogEntryEvent__e`) so logs persist even when transactions roll back.

**GitHub:** github.com/jongpie/NebulaLogger

**Key features:**
- Unified logging across Apex, LWC, Aura, Flow, and OmniStudio
- Persistent log storage in custom objects (`Log__c`, `LogEntry__c`)
- Log levels: DEBUG, INFO, WARN, ERROR, FATAL
- Automatic data masking of sensitive data (via LogEntryDataMaskRule__mdt)
- Tagging system for categorizing/filtering logs
- Real-time log streaming via platform events
- Plugin framework for extending (Slack notifications, custom handlers)
- Reports and dashboards on log data
- Configurable retention and automatic purging

**Install (unlocked package via SF CLI):**
```bash
sf package install --wait 20 --security-type AdminsOnly --package <versionId>
# Get latest version ID from the GitHub releases page
```

**Usage in Apex:**
```apex
Logger.info('Processing started for ' + accounts.size() + ' accounts');
try {
    // business logic
    processAccounts(accounts);
    Logger.debug('Processing complete');
} catch (Exception e) {
    Logger.error('Processing failed', e);  // auto-captures stack trace
}
Logger.saveLog();  // always call at end — publishes the platform event
```

**Usage in Flows:**
- Use invocable actions: "Add Log Entry", "Add Log Entry for an SObject Record",
  "Add Log Entry for an SObject Collection"
- Call "Save Log" action after your log entries
- On fault paths: use "Add Log Entry" with `{!$Flow.FaultMessage}` as the message

**Usage in LWC:**
```javascript
import Logger from 'c/logger';  // provided component

handleError(error) {
    Logger.error('Component error occurred').setError(error);
    Logger.saveLog();
}
```

### Custom Logging Framework (DIY Alternative)

If you cannot install packages, build a minimal framework:

**Custom object:** `Application_Log__c`
- `Level__c` (Picklist: DEBUG, INFO, WARN, ERROR, FATAL)
- `Source__c` (Text: class/trigger/flow name)
- `Message__c` (Long Text)
- `StackTrace__c` (Long Text)
- `TransactionId__c` (Text: unique per request for correlation)
- `RecordId__c` (Text: related record if applicable)
- `UserId__c` (Lookup to User)
- `Timestamp__c` (DateTime)

**Key implementation considerations:**
- Buffer log records in a static List and insert all at once via `finally` block or
  at transaction end — avoids DML per log statement
- Consider using Platform Events instead of direct DML so logs survive rollbacks
- Use Custom Metadata Types (`LoggerSetting__mdt`) to control minimum log level per
  environment (WARN in production, DEBUG in sandboxes)
- Schedule a cleanup batch job to purge old logs and manage storage


## Debugging Tools

### Salesforce Inspector (Chrome Extension)

A browser extension that provides real-time insights into SOQL queries, record data,
governor limits, and more directly in your Salesforce UI.

**Key features:**
- Run SOQL/SOSL queries inline
- Export query results to CSV/Excel
- View/edit record field values directly
- Inspect governor limit consumption
- Quick data import/export
- View field API names on any page layout

### VS Code Debugging Commands (Quick Reference)

| Command | What It Does |
|---------|-------------|
| `SFDX: Launch Apex Replay Debugger with Current File` | Run and debug current file |
| `SFDX: Launch Apex Replay Debugger with Last Log File` | Re-run last debug session |
| `SFDX: Turn On Apex Debug Log for Replay Debugger` | Enable trace flag (30 min) |
| `SFDX: Get Apex Debug Logs` | Download logs from org |
| `SFDX: Update Checkpoints in Org` | Upload checkpoint definitions |
| `SFDX: Toggle Checkpoint` | Set/remove a checkpoint |
| `SFDX: Launch Apex Debugger Session` | Start Interactive Debugger |

### SF CLI Log Commands

```bash
# Stream logs in real time
sf apex tail log
sf apex tail log --color
sf apex tail log --color --debug-level MyCustomLevel

# Filter streamed logs to only System.debug output
sf apex tail log --color | grep USER_DEBUG

# List available logs
sf apex log list

# Get specific log
sf apex log get --log-id <logId>
sf apex log get --number 5 --output-dir ./logs

# Run anonymous Apex (quick debugging)
sf apex run --file script.apex
echo "System.debug(UserInfo.getUserName());" | sf apex run
```


## Debugging Decision Guide

```
Problem reported
    │
    ├─ Apex error/exception?
    │   ├─ Can reproduce? → Replay Debugger (set breakpoints, step through)
    │   ├─ Intermittent?  → Add Nebula Logger, check LIMIT_USAGE_FOR_NS
    │   └─ Production?    → Debug logs + Replay Debugger (log analysis only)
    │
    ├─ Flow error?
    │   ├─ "Unhandled fault" → Check fault emails, add fault paths
    │   ├─ Wrong behavior   → Flow Builder Debug mode, trace variable values
    │   └─ Performance      → Debug logs with Workflow=FINER, check SOQL/DML counts
    │
    ├─ LWC issue?
    │   ├─ JS error        → Chrome DevTools, Pause on exceptions
    │   ├─ Data not loading → Network tab (check API calls), inspect @wire params
    │   ├─ Rendering issue → Elements panel, check lwc:if / template conditions
    │   └─ Styling issue   → Elements panel → Styles tab
    │
    ├─ Governor limit error?
    │   ├─ Which limit?    → Read the error message, check LIMIT_USAGE_FOR_NS
    │   ├─ SOQL 101        → Search for SOQL_EXECUTE in log, check for queries in loops
    │   ├─ DML 151         → Search for DML_BEGIN in log, check for DML in loops
    │   ├─ CPU time        → Profile with Apex Profiling=FINEST, check nested loops
    │   └─ Heap size       → Check query field lists, collection sizes, JSON payloads
    │
    └─ Integration error?
        ├─ Callout failing → Debug log with Callout=FINE, check request/response
        ├─ Auth error      → Verify Named Credential, check token expiry
        └─ Timeout         → Check response times in log, increase timeout setting
```


## Common Deployment Error Patterns and Solutions

Real-world deployment failures and their fixes. Reference when a deploy fails unexpectedly.

### 1. Flow XML: Whitespace in Element References

**Error:** `The value for the field "collectionReference" is not valid` / element not found

**Rule:** All XML element reference values in Flow metadata MUST be on a single line
with no leading/trailing whitespace or newlines. Salesforce treats whitespace inside
reference tags as literal text.

**Affected tags:** `<leftValueReference>`, `<rightValue><elementReference>`,
`<collectionReference>`, `<assignToReference>`, `<inputReference>`,
`<targetReference>`, `<connector><targetReference>`, `<subjectNameOrId>`,
`<assignRecordIdToReference>`, and all tags containing Flow element name references.

```xml
<!-- WRONG - whitespace treated as part of the value -->
<leftValueReference>
  Get_Open_Renewal_Tasks
</leftValueReference>

<!-- CORRECT - single line, no whitespace -->
<leftValueReference>Get_Open_Renewal_Tasks</leftValueReference>
```

**Detection:**
```bash
# Find Flow XML with multiline references
grep -Pzn '<(leftValueReference|elementReference|collectionReference|assignToReference|inputReference|targetReference)>\s*\n' force-app/**/*.flow-meta.xml
```

**Fix:** Remove all whitespace/newlines inside reference tags. If your formatter adds
line breaks, add `*.flow-meta.xml` to `.prettierignore`.

### 2. Flow recordUpdates: Cannot Combine inputReference with inputAssignments

**Error:** `You can't use the sObjectInputReference field with the inputAssignments field`

**Rule:** In `<recordUpdates>`, choose ONE pattern - never combine them:

| Pattern | Elements Used | Use Case |
|---------|--------------|----------|
| **A - Direct update** | `<object>` + `<filters>` + `<inputAssignments>` | Update records matching criteria with specific field values |
| **B - Collection update** | `<inputReference>` only (no filters, no object, no inputAssignments) | Update a pre-built collection of records |

For updating queried records, use the **Loop + Assignment** pattern:
1. Get Records -> Loop through results
2. In Assignment: set field values on loop variable
3. Add modified loop variable to a collection (`<isCollection>true</isCollection>`)
4. After loop: `<recordUpdates>` with only `<inputReference>` pointing to the collection

### 3. Task List View: Column Name Format Depends on Object Path

**Error:** `Could not resolve list view column: TASK.SUBJECT`

| Column | Under `objects/Task/` | Under `objects/Activity/` |
|--------|----------------------|--------------------------|
| Subject | `SUBJECT` | `TASK.SUBJECT` |
| Related To | `WHAT_NAME` | `TASK.WHAT_NAME` |
| Due Date | `DUE_DATE` | `TASK.DUE_DATE` |
| Priority | `PRIORITY` | `TASK.PRIORITY` |
| Status | `STATUS` | `TASK.STATUS` |
| Assigned Alias | `CORE.USERS.ALIAS` | `CORE.USERS.ALIAS` |

Filter fields under `objects/Task/`: use bare names (`STATUS`, `PRIORITY`, `IS_CLOSED`).

### 4. Task List View: Queue Scope Requires objects/Task/ Path

**Error:** `Cannot assign to queues for: Activity`

List views with `<filterScope>Queue</filterScope>` for Tasks MUST be under
`objects/Task/listViews/`, not `objects/Activity/listViews/`.

### 5. FlexiPage: Always Retrieve Before Modifying

**Error:** Generic FlexiPage deploy failure / "An unexpected error occurred"

FlexiPages drift between repo and org (component identifiers, property values, template
versions change via the UI). Before modifying:

```bash
# 1. Retrieve current version from org
sf project retrieve start --metadata "FlexiPage:My_Page" --ignore-conflicts
# 2. Apply your changes to the retrieved version
# 3. Deploy the modified version
```

Never modify a FlexiPage from the repo without first confirming it matches the org state.

### 6. Flow Scheduled Trigger: Required Fields

Scheduled Flows (`<triggerType>Scheduled</triggerType>`) require ALL of these
`<start>` sub-elements - missing any causes a deploy error:
- `<schedule><frequency>` - e.g., Once, Daily, Weekly
- `<schedule><startDate>` - format: YYYY-MM-DD
- `<schedule><startTime>` - format: HH:MM:SS.SSSZ
- `<scheduledPaths>` - at least one scheduled path with a `<connector>`

### 7. Flow Chatter Post: Dynamic Record Targets

When posting to Chatter on a record created in the same Flow, you cannot reference
the record variable's Id directly. Instead:
1. On `<recordCreates>`: add `<assignRecordIdToReference>varCreatedRecordId</assignRecordIdToReference>`
2. Define a `<variables>` element for `varCreatedRecordId` (dataType: String)
3. In Chatter `<actionCalls>`: reference `varCreatedRecordId` as subjectNameOrId input

### 8. Deployment Strategy: Use Targeted Source Paths

Avoid full project deploys when deploying specific features or when the full project
has known issues in other components:

```bash
# Deploy only specific files - avoids failures from unrelated metadata
sf project deploy start \
  --source-dir force-app/main/default/flows/My_Flow.flow-meta.xml \
  --source-dir force-app/main/default/classes/MyClass.cls \
  --ignore-conflicts

# Use -d shorthand for targeted deploys during iterative development
sf project deploy start \
  -d force-app/main/default/objects/MyObject__c \
  -d force-app/main/default/classes/MyClass.cls

# Validate first without deploying (dry run)
sf project deploy start \
  --source-dir force-app/main/default/flows/My_Flow.flow-meta.xml \
  --dry-run
```

**Always test targeted deploys first** — when the full project has known issues or
unrelated failures, use `-d` / `--source-dir` to deploy only the components you're
working on. This isolates your changes from pre-existing problems.

### 9. Verify Pre-Existing Failures Before Debugging

When a deploy fails, always test with the **unmodified file** first to distinguish
your errors from org drift or pre-existing issues:

```bash
# 1. Stash your changes
git stash

# 2. Deploy the unmodified version
sf project deploy start \
  -d force-app/main/default/objects/MyObject__c/fields/MyField__c.field-meta.xml

# 3. If the unmodified file also fails, the problem is org drift — not your change
# 4. Restore your changes
git stash pop
```

This prevents wasting time debugging "your" errors that are actually caused by
org-level changes (field deletions, permission changes, package upgrades) that
happened outside source control.

### 10. CampaignMember: Standard Junction Object Restrictions

CampaignMember is a special standard junction object (Campaign ↔ Contact/Lead) with
metadata restrictions that differ from regular objects:

- **No field history tracking** — `enableHistory` and `trackHistory` are not
  supported. The API will reject metadata that includes history tracking settings.
- **Limited metadata API support** — certain operations available on regular custom
  or standard objects are not supported for CampaignMember.
- **Standard fields are implicit** — fields like `Status` and `FirstRespondedDate`
  exist on the object but cannot be explicitly declared in layout XML.

When working with CampaignMember, retrieve the current metadata from the org first
to see exactly which elements are supported:
```bash
sf project retrieve start --metadata "CustomObject:CampaignMember" --target-org myorg
```

### 11. Related List Field References: Format Varies by List Type

In Layout XML, related list column references use different formats depending on
the related list type. CampaignMember-related lists are a common source of errors:

| Related List | Campaign Name Column | Custom Field Column |
|-------------|---------------------|-------------------|
| `RelatedCampaignList` | `CAMPAIGN.NAME` | `CampaignMember.MyField__c` |
| Standard CampaignMember fields | Cannot be explicitly specified in layout XML | N/A |

**Rules:**
- `RelatedCampaignList` uses `CAMPAIGN.NAME` for the campaign name (parent object
  prefix), but `CampaignMember.<field>` for custom fields on the junction object.
- Standard CampaignMember fields (`Status`, `FirstRespondedDate`) are managed by
  Salesforce and **cannot** be explicitly listed as columns in the layout XML — they
  appear automatically.
- When in doubt, retrieve the layout from the org and inspect which columns
  Salesforce includes:
  ```bash
  sf project retrieve start --metadata "Layout:Contact-Contact Layout" --target-org myorg
  ```

---

## Debugging Anti-Patterns

| Anti-Pattern | Why It's Bad | Better Approach |
|-------------|-------------|-----------------|
| Setting ALL log levels to FINEST | Generates huge logs, hits size limits, slow | Set only relevant categories to FINEST |
| Leaving trace flags on permanently | Slows org performance, fills log storage | Enable only when debugging, remove when done |
| Using System.debug() in production code | No structured search, fills logs with noise | Use a logging framework (Nebula Logger) |
| Debugging directly in production | Risk to live data, limited tooling | Use sandbox/scratch org, reproduce there |
| Ignoring LIMIT_USAGE_FOR_NS | Missing early warnings of limits pressure | Always check this section in debug logs |
| Not adding fault paths to flows | Users get unhelpful "unhandled fault" error | Always add fault paths to data/action elements |
| Printing entire SObjects in debug | Massive log output, may expose sensitive data | Log only the specific fields you need |
| No transaction correlation ID | Can't trace related log entries together | Use Nebula Logger or a custom TransactionId |


---

## Sources

- developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_debugging_debug_log.htm
- developer.salesforce.com/docs/platform/sfvscode-extensions/guide/replay-debugger.html
- developer.salesforce.com/docs/platform/lwc/guide/debug-intro.html
- developer.salesforce.com/docs/platform/lwc/guide/debug-dev-tools.html
- developer.salesforce.com/docs/platform/lwc/guide/debug-mode-enable.html
- help.salesforce.com/s/articleView?id=platform.flow_test_debug.htm
- trailhead.salesforce.com/content/learn/projects/find-and-fix-bugs-with-apex-replay-debugger
- trailhead.salesforce.com/content/learn/modules/developer_console/developer_console_logs
- trailhead.salesforce.com/content/learn/modules/lwc-troubleshooting/get-ready-to-troubleshoot
- trailhead.salesforce.com/content/learn/modules/flow-implementation-2/handle-flow-errors-with-fault-paths
- github.com/jongpie/NebulaLogger
- blog.beyondthecloud.dev/blog/apex-debugging
- blog.beyondthecloud.dev/blog/chrome-dev-tools-for-salesforce-lwc-developers
