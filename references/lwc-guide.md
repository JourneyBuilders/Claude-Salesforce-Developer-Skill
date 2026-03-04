# Lightning Web Components (LWC) Development Guide

## Table of Contents
1. [Component Structure](#component-structure)
2. [Data Binding & Reactivity](#data-binding--reactivity)
3. [Wire Service & Apex Calls](#wire-service--apex-calls)
4. [Navigation & Events](#navigation--events)
5. [Lightning Data Service (LDS)](#lightning-data-service-lds)
6. [Component Communication](#component-communication)
7. [Best Practices](#best-practices)
8. [Spring '26 Updates](#spring-26-updates)

---

## Component Structure

```
force-app/main/default/lwc/myComponent/
├── myComponent.html        # Template (required)
├── myComponent.js          # Controller (required)
├── myComponent.js-meta.xml # Metadata config (required)
├── myComponent.css         # Styles (optional)
└── __tests__/              # Jest tests (optional)
    └── myComponent.test.js
```

### Metadata XML Configuration
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>66.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
        <target>lightning__FlowScreen</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <objects>
                <object>Account</object>
            </objects>
            <property name="title" type="String" label="Card Title" default="Account Details"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

---

## Data Binding & Reactivity

### Reactive Properties
```javascript
import { LightningElement, api, track } from 'lwc';

export default class MyComponent extends LightningElement {
    // Public reactive property (set by parent or App Builder)
    @api recordId;
    @api objectApiName;

    // Private reactive properties — reassignment triggers re-render
    greeting = 'Hello';
    items = [];

    // Computed property (getter) — automatically re-evaluates
    get formattedGreeting() {
        return `${this.greeting}, World!`;
    }

    // Mutating object/array properties: reassign to trigger reactivity
    addItem(item) {
        this.items = [...this.items, item]; // Spread to new array
    }

    updateItem(index, value) {
        this.items = this.items.map((item, i) =>
            i === index ? { ...item, ...value } : item
        );
    }
}
```

### Template Directives
```html
<template>
    <!-- Conditional rendering -->
    <template lwc:if={isLoaded}>
        <p>{greeting}</p>
    </template>
    <template lwc:elseif={isLoading}>
        <lightning-spinner alternative-text="Loading"></lightning-spinner>
    </template>
    <template lwc:else>
        <p>Error occurred</p>
    </template>

    <!-- Iteration -->
    <template for:each={items} for:item="item">
        <div key={item.id}>
            {item.name}
        </div>
    </template>

    <!-- Dynamic components (API 55.0+) -->
    <lwc:component lwc:is={componentConstructor}></lwc:component>
</template>
```

---

## Wire Service & Apex Calls

### Wire to Apex Method (Cacheable/Reactive)
```javascript
import { LightningElement, wire, api } from 'lwc';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

export default class AccountList extends LightningElement {
    @api recordId;

    // Wire to a property — auto-provisions data and error
    @wire(getAccounts, { parentId: '$recordId' })
    accounts;

    // Wire to a function — more control
    @wire(getAccounts, { parentId: '$recordId' })
    wiredAccounts({ data, error }) {
        if (data) {
            this.accounts = data;
            this.error = undefined;
        } else if (error) {
            this.error = error;
            this.accounts = undefined;
        }
    }
}
```

### Imperative Apex Calls (Non-Cacheable / DML)
```javascript
import { LightningElement } from 'lwc';
import saveAccount from '@salesforce/apex/AccountController.saveAccount';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class AccountForm extends LightningElement {
    isLoading = false;

    async handleSave() {
        this.isLoading = true;
        try {
            const result = await saveAccount({ acc: this.accountData });
            this.dispatchEvent(new ShowToastEvent({
                title: 'Success',
                message: 'Account saved',
                variant: 'success'
            }));
        } catch (error) {
            this.dispatchEvent(new ShowToastEvent({
                title: 'Error',
                message: error.body?.message || 'Unknown error',
                variant: 'error'
            }));
        } finally {
            this.isLoading = false;
        }
    }
}
```

### Refresh Wire Data After DML
```javascript
import { refreshApex } from '@salesforce/apex';

// Store the wired result reference
@wire(getAccounts)
wiredResult;

async handleSave() {
    await saveAccount({ acc: this.data });
    await refreshApex(this.wiredResult); // Re-fetches wire data
}
```

---

## Navigation & Events

### NavigationMixin
```javascript
import { LightningElement } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';

export default class MyNav extends NavigationMixin(LightningElement) {
    navigateToRecord(recordId) {
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: {
                recordId: recordId,
                objectApiName: 'Account',
                actionName: 'view'
            }
        });
    }

    navigateToList() {
        this[NavigationMixin.Navigate]({
            type: 'standard__objectPage',
            attributes: {
                objectApiName: 'Account',
                actionName: 'list'
            },
            state: {
                filterName: 'Recent'
            }
        });
    }

    // Navigate to Flow (Spring '26)
    navigateToFlow() {
        this[NavigationMixin.Navigate]({
            type: 'standard__flow',
            attributes: {
                flowApiName: 'My_Flow'
            },
            state: {
                inputVar1: 'value1'
            }
        });
    }
}
```

---

## Lightning Data Service (LDS)

### getRecord / getFieldValue (No Apex Needed)
```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

const FIELDS = [NAME_FIELD, INDUSTRY_FIELD];

export default class AccountDetail extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: FIELDS })
    account;

    get name() {
        return getFieldValue(this.account.data, NAME_FIELD);
    }

    get industry() {
        return getFieldValue(this.account.data, INDUSTRY_FIELD);
    }
}
```

### createRecord / updateRecord / deleteRecord
```javascript
import { createRecord, updateRecord, deleteRecord } from 'lightning/uiRecordApi';
import ACCOUNT_OBJECT from '@salesforce/schema/Account';
import NAME_FIELD from '@salesforce/schema/Account.Name';

// Create
const fields = {};
fields[NAME_FIELD.fieldApiName] = 'New Account';
const recordInput = { apiName: ACCOUNT_OBJECT.objectApiName, fields };
const account = await createRecord(recordInput);

// Update
const fields = {};
fields['Id'] = this.recordId;
fields[NAME_FIELD.fieldApiName] = 'Updated Name';
await updateRecord({ fields });

// Delete
await deleteRecord(this.recordId);
```

---

## Component Communication

### Parent → Child: @api properties and methods
```javascript
// Child
export default class ChildComponent extends LightningElement {
    @api message;
    @api
    refreshData() {
        // Parent can call this method
    }
}
```
```html
<!-- Parent template -->
<c-child-component message="Hello" lwc:ref="child"></c-child-component>
```
```javascript
// Parent calling child method
this.refs.child.refreshData();
```

### Child → Parent: Custom Events
```javascript
// Child fires event
this.dispatchEvent(new CustomEvent('selected', {
    detail: { recordId: this.selectedId },
    bubbles: false,   // default: false
    composed: false    // default: false
}));
```
```html
<!-- Parent listens -->
<c-child-component onselected={handleSelected}></c-child-component>
```

### Unrelated Components: Lightning Message Service (LMS)
```javascript
import { publish, subscribe, MessageContext } from 'lightning/messageService';
import MY_CHANNEL from '@salesforce/messageChannel/MyChannel__c';

@wire(MessageContext) messageContext;

// Publish
publish(this.messageContext, MY_CHANNEL, { recordId: '001...' });

// Subscribe
connectedCallback() {
    this.subscription = subscribe(this.messageContext, MY_CHANNEL, (message) => {
        this.handleMessage(message);
    });
}
```

---

## Best Practices

1. **Use LDS over Apex** when possible — automatic caching, no Apex to maintain
2. **Always handle errors** in Apex calls — show toast or inline error
3. **Use `lwc:if` over `if:true`** — `if:true` is deprecated
4. **Minimize DOM operations** — keep templates simple, avoid deep nesting
5. **Use `@api` sparingly** — only for properties/methods that parents need
6. **Avoid `querySelector` when possible** — use data binding and refs instead
7. **Lazy load data** — don't fetch everything on connectedCallback
8. **Use `lightning-record-*` base components** for standard CRUD forms
9. **CSS custom properties** for theming instead of hardcoded values
10. **Write Jest tests** for component logic

### Performance Tips
- Use `cacheable=true` on Apex methods that only read data
- Use `lightning-datatable` for large lists (virtual scrolling built-in)
- Debounce search inputs (300ms+)
- Use loading spinners for async operations

---

## Spring '26 Updates

### Complex Template Expressions (Beta, API 66.0)
```html
<!-- Before: needed a getter for every computed value -->
<!-- After: inline expressions in templates -->
<template>
    <p>{item.price * item.quantity}</p>
    <template lwc:if={items.length > 0}>
        <p>Total: {items.reduce((sum, i) => sum + i.price, 0)}</p>
    </template>
</template>
```
Note: This is beta — use getters for production code until GA.

### Navigate to Flow (GA)
Direct navigation to screen flows from LWC using `standard__flow` page reference type.

### Error Console
Non-fatal errors now logged in background Error Console instead of red box popups.
Enable: Setup → User Interface → "Use Error Console for error reporting in Lightning Experience"

### GraphQL Mutations
LWC can now use GraphQL API for mutations (create, update, delete) in addition to queries,
reducing the need for Apex in simple CRUD scenarios.
