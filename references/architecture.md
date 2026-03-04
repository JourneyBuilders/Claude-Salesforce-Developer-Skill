# Salesforce Architecture Guide (architect.salesforce.com)

## Table of Contents
1. [Well-Architected Framework](#well-architected-framework)
2. [Architect Decision Guides](#architect-decision-guides)
3. [Record-Triggered Automation Decision Guide](#record-triggered-automation)
4. [Async Processing Decision Guide](#async-processing)
5. [Building Forms Decision Guide](#building-forms)
6. [Event-Driven Architecture Decision Guide](#event-driven-architecture)
7. [Integration Patterns](#integration-patterns)
8. [Data Architecture](#data-architecture)
9. [Agentforce Architecture](#agentforce-architecture)
10. [Reference Diagrams](#reference-diagrams)

Source: architect.salesforce.com — the authoritative Salesforce resource for architecture
decisions, patterns, anti-patterns, and reference architectures.

---

## Well-Architected Framework

The Salesforce Well-Architected Framework organizes guidance around three pillars:
**Trusted**, **Easy**, and **Adaptable**. Use these as compass points for every design decision.

### Trusted (solutions that protect stakeholders)
Solutions must be secure, compliant, and reliable.

**Secure**: Enforce least-privilege access. Use permission sets over profiles. Apply CRUD/FLS
in every query (`WITH USER_MODE`). Use Named Credentials for all external auth. Never
hardcode credentials or session IDs.

**Compliant**: Follow data residency requirements. Implement field-level encryption (Shield)
where needed. Audit trail for sensitive operations. Consent management for PII.

**Reliable**: Design for governor limits from day one. Handle errors gracefully. Implement
retry logic for integrations. Monitor with custom error logging (e.g., Nebula Logger).
Use circuit breaker patterns for external dependencies.

### Easy (solutions that deliver value fast)
Solutions should be intentional, automated, and engaging.

**Intentional**: Solve the actual business problem. Use standard objects and features before
custom. Every customization should trace back to a requirement. Document architectural
decisions and their rationale (Architecture Decision Records).

**Automated**: Prefer declarative automation (Flows) for maintainability. Use clicks-before-code
as the default. Automate testing and deployment (CI/CD). Use scheduled flows and batch Apex
for recurring operations.

**Engaging**: Design for the user. Lightning App Builder for personalized pages. Dynamic Forms
for contextual field display. In-app guidance for adoption. Optimize for mobile if applicable.

### Adaptable (solutions that evolve with the business)
Solutions should be resilient, composable, and scalable.

**Resilient**: Build for change. Use Custom Metadata Types over hardcoded values. Abstract
integrations behind service layers. Design triggers and automations to handle bulk operations.

**Composable**: Build modular solutions. Separation of concerns (trigger handler pattern,
service layer, selector layer). Use unlocked packages for modular deployments. APIs for
everything that might need to integrate.

**Scalable**: Design data models for growth. Index critical query fields. Use async processing
for heavy workloads. Paginate results. Test with realistic data volumes (200+ records minimum).

### Evaluating Design Decisions
For every architectural choice, ask:
1. **Is it Trusted?** Secure, compliant, reliable?
2. **Is it Easy?** Intentional, automated, engaging?
3. **Is it Adaptable?** Resilient, composable, scalable?

Also apply the DVF lens:
- **Desirable**: Does the user want it?
- **Viable**: Does the business support it?
- **Feasible**: Can the technology deliver it?

---

## Architect Decision Guides

architect.salesforce.com publishes decision guides for choosing the right tool or pattern.
Always consult the relevant guide when facing an architectural choice. Key guides:

### Platform Decision Guides
| Guide | Use When |
|-------|----------|
| Record-Triggered Automation | Choosing between Flow, Apex, or other automation |
| Asynchronous Processing | Selecting async pattern (Queueable, Batch, etc.) |
| Building Forms | Choosing form technology (Dynamic Forms, Screen Flow, LWC, OmniStudio) |
| Event-Driven Architecture | Designing event/message-based systems |
| Step-Based Async Framework | Complex multi-step async processing |
| Migrating Changes Between Environments | Deployment strategy selection |

### Data and Integration Decision Guides
| Guide | Use When |
|-------|----------|
| Data Integration | Choosing sync vs async, replication vs virtualization |
| Data 360 Interoperability | Data Cloud ingestion vs Zero Copy federation |
| Data 360 Provisioning | Multi-org Data Cloud architecture |

### Agentforce Decision Guides
| Guide | Use When |
|-------|----------|
| Agentforce Architecture | Designing AI agent topology and patterns |
| Agent Development Lifecycle | Planning agent build, test, deploy workflow |

---

## Record-Triggered Automation

### Decision Framework

**Use Before-Save Flow when:**
- Updating fields on the SAME record that triggered the flow
- No need to create/update/delete OTHER records
- Fast field updates, default values, simple validations
- Performance: fastest option (no extra DML)

**Use After-Save Flow when:**
- Creating, updating, or deleting OTHER records
- Sending email notifications or platform events
- Calling invocable Apex for complex processing
- Need access to record ID (which is set after insert)

**Use Apex Trigger when:**
- Complex business logic with many conditions
- Operations requiring precise transaction control
- Performance-critical processing on high-volume objects
- Complex cross-object validation or processing
- Need for sophisticated error handling
- Operations that must handle 200+ record batches efficiently

### Combining Flow and Apex
Best practice: Use Flow for orchestration and routing, Apex (via Invocable) for complex
computation. This gives admins visibility into the flow while keeping heavy logic in testable Apex.

```
Record Change → Before-Save Flow (field defaults)
             → Apex Before Trigger (complex validation)
             → After-Save Flow (related record updates)
             → Apex After Trigger (async callouts via Queueable)
```

### Order of Execution (Know This!)
1. System validation rules
2. Before triggers (Apex)
3. System validation rules (again)
4. Custom validation rules
5. Duplicate rules
6. Before-save record-triggered flows
7. After triggers (Apex)
8. Assignment rules
9. Auto-response rules
10. Workflow rules (legacy)
11. After-save record-triggered flows
12. Entitlement rules
13. Roll-up summary fields
14. Cross-object workflow updates
15. Criteria-based sharing recalculation

---

## Async Processing

### Selection Matrix

| Pattern | Max Records | Chaining | Callouts | State | Use Case |
|---------|------------|----------|----------|-------|----------|
| **Queueable** | ~50k/job | Yes (1 child) | Yes (AllowsCallouts) | Yes (via members) | Default async choice |
| **Batch Apex** | Millions | No (but sequential) | Yes (AllowsCallouts) | Yes (Stateful) | Large data processing |
| **@future** | N/A | No | Yes | No | Simple one-off async (legacy) |
| **Scheduled Apex** | Depends | Via Batch/Queue | Depends | Depends | Recurring timed jobs |
| **Platform Event Trigger** | Per event | N/A | No direct | No | Event-driven decoupling |
| **Scheduled Flow** | ~50k | Via actions | Via invocable | Via variables | Admin-friendly batch |
| **Apex Cursors** (Spring '26) | 50M rows | N/A | N/A | Cursor state | Paginated large queries |

### Architect Recommendation (from architect.salesforce.com)
1. **Default to Queueable** for most async needs — replaces @future in all cases
2. **Use Batch Apex** only when processing millions of records or need start/execute/finish
3. **Use Platform Events** when you need to decouple processes or cross transaction boundaries
4. **Use Scheduled Apex/Flow** for time-based recurring operations
5. **Avoid @future** — Queueable does everything @future does, plus supports complex types
   and chaining

---

## Building Forms

### Selection Matrix

| Tool | Complexity | Skills | When to Use |
|------|-----------|--------|-------------|
| **Dynamic Forms** | Low | Admin | Simple record pages, conditional field visibility |
| **Screen Flow** | Medium | Admin | Multi-step wizards, guided data entry |
| **Screen Flow + LWC** | Medium-High | Admin + Dev | Flow navigation with custom UI components |
| **LWC** | High | Developer | Full custom UI, complex interactions, performance-critical |
| **OmniStudio** | High | Specialist | Industry-specific, complex guided processes |

### Decision Process
1. Can Dynamic Forms solve it? → Use Dynamic Forms
2. Does it need multi-step or guided input? → Screen Flow
3. Does it need custom UI within a flow? → Screen Flow + LWC
4. Does it need full custom control? → Pure LWC
5. Is it an industry-specific guided process? → OmniStudio

---

## Event-Driven Architecture

### When to Use Events
- Processes that don't require synchronous responses
- Decoupling systems that shouldn't be tightly integrated
- Fan-out patterns (one event, multiple consumers)
- Cross-org communication
- Near-real-time data synchronization

### When NOT to Use Events
- User is waiting for a response (use synchronous instead)
- Simple record updates (use Flow or Trigger)
- Data requires guaranteed ordering
- Process requires immediate consistency

### Eventing Tools

| Tool | Direction | Use Case |
|------|-----------|----------|
| **Platform Events** | Publish from Salesforce/external | Custom event messaging within and across Salesforce |
| **Change Data Capture (CDC)** | From Salesforce | Track and react to record changes externally |
| **Pub/Sub API** | Bidirectional | Modern subscribe/publish for external systems (replaces Streaming API) |
| **MuleSoft Anypoint** | Bidirectional | Enterprise event bus, complex routing, transformation |

### Best Practices (from architect.salesforce.com)
- Use **Pub/Sub API** for all new publish/subscribe patterns (not Streaming API)
- Use **Platform Events** over CDC when you need custom event structures
- Document all events, triggers, and downstream consumers
- Use consistent naming conventions across all systems
- Design for **idempotency** — consumers must handle duplicate events
- Implement **dead letter queues** for failed event processing
- Add logic to prevent endless loops (event triggers event triggers event...)
- Consider event allocation limits when using platform events internally

---

## Integration Patterns

From architect.salesforce.com's integration guidance:

### Pattern Selection

| Pattern | When to Use |
|---------|-------------|
| **Request-Reply (Sync)** | User needs immediate response. Small data volumes. |
| **Fire-and-Forget (Async)** | No response needed. External system processes later. |
| **Batch Data Sync** | Large data volumes. Scheduled intervals. |
| **Remote Call-In** | External system pushes data to Salesforce. |
| **Data Virtualization** | Display external data without storing. Real-time lookup. |
| **Event-Driven** | Loosely coupled. Near-real-time. Multiple consumers. |

### Authentication Decision
- **JWT Bearer Token**: Server-to-server, CI/CD, ETL (no user interaction)
- **Authorization Code + PKCE**: Mobile apps, SPAs (no secure backend)
- **Web Server (Auth Code + Secret)**: Web apps with secure backend
- **SAML Bearer**: Leverage existing SSO (Okta, Azure AD)
- Avoid Username-Password flow (legacy, insecure)

### Named Credentials (Always Use)
- Centralized endpoint and auth management
- No credentials in code or config
- External Credentials (modern) for new integrations
- Supports OAuth, JWT, custom auth, AWS Signature

---

## Data Architecture

### Data Model Best Practices
- Use **standard objects** before creating custom ones
- Use **lookup relationships** for flexibility, **master-detail** for roll-ups and cascade delete
- Avoid excessive custom objects — consider if Custom Metadata or Big Objects fit better
- Index fields used in WHERE clauses (External ID, Unique, or request via Support)
- Plan for **Large Data Volumes (LDV)**: skinny tables, archive strategies, indexed queries
- Use **Record Types** over boolean fields for categorization

### Multi-Org Architecture
- **Single Org**: Default choice. Simpler governance, lower cost.
- **Multi-Org**: Only when required by regulation, acquisition, or extreme process divergence.
- **Data Cloud One**: For multi-org environments, unify data in a single Data 360 Home Org
  with Companion Orgs consuming shared data.

---

## Agentforce Architecture

### Agent Types (from architect.salesforce.com)
1. **Conversational**: Chat/voice interaction with users
2. **Proactive**: Triggered by events or conditions, acts without prompting
3. **Ambient**: Monitors context and surfaces relevant information
4. **Autonomous**: Operates independently with minimal human oversight
5. **Collaborative**: Multiple agents working together (Agent-to-Agent / A2A)

### Agent Development Lifecycle (ADLC)
1. **Ideation & Design**: Define agent purpose, topics, actions, guardrails
2. **Development (Inner Loop)**: Build topics, instructions, actions (Flow/Apex)
3. **Testing & Validation**: Agent test specs, Testing Center, utterance testing
4. **Deployment**: Package agent metadata, deploy via CLI or change sets
5. **Monitoring & Tuning (Outer Loop)**: Analyze conversations, refine instructions

### Agent Script (Spring '26)
Declarative YAML-based DSL for structuring agent behavior. Provides deterministic
guardrails around LLM reasoning — balances AI flexibility with enterprise predictability.

### Building Agents via CLI
```bash
# Generate agent spec
sf agent generate spec --output-dir config/agents

# Create agent from spec
sf agent create --spec-file config/agents/my-agent.yaml \
  --name "My Agent" --target-org myorg

# Test agent
sf agent test create --spec config/agents/test-spec.yaml --target-org myorg
sf agent test run --name MyAgentTest --target-org myorg

# Preview agent
sf agent preview --api-name My_Agent --target-org myorg
```

---

## Reference Diagrams

architect.salesforce.com provides 13+ solution architecture reference diagrams. When
designing complex solutions, reference these for proven patterns:

- **Sales Cloud Architecture**: Lead-to-cash process design
- **Service Cloud Architecture**: Case management, knowledge, omni-channel
- **Experience Cloud Architecture**: Portal, community, headless patterns
- **Integration Architecture**: Hub-and-spoke, point-to-point, event bus
- **Data Architecture**: LDV strategies, archiving, data lifecycle
- **Identity & Access Management**: SSO, OAuth flows, MFA patterns
- **Mobile Architecture**: Salesforce Mobile, custom apps, offline patterns
- **Analytics Architecture**: Reports, dashboards, CRM Analytics, Data Cloud
- **Agentforce Architecture**: Agent topology, A2A communication, MCP integration

### Where to Find Them
architect.salesforce.com/fundamentals — Whitepapers and deep architectural guidance
architect.salesforce.com/decision-guides — Tool and pattern selection frameworks
architect.salesforce.com/well-architected — Patterns, anti-patterns, and the Well-Architected explorer
architect.salesforce.com/well-architected/explorer — Searchable pattern and anti-pattern database

---

## Architecture Anti-Patterns (from Well-Architected Explorer)

### Common Anti-Patterns to Avoid
| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| God Object | One custom object stores everything | Normalize into related objects |
| Point-to-point Integration Spaghetti | Every system directly connected | Use middleware / event bus |
| Org Sprawl | Too many orgs without governance | Consolidate with multi-org strategy |
| Monolithic Trigger | All logic in one massive trigger | Trigger handler framework + services |
| Hardcoded Config | IDs, URLs, settings in Apex | Custom Metadata Types, Named Credentials |
| Over-Customization | Custom everything when standard works | Standard objects + configuration first |
| No Error Strategy | Silent failures, no logging | Centralized error logging, alerts |
| Sync-Everything Integration | Real-time sync when batch would suffice | Match pattern to requirement |
| Profile-Based Permissions | Complex profile matrix | Minimum profile + Permission Sets |
| Test Data in Production | Using real data for testing | Synthetic test data, proper sandboxes |
