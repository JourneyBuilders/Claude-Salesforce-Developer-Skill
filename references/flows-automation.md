# Flows & Automation Guide

## Flow Types & When to Use

| Flow Type | Use When |
|-----------|----------|
| **Record-Triggered** | Automate on record create/update/delete (replaces Process Builder & Workflow Rules) |
| **Screen Flow** | Guided user interaction, wizards, custom forms |
| **Schedule-Triggered** | Recurring batch processing (replace scheduled Apex for simple cases) |
| **Platform Event-Triggered** | React to platform events asynchronously |
| **Autolaunched** | Called from Apex, other Flows, or REST API |

### Decision: Flow vs Apex

**Use Flow when:** Simple field updates, email alerts, task creation, screen-based wizards, admin-maintainable logic.

**Use Apex when:** Complex business logic, precise transaction control, external callouts with complex handling, high-volume processing (50k+), complex error handling.

## Record-Triggered Flows

### Trigger Types
- **Before Save** — Modify the triggering record (fast, no DML needed)
- **After Save** — Create/update other records, send notifications
- **Before/After Delete** — Validate or cleanup

### Best Practice: One Flow per object per trigger timing. Use Decision elements to branch.

## Screen Flows in LWC

```html
<lightning-flow flow-api-name="My_Flow"
    flow-input-variables={inputVariables}
    onstatuschange={handleFlowStatus}>
</lightning-flow>
```

## Invocable Apex (Call Apex from Flow)

```apex
public class InvocableProcessor {
    @InvocableMethod(label='Process Records' category='Custom')
    public static List<Result> process(List<Request> requests) {
        List<Result> results = new List<Result>();
        // Always bulkify - process all requests together
        Set<Id> ids = new Set<Id>();
        for (Request req : requests) { ids.add(req.recordId); }
        // Single query, process, return results
        return results;
    }

    public class Request {
        @InvocableVariable(required=true label='Record Id')
        public Id recordId;
    }

    public class Result {
        @InvocableVariable(label='Status')
        public String status;
    }
}
```

## Launch Flow from Apex

```apex
Map<String, Object> inputs = new Map<String, Object>();
inputs.put('recordId', accountId);
Flow.Interview myFlow = Flow.Interview.createInterview('My_Flow', inputs);
myFlow.start();
String result = (String) myFlow.getVariableValue('outputStatus');
```

## Platform Events for Decoupling

```apex
// Publish event from Apex
EventBus.publish(new My_Event__e(Record_Id__c = accountId, Action__c = 'Updated'));
```

Flow subscribes to the platform event and processes asynchronously. Great for avoiding trigger recursion and decoupling processes.

## Flow Best Practices

1. One Flow per object per trigger timing — consolidate with Decision elements
2. Use Before Save for field updates (faster, no DML)
3. Bulkify: Use Get Records outside loops, use collection variables
4. Always set entry conditions to minimize execution
5. Use Fault paths on DML operations for error handling
6. Naming: prefix with object name, e.g. "Account - Set Default Industry"
7. Flow tests (beta) or test via Apex by triggering DML
8. Flows share same governor limits as Apex (100 SOQL, 150 DML, etc.)

## Spring '26 Flow Updates

- Collapsible branching (Decision, Loop, Wait elements)
- Agentforce natural language editing for record/schedule-triggered flows
- Flow version comparison (side by side)
- On-canvas performance metrics (run counts, status distributions)
- Kanban board view for flow elements
