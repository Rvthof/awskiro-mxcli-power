# Browse Integration Services and Contracts

This skill covers discovering external services, browsing cached contracts, and querying integration assets via the catalog.

## When to Use This Skill

- User asks what external services are configured in the project
- User wants to see available entities or actions from an OData service
- User wants to browse a business event contract (AsyncAPI)
- User asks about integration catalog tables
- User wants to find entities available in a contract but not yet imported

## Discovery: List All Services

```sql
-- All OData clients (consumed services)
SHOW ODATA CLIENTS;

-- All published OData services
SHOW ODATA SERVICES;

-- All consumed REST services
SHOW REST CLIENTS;

-- All published REST services
SHOW PUBLISHED REST SERVICES;

-- All business event services
SHOW BUSINESS EVENT SERVICES;

-- All database connections
SHOW DATABASE CONNECTIONS;

-- All external entities (imported from OData)
SHOW EXTERNAL ENTITIES;

-- All external actions used in microflows
SHOW EXTERNAL ACTIONS;
```

## Contract Browsing: OData $metadata

`CREATE ODATA CLIENT` auto-fetches and caches the `$metadata` XML. Browse it without network access:

```sql
-- List all entity types from the contract
SHOW CONTRACT ENTITIES FROM MyModule.SalesforceAPI;

-- List actions/functions
SHOW CONTRACT ACTIONS FROM MyModule.SalesforceAPI;

-- Inspect a specific entity (properties, keys, navigation)
DESCRIBE CONTRACT ENTITY MyModule.SalesforceAPI.PurchaseOrder;

-- Generate a CREATE EXTERNAL ENTITY statement from the contract
DESCRIBE CONTRACT ENTITY MyModule.SalesforceAPI.PurchaseOrder FORMAT mdl;

-- Inspect an action's signature
DESCRIBE CONTRACT ACTION MyModule.SalesforceAPI.CreateOrder;
```

## Contract Browsing: AsyncAPI (Business Events)

Business event client services cache the AsyncAPI YAML:

```sql
-- List channels
SHOW CONTRACT CHANNELS FROM MyModule.ShopEventsClient;

-- List messages with payload info
SHOW CONTRACT MESSAGES FROM MyModule.ShopEventsClient;

-- Inspect a message's payload properties
DESCRIBE CONTRACT MESSAGE MyModule.ShopEventsClient.OrderChangedEvent;
```

## Catalog Queries (requires REFRESH CATALOG)

```sql
REFRESH CATALOG;

-- All contract entities across all OData clients
SELECT ServiceQualifiedName, EntityName, EntitySetName, PropertyCount, Summary
FROM CATALOG.CONTRACT_ENTITIES;

-- All contract actions
SELECT ServiceQualifiedName, ActionName, ParameterCount, ReturnType
FROM CATALOG.CONTRACT_ACTIONS;

-- All contract messages
SELECT ServiceQualifiedName, MessageName, ChannelName, OperationType, PropertyCount
FROM CATALOG.CONTRACT_MESSAGES;

-- Find available entities NOT YET imported
SELECT ce.EntityName, ce.ServiceQualifiedName, ce.PropertyCount
FROM CATALOG.CONTRACT_ENTITIES ce
LEFT JOIN CATALOG.EXTERNAL_ENTITIES ee
  ON ce.ServiceQualifiedName = ee.ServiceName AND ce.EntityName = ee.RemoteName
WHERE ee.Id IS NULL;

-- All REST operations across all consumed services
SELECT ServiceQualifiedName, HttpMethod, Path, Name
FROM CATALOG.REST_OPERATIONS
ORDER BY ServiceQualifiedName, Path;

-- Cross-cutting: all integration services in a module
SELECT ObjectType, QualifiedName
FROM CATALOG.OBJECTS
WHERE ObjectType IN ('ODATA_CLIENT', 'REST_CLIENT', 'ODATA_SERVICE',
  'PUBLISHED_REST_SERVICE', 'BUSINESS_EVENT_SERVICE', 'DATABASE_CONNECTION')
AND ModuleName = 'Integration';
```

## Workflow: Import Entities from a Contract

### Bulk import (all or filtered)

```sql
-- Import all entity types at once
CREATE EXTERNAL ENTITIES FROM MyModule.SalesforceAPI;

-- Import into a different module
CREATE EXTERNAL ENTITIES FROM MyModule.SalesforceAPI INTO Integration;

-- Import only specific entities
CREATE EXTERNAL ENTITIES FROM MyModule.SalesforceAPI ENTITIES (PurchaseOrder, Supplier);

-- Idempotent re-import (updates existing)
CREATE OR MODIFY EXTERNAL ENTITIES FROM MyModule.SalesforceAPI;
```

### Single entity (with customization)

1. Browse available entities:
   ```sql
   SHOW CONTRACT ENTITIES FROM MyModule.SalesforceAPI;
   ```

2. Inspect the entity you want:
   ```sql
   DESCRIBE CONTRACT ENTITY MyModule.SalesforceAPI.PurchaseOrder;
   ```

3. Generate the CREATE statement:
   ```sql
   DESCRIBE CONTRACT ENTITY MyModule.SalesforceAPI.PurchaseOrder FORMAT mdl;
   ```

4. Copy, customize (remove unwanted attributes), and execute:
   ```sql
   CREATE EXTERNAL ENTITY MyModule.PurchaseOrder
   FROM ODATA CLIENT MyModule.SalesforceAPI (
       EntitySet: 'PurchaseOrders',
       RemoteName: 'PurchaseOrder',
       Countable: Yes
   )
   (
       Number: Long,
       Status: String(200),
       SupplierName: String(200),
       GrossAmount: Decimal
   );
   ```
