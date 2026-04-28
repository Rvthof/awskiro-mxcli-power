# Business Events

## When to Use This Skill

Use this skill when the user wants to:
- Define event-driven APIs using Kafka/message brokers
- Create business event services that publish events
- View or describe existing business event services
- Set up publish/subscribe message channels

## Commands

### View Business Events

```sql
-- List all business event service documents
SHOW BUSINESS EVENT SERVICES;

-- Filter by module
SHOW BUSINESS EVENT SERVICES IN MyModule;

-- List all business event client documents (future)
SHOW BUSINESS EVENT CLIENTS;

-- List individual messages across all services
SHOW BUSINESS EVENTS;

-- Filter messages by module
SHOW BUSINESS EVENTS IN MyModule;

-- Full MDL description (round-trippable)
DESCRIBE BUSINESS EVENT SERVICE Module.ServiceName;
```

### Create a Business Event Service

```sql
CREATE BUSINESS EVENT SERVICE Module.CustomerEventsApi
(
  ServiceName: 'CustomerEventsApi',
  EventNamePrefix: 'com.example'
)
{
  MESSAGE CustomerChangedEvent (CustomerId: Long) PUBLISH
    ENTITY Module.PBE_CustomerChangedEvent;
  MESSAGE AddressChangedEvent (AddressId: Long) PUBLISH
    ENTITY Module.PBE_AddressChangedEvent;
};
```

### Create or Replace (Overwrite Existing)

```sql
CREATE OR REPLACE BUSINESS EVENT SERVICE Module.CustomerEventsApi
(
  ServiceName: 'CustomerEventsApi',
  EventNamePrefix: ''
)
{
  MESSAGE CustomerChangedEvent (CustomerId: Long) PUBLISH
    ENTITY Module.PBE_CustomerChangedEvent;
};
```

### Drop a Business Event Service

```sql
DROP BUSINESS EVENT SERVICE Module.CustomerEventsApi;
```

## Message Definition Syntax

```
MESSAGE <MessageName> (<AttrName>: <Type>, ...) PUBLISH|SUBSCRIBE
  [ENTITY <Module.EntityName>]
  [MICROFLOW <Module.MicroflowName>];
```

### Supported Attribute Types
- `String` - Text
- `Integer` - 32-bit integer
- `Long` - 64-bit integer
- `Decimal` - Precise decimal number
- `Boolean` - True/false
- `DateTime` - Date and time

## Service Properties

| Property | Description |
|----------|-------------|
| `ServiceName` | The service name used in the event broker |
| `EventNamePrefix` | Prefix added to event names (can be empty) |
| `Folder` | Optional folder path for the service document |

## Operations

| Operation | Description |
|-----------|-------------|
| `PUBLISH` | This service publishes the event (other apps subscribe) |
| `SUBSCRIBE` | This service subscribes to the event (other apps publish) |

## Publishing Events from Microflows

There is no dedicated microflow activity for publishing business events. Instead, Mendix
provides Java actions in the `BusinessEvents` marketplace module. Use `CALL JAVA ACTION`
to publish an event from a microflow:

```sql
-- Create an event entity instance and publish it
CREATE MICROFLOW Module.ACT_PublishCustomerChanged
  FOLDER 'ACT'
BEGIN
  DECLARE $Event Module.PBE_CustomerChangedEvent;
  $Event = CREATE Module.PBE_CustomerChangedEvent (CustomerId = $CustomerId);
  COMMIT $Event;
  CALL JAVA ACTION BusinessEvents.PublishBusinessEvent_V2(EventObject = $Event);
END;
```

### Available Java Actions (from BusinessEvents module)

| Java Action | Description |
|-------------|-------------|
| `BusinessEvents.PublishBusinessEvent_V2` | Publish an event (recommended) |
| `BusinessEvents.PublishBusinessEvent` | Publish an event (legacy) |
| `BusinessEvents.ConsumeBusinessEvent` | Consume/acknowledge an event |
| `BusinessEvents.PublishEvents` | Publish multiple events |
| `BusinessEvents.StartupBusinessEvents` | Initialize the event broker connection |
| `BusinessEvents.ShutdownBusinessEvents` | Shut down the event broker connection |

### Typical Pattern

1. Define a Business Event Service with PUBLISH messages
2. Create entities prefixed with `PBE_` that **extend `BusinessEvents.PublishedBusinessEvent`**
3. Entity attributes must **exactly match** the message attributes (no extra attributes on the entity)
4. In microflows, create an instance of the event entity, populate its attributes, commit it
5. Call `BusinessEvents.PublishBusinessEvent_V2` passing the entity instance
6. For subscribe operations, link a handler microflow in the service definition

## Checklist

- [ ] Linked entities must exist before creating the service
- [ ] **Entities must extend `BusinessEvents.PublishedBusinessEvent`** (for published events)
- [ ] **Entity attributes must exactly match the message attributes** (no extra/missing attributes)
- [ ] Entity names use qualified format: `Module.EntityName`
- [ ] Entities for published events are conventionally prefixed with `PBE_`
- [ ] Attribute types must be valid (String, Integer, Long, Decimal, Boolean, DateTime)
- [ ] Use `DESCRIBE BUSINESS EVENT SERVICE` to verify the result
- [ ] The DESCRIBE output is parseable and can be used as a CREATE statement
- [ ] To publish events from microflows, use `CALL JAVA ACTION BusinessEvents.PublishBusinessEvent_V2`
- [ ] The `BusinessEvents` module must be included in the project (marketplace module)
