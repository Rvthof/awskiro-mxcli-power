# OData Data Sharing Between Mendix Apps

This skill covers how to use OData services to share data between Mendix applications, with emphasis on using view entities as an abstraction layer to decouple the API contract from the internal domain model.

## When to Use This Skill

- User asks to expose data from one Mendix app to another
- User wants to set up inter-app communication via OData
- User needs to create an API layer that abstracts internal entities
- User asks about external entities, consumed/published OData services
- User wants to decouple modules or apps for independent deployment
- User asks about the view entity pattern for OData services

## Architecture Overview

OData data sharing follows a **producer/consumer** pattern with three layers:

```
┌─────────────────────────────────────────────┐
│  PRODUCER APP                               │
│                                             │
│  Persistent Entities  ──▶  View Entities    │
│  (Shop.Customer,          (Api.CustomerVE)  │
│   Shop.Address)                             │
│                          ▼                  │
│                    OData Service             │
│                   (Api.CustomerApi)          │
└──────────────────────┬──────────────────────┘
                       │ HTTP/OData4
┌──────────────────────▼──────────────────────┐
│  CONSUMER APP                               │
│                                             │
│                    OData Client             │
│                  (Client.CustomerApiClient)  │
│                          ▼                  │
│                  External Entities           │
│                 (Client.CustomersEE)         │
│                          ▼                  │
│                  Pages & Microflows          │
└─────────────────────────────────────────────┘
```

### Why View Entities?

Publishing persistent entities directly exposes your internal schema. When you change a column name, add a table, or restructure associations, every consumer app breaks. **View entities** solve this:

1. **Stable API contract** -- the view's shape stays the same even when the underlying tables change
2. **Flattened data** -- joins across multiple tables into a single flat resource (e.g., Customer + BillingAddress + DeliveryAddress into one `CustomerAddressVE`)
3. **Computed fields** -- add calculated columns like `FullAddress` or `ActivePrice` using OQL expressions
4. **Filtered datasets** -- restrict what's visible (e.g., only active products, cheap products)
5. **Aggregations** -- expose pre-aggregated metrics (e.g., orders per day, sum of line items)

## Step-by-Step: Read-Only API with View Abstraction

### Step 1: Create the Producer Module and Role

```sql
CREATE MODULE ProductApi;

CREATE MODULE ROLE ProductApi.ApiUser
  DESCRIPTION 'Role for OData API access';
```

### Step 2: Create View Entities as the API Layer

Instead of publishing `Shop.Product` and `Shop.Price` directly, create a view that joins and flattens them:

```sql
/**
 * Flattened product with current active price.
 * Joins Product with the most recent Price entry.
 */
CREATE VIEW ENTITY ProductApi.ProductWithPriceVE (
  ProductId: Integer,
  Name: String,
  Description: String,
  PriceInEuro: Decimal
) AS (
  select p.ID         as ProdId
  ,      p.ProductId  as ProductId
  ,      p.Name       as Name
  ,      p.Description as Description
  ,      ( select pr.PriceInEuro
           from   Shop.Price as pr
           where  pr.StartDate <= '[%BeginOfTomorrow%]'
           and    pr/Shop.Price_Product = p.ID
           order  by pr.StartDate desc
           limit  1
         ) as PriceInEuro
  from   Shop.Product as p
  where  p.IsActive
);

GRANT ProductApi.ApiUser ON ProductApi.ProductWithPriceVE
  (READ *, WRITE *);
```

For aggregated data:

```sql
/**
 * Daily sales totals for cheap products.
 */
CREATE VIEW ENTITY ProductApi.CheapProductSalesVE (
  OrderDate: DateTime,
  TotalItems: Long
) AS (
  select o.OrderDate     as OrderDate
  ,      sum(ol.Amount)  as TotalItems
  from   Shop.OrderLine as ol
    left join Shop.OrderLine_Order/Shop."Order" as o
  where  ol/Shop.OrderLine_Product/Shop.Product.PriceInEuro < 100
  group by o.OrderDate
  order by o.OrderDate desc
  limit 1000
);

GRANT ProductApi.ApiUser ON ProductApi.CheapProductSalesVE
  (READ *, WRITE *);
```

For flattening across associations:

```sql
/**
 * Customer with billing and delivery address flattened into one resource.
 */
CREATE VIEW ENTITY ProductApi.CustomerAddressVE (
  CustomerId: Long,
  CustomerName: String,
  Email: String,
  BillingStreet: String,
  BillingCity: String,
  BillingCountry: String,
  DeliveryStreet: String,
  DeliveryCity: String,
  DeliveryCountry: String
) AS (
  select c.ID                              as CustomerID
  ,      c.CustomerId                      as CustomerId
  ,      c.FirstName + ' ' + c.LastName    as CustomerName
  ,      c.EmailAddress                    as Email
  ,      ba.Streetname                     as BillingStreet
  ,      ba.City                           as BillingCity
  ,      ba.Country                        as BillingCountry
  ,      da.Streetname                     as DeliveryStreet
  ,      da.City                           as DeliveryCity
  ,      da.Country                        as DeliveryCountry
  from   Shop.Customer as c
    left outer join c/Shop.BillingAddress_Customer/Shop.Address as ba
    left outer join c/Shop.DeliveryAddress_Customer/Shop.Address as da
);

GRANT ProductApi.ApiUser ON ProductApi.CustomerAddressVE
  (READ *, WRITE *);
```

### Step 3: Publish the OData Service

```sql
/**
 * Product and customer data API.
 * Exposes flattened views for external consumers.
 */
CREATE ODATA SERVICE ProductApi.ProductDataApi (
  Path: 'odata/productdataapi/v1/',
  Version: '1.0.0',
  ODataVersion: OData4,
  Namespace: 'DefaultNamespace',
  ServiceName: 'ProductDataApi',
  Summary: 'Product and customer data API',
  PublishAssociations: No
)
AUTHENTICATION Basic
{
  PUBLISH ENTITY ProductApi.ProductWithPriceVE AS 'Product' (
    ReadMode: ReadFromDatabase,
    InsertMode: NotSupported,
    UpdateMode: NotSupported,
    DeleteMode: NotSupported
  )
  EXPOSE (
    ProductId AS 'ProductId' (Filterable, Sortable, Key),
    Name AS 'Name' (Filterable, Sortable),
    Description AS 'Description' (Filterable, Sortable),
    PriceInEuro AS 'PriceInEuro' (Filterable, Sortable)
  );

  PUBLISH ENTITY ProductApi.CustomerAddressVE AS 'CustomerAddress' (
    ReadMode: ReadFromDatabase,
    InsertMode: NotSupported,
    UpdateMode: NotSupported,
    DeleteMode: NotSupported
  )
  EXPOSE (
    CustomerId AS 'CustomerId' (Filterable, Sortable, Key),
    CustomerName AS 'CustomerName' (Filterable, Sortable),
    Email AS 'Email' (Filterable, Sortable),
    BillingStreet AS 'BillingStreet' (Filterable, Sortable),
    BillingCity AS 'BillingCity' (Filterable, Sortable),
    BillingCountry AS 'BillingCountry' (Filterable, Sortable),
    DeliveryStreet AS 'DeliveryStreet' (Filterable, Sortable),
    DeliveryCity AS 'DeliveryCity' (Filterable, Sortable),
    DeliveryCountry AS 'DeliveryCountry' (Filterable, Sortable)
  );
};

GRANT ACCESS ON ODATA SERVICE ProductApi.ProductDataApi
  TO ProductApi.ApiUser;
```

### Step 4: Set Up the Consumer App

In the consuming application, create an OData client and external entities:

```sql
CREATE MODULE ProductClient;

CREATE MODULE ROLE ProductClient.User;

-- Location constant (configure per environment)
CREATE CONSTANT ProductClient.ProductDataApiLocation
  TYPE String
  DEFAULT 'http://localhost:8080/odata/productdataapi/v1/';

-- OData client connection
CREATE ODATA CLIENT ProductClient.ProductDataApiClient (
  ODataVersion: OData4,
  MetadataUrl: 'http://localhost:8080/odata/productdataapi/v1/$metadata',
  Timeout: 300,
  ServiceUrl: '@ProductClient.ProductDataApiLocation',
  UseAuthentication: Yes,
  HttpUsername: 'MxAdmin',
  HttpPassword: '1'
);

-- External entities (mapped from published service)
CREATE EXTERNAL ENTITY ProductClient.ProductsEE
FROM ODATA CLIENT ProductClient.ProductDataApiClient
(
  EntitySet: 'Product',
  RemoteName: 'Product',
  Countable: Yes
)
(
  ProductId: Long,
  Name: String,
  Description: String,
  PriceInEuro: Decimal
);

GRANT ProductClient.User ON ProductClient.ProductsEE (READ *);

CREATE EXTERNAL ENTITY ProductClient.CustomerAddressesEE
FROM ODATA CLIENT ProductClient.ProductDataApiClient
(
  EntitySet: 'CustomerAddress',
  RemoteName: 'CustomerAddress',
  Countable: Yes
)
(
  CustomerId: Long,
  CustomerName: String,
  Email: String,
  BillingStreet: String,
  BillingCity: String,
  BillingCountry: String,
  DeliveryStreet: String,
  DeliveryCity: String,
  DeliveryCountry: String
);

GRANT ProductClient.User ON ProductClient.CustomerAddressesEE (READ *);
```

**Bulk alternative:** Instead of creating external entities one by one, import all (or a subset) from the contract:

```sql
-- All entities from the service
CREATE EXTERNAL ENTITIES FROM ProductClient.ProductDataApiClient;

-- Or specific ones only
CREATE EXTERNAL ENTITIES FROM ProductClient.ProductDataApiClient
  ENTITIES (Product, CustomerAddress);

-- Idempotent re-import
CREATE OR MODIFY EXTERNAL ENTITIES FROM ProductClient.ProductDataApiClient;
```

## Step-by-Step: Read-Write API with Microflow Handlers

For write operations (insert, update, delete), the OData service delegates to microflows that map between the view entity and the underlying persistent entities.

### Step 1: Create CUD Microflows on the Producer

Each microflow receives the view entity and an `$HttpRequest` parameter:

```sql
/**
 * Handles INSERT on ProductWithPriceVE.
 * Creates a new Product and initial Price entry.
 */
CREATE MICROFLOW ProductApi.InsertProductWithPriceVE (
  $ProductWithPriceVE: ProductApi.ProductWithPriceVE,
  $HttpRequest: System.HttpRequest
)
BEGIN
  -- Map view fields to persistent entities
  $Product = CREATE Shop.Product (
    Name = $ProductWithPriceVE/Name,
    Description = $ProductWithPriceVE/Description,
    IsActive = true
  );
  COMMIT $Product;

  $Price = CREATE Shop.Price (
    PriceInEuro = $ProductWithPriceVE/PriceInEuro,
    StartDate = '[%CurrentDateTime%]'
  );
  CHANGE $Price (Shop.Price_Product = $Product);
  COMMIT $Price;
END;

GRANT EXECUTE ON MICROFLOW ProductApi.InsertProductWithPriceVE
  TO ProductApi.ApiUser;

/**
 * Handles UPDATE on ProductWithPriceVE.
 * Updates the Product name/description and creates a new Price entry.
 */
CREATE MICROFLOW ProductApi.UpdateProductWithPriceVE (
  $ProductWithPriceVE: ProductApi.ProductWithPriceVE,
  $HttpRequest: System.HttpRequest
)
BEGIN
  RETRIEVE $Product FROM Shop.Product
    WHERE ProductId = $ProductWithPriceVE/ProductId
    LIMIT 1;

  CHANGE $Product (
    Name = $ProductWithPriceVE/Name,
    Description = $ProductWithPriceVE/Description
  );
  COMMIT $Product;
END;

GRANT EXECUTE ON MICROFLOW ProductApi.UpdateProductWithPriceVE
  TO ProductApi.ApiUser;

/**
 * Handles DELETE on ProductWithPriceVE.
 * Soft-deletes the product by setting IsActive = false.
 */
CREATE MICROFLOW ProductApi.DeleteProductWithPriceVE (
  $ProductWithPriceVE: ProductApi.ProductWithPriceVE,
  $HttpRequest: System.HttpRequest
)
BEGIN
  RETRIEVE $Product FROM Shop.Product
    WHERE ProductId = $ProductWithPriceVE/ProductId
    LIMIT 1;

  CHANGE $Product (IsActive = false);
  COMMIT $Product;
END;

GRANT EXECUTE ON MICROFLOW ProductApi.DeleteProductWithPriceVE
  TO ProductApi.ApiUser;
```

### Step 2: Wire Microflows to Published Entity

Set `InsertMode`, `UpdateMode`, `DeleteMode` to `CallMicroflow`:

```sql
  PUBLISH ENTITY ProductApi.ProductWithPriceVE AS 'Product' (
    ReadMode: ReadFromDatabase,
    InsertMode: MICROFLOW ProductApi.InsertProductWithPriceVE,
    UpdateMode: MICROFLOW ProductApi.UpdateProductWithPriceVE,
    DeleteMode: MICROFLOW ProductApi.DeleteProductWithPriceVE
  )
  EXPOSE (...);
```

### Step 3: Grant Write Access on External Entity

On the consumer side, grant CREATE, WRITE, and DELETE rights:

```sql
GRANT ProductClient.User ON ProductClient.ProductsEE
  (CREATE, DELETE, READ *, WRITE *);
```

The consumer can now create, update, and delete products through the OData API, and the producer's microflows handle the mapping to persistent entities.

## Advanced: Configuration Microflow for Custom Headers

When the consumer needs to pass custom headers (e.g., for audit trails or user context), use a configuration microflow:

```sql
/**
 * Adds current user name as custom header for audit logging.
 */
CREATE MICROFLOW ProductClient.SetClientHeaders (
  $httpResponse: System.HttpResponse
)
RETURNS List of System.HttpHeader AS $HttpHeaderList
BEGIN
  $HttpHeaderList = CREATE LIST of System.HttpHeader;
  $NewHttpHeader = CREATE System.HttpHeader (
    Key = 'X-Audit-User',
    Value = $currentUser/Name
  );
  ADD $NewHttpHeader TO $HttpHeaderList;
  RETURN $HttpHeaderList;
END;
```

Reference it in the client:

```sql
CREATE ODATA CLIENT ProductClient.ProductDataApiClient (
  ...
  ConfigurationMicroflow: MICROFLOW ProductClient.SetClientHeaders
);
```

## API Versioning

When your API contract changes, create a new version rather than breaking existing consumers:

```sql
-- v1: Original API (keep running for existing consumers)
CREATE ODATA SERVICE ProductApi.ProductDataApi (
  Path: 'odata/productdataapi/v1/',
  Version: '1.0.0',
  ...
);

-- v2: New version with additional fields
CREATE ODATA SERVICE ProductApi.ProductDataApi_v2 (
  Path: 'odata/productdataapi/v2/',
  Version: '2.0.0',
  ODataVersion: OData4,
  ServiceName: 'ProductDataApi',
  Summary: 'Product API v2 - includes weight and tags',
  ...
)
AUTHENTICATION Basic
{
  PUBLISH ENTITY ProductApi.ProductWithPriceAndTagsVE AS 'Product' (
    ReadMode: ReadFromDatabase,
    InsertMode: MICROFLOW ProductApi.InsertProductV2,
    UpdateMode: MICROFLOW ProductApi.UpdateProductV2,
    DeleteMode: MICROFLOW ProductApi.DeleteProductV2
  )
  EXPOSE (...);
};
```

## Folder Organization

Use the `Folder` property to organize OData documents within modules:

```sql
CREATE ODATA CLIENT ProductClient.ProductDataApiClient (
  ODataVersion: OData4,
  MetadataUrl: 'http://localhost:8080/odata/productdataapi/v1/$metadata',
  Folder: 'Integration/ProductAPI'
);

CREATE ODATA SERVICE ProductApi.ProductDataApi (
  Path: 'odata/productdataapi/v1/',
  Version: '1.0.0',
  ODataVersion: OData4,
  Folder: 'Integration/APIs'
)
AUTHENTICATION Basic
{ ... };
```

Folders are created automatically if they don't exist. Use `/` for nested folders.

## Module Organization Conventions

Follow this naming convention for clean separation:

| Module | Purpose | Contains |
|--------|---------|----------|
| `Shop` | Core domain | Persistent entities, business logic |
| `ShopApi` or `ShopViews` | API layer (producer) | View entities, OData service, CUD microflows |
| `ShopClient` or `ShopViewsClient` | API consumer | OData client, external entities, client constants |

This keeps the API contract separate from the domain logic, and the consumer separate from the producer.

## Checklist

Before publishing:
- [ ] View entities expose only the fields consumers need (no internal IDs unless needed for writes)
- [ ] View entity has at least one `Key` field for OData identity
- [ ] Module role created and granted on view entities (READ, optionally WRITE)
- [ ] OData service has AUTHENTICATION set (Basic, Session, or Microflow)
- [ ] GRANT ACCESS ON ODATA SERVICE to the API module role
- [ ] CUD microflows (if writable) accept `($ViewEntity, $HttpRequest)` parameters
- [ ] CUD microflows granted EXECUTE to the API module role

Before consuming:
- [ ] Location constant created for environment-specific URLs
- [ ] OData client points to `$metadata` URL and uses `ServiceUrl: '@Module.Constant'`
- [ ] External entities match the published exposed names and types
- [ ] Module role created and granted on external entities (READ, optionally CREATE/WRITE/DELETE)

## Exploration Commands

Use these commands to inspect existing OData setup in a project:

```sql
-- List all published and consumed services
SHOW ODATA SERVICES;
SHOW ODATA CLIENTS;

-- Inspect a specific service
DESCRIBE ODATA SERVICE ShopViews.ShopViewsApi;
DESCRIBE ODATA CLIENT ShopViewsClient.ShopViewsApiClient;

-- See external entities and view entities
SHOW ENTITIES IN ShopViewsClient;
SHOW EXTERNAL ENTITIES;
SHOW EXTERNAL ACTIONS;

-- Browse available assets from cached $metadata contract
SHOW CONTRACT ENTITIES FROM ShopViewsClient.ShopViewsApiClient;
SHOW CONTRACT ACTIONS FROM ShopViewsClient.ShopViewsApiClient;
DESCRIBE CONTRACT ENTITY ShopViewsClient.ShopViewsApiClient.Product;
DESCRIBE CONTRACT ENTITY ShopViewsClient.ShopViewsApiClient.Product FORMAT mdl;

-- Check security setup
SHOW ACCESS ON ODATA SERVICE ShopViews.ShopViewsApi;
SHOW MODULE ROLES IN ShopViews;
```
