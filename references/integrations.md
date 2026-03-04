# Integration Patterns Guide

## Authentication Patterns

### Named Credentials (Recommended)
Never hardcode endpoints or credentials. Use Named Credentials for all external callouts.

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_Named_Credential/api/v1/accounts');
req.setMethod('GET');
req.setHeader('Content-Type', 'application/json');

Http http = new Http();
HttpResponse res = http.send(req);
if (res.getStatusCode() == 200) {
    Map<String, Object> result = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
}
```

### External Credentials (Modern approach)
API 59.0+: External Credentials replace Legacy Named Credentials.
Define authentication at the External Credential level, reference via Named Credential.

## REST Callouts

### Basic Pattern
```apex
public class ExternalApiService {
    public static HttpResponse callApi(String endpoint, String method, String body) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod(method);
        req.setHeader('Content-Type', 'application/json');
        req.setTimeout(30000); // 30 seconds

        if (String.isNotBlank(body)) {
            req.setBody(body);
        }

        Http http = new Http();
        return http.send(req);
    }
}
```

### Callout from Trigger (must be async)
```apex
// Callouts not allowed in trigger context — use Queueable
public class AccountSyncJob implements Queueable, Database.AllowsCallouts {
    private List<Id> accountIds;

    public AccountSyncJob(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
        // Make callout here
        HttpResponse res = ExternalApiService.callApi(
            'callout:My_API/sync',
            'POST',
            JSON.serialize(accounts)
        );
    }
}

// In trigger handler:
System.enqueueJob(new AccountSyncJob(accountIds));
```

### Governor Limits for Callouts
- 100 callouts per transaction
- 120 seconds total callout time per transaction
- 12 MB max response size per callout
- Callout timeout: max 120 seconds per callout

## Inbound REST API (Custom Apex REST)

```apex
@RestResource(urlMapping='/accounts/*')
global class AccountRestService {
    @HttpGet
    global static Account getAccount() {
        RestRequest req = RestContext.request;
        String accountId = req.requestURI.substringAfterLast('/');
        return [SELECT Id, Name, Industry FROM Account WHERE Id = :accountId WITH USER_MODE];
    }

    @HttpPost
    global static Id createAccount(String name, String industry) {
        Account a = new Account(Name = name, Industry = industry);
        insert a;
        return a.Id;
    }

    @HttpPatch
    global static Account updateAccount() {
        RestRequest req = RestContext.request;
        String accountId = req.requestURI.substringAfterLast('/');
        Account a = [SELECT Id FROM Account WHERE Id = :accountId];
        Map<String, Object> params = (Map<String, Object>) JSON.deserializeUntyped(req.requestBody.toString());
        // Apply updates...
        update a;
        return a;
    }
}
```

Endpoint: `POST /services/apexrest/accounts`

## Platform Events (Event-Driven Architecture)

### Define Platform Event
In Setup or via metadata: `My_Event__e` with custom fields.

### Publish
```apex
My_Event__e event = new My_Event__e(
    Record_Id__c = recordId,
    Action__c = 'Created',
    Payload__c = JSON.serialize(data)
);
Database.SaveResult sr = EventBus.publish(event);
if (!sr.isSuccess()) {
    // Handle publish failure
}
```

### Subscribe (Apex Trigger on Platform Event)
```apex
trigger MyEventTrigger on My_Event__e (after insert) {
    for (My_Event__e event : Trigger.new) {
        // Process event
        // Each event runs in its own transaction with own limits
    }
}
```

### Subscribe from LWC (Streaming)
```javascript
import { subscribe, unsubscribe } from 'lightning/empApi';

connectedCallback() {
    subscribe('/event/My_Event__e', -1, (message) => {
        console.log('Event received:', JSON.stringify(message));
    }).then(subscription => {
        this.subscription = subscription;
    });
}
```

## External Services (Declarative Integrations)

Register an OpenAPI spec to auto-generate invocable actions usable in Flows.
Spring '26 adds decomposition for ExternalServiceRegistration metadata.

## Integration Best Practices

1. **Named Credentials** for all external endpoints (never hardcode URLs/keys)
2. **Async for trigger callouts** — use Queueable with Database.AllowsCallouts
3. **Retry logic** for transient failures (use Platform Events or Queueable chaining)
4. **Idempotency** — design APIs to handle duplicate calls safely
5. **Bulk patterns** — batch multiple records into single callouts where possible
6. **Error logging** — create custom object or use Platform Events for error tracking
7. **Timeouts** — set appropriate timeouts, handle timeout exceptions
8. **External Client Apps** (Spring '26) — use instead of Connected Apps for new integrations
