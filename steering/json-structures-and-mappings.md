# JSON Structures, Import Mappings & Export Mappings

This skill covers creating and managing JSON structures, import mappings, and export mappings in Mendix using MDL.

## Key Concepts

### JSON Structures
A JSON structure defines the schema of a JSON payload. It stores a JSON snippet and auto-derives an element tree with paths, types, and custom names.

### Import Mappings
An import mapping converts a JSON string into Mendix entity objects. It maps JSON fields to entity attributes.

### Export Mappings
An export mapping converts Mendix entity objects into a JSON string. It maps entity attributes to JSON fields.

### Critical: Import and Export Need Different Domain Models

**Import and export mappings for the same JSON structure typically require different entity structures.**

- **Import**: The child entity owns the FK to the parent (`FROM Child TO Parent`). Arrays map directly to the item entity — no intermediate container entity needed.
- **Export**: The domain model mirrors the JSON structure. Arrays need an intermediate container entity (e.g., `Items`) plus an item entity (e.g., `ItemsItem`). The container links to the parent, the item links to the container.

---

## JSON Structures

### Create

```sql
CREATE JSON STRUCTURE Module.JSON_Pet
  SNIPPET '{"id": 1, "name": "Fido", "status": "available"}';
```

For multi-line JSON, use dollar-quoting:
```sql
CREATE JSON STRUCTURE Module.JSON_Order
  SNIPPET $${
  "orderId": 100,
  "customer": {"name": "Alice", "email": "alice@example.com"},
  "items": [{"sku": "A1", "quantity": 2, "price": 9.99}]
}$$;
```

Custom name mapping (rename JSON fields):
```sql
CREATE JSON STRUCTURE Module.JSON_Pet
  SNIPPET '{"id": 1, "name": "Fido"}'
  CUSTOM NAME MAP ('id' AS '_id');
```

### Browse

```sql
SHOW JSON STRUCTURES;
SHOW JSON STRUCTURES IN Module;
DESCRIBE JSON STRUCTURE Module.JSON_Pet;
DROP JSON STRUCTURE Module.JSON_Pet;
```

---

## Import Mappings

### Domain Model for Import

For import mappings, associations point FROM the child entity TO the parent:

```sql
CREATE NON-PERSISTENT ENTITY Module.OrderResponse (
  OrderId: Integer
);
/

CREATE NON-PERSISTENT ENTITY Module.CustomerInfo (
  Name: String,
  Email: String
);
/

CREATE NON-PERSISTENT ENTITY Module.OrderItem (
  Sku: String,
  Quantity: Integer,
  Price: Decimal
);
/

-- Child entity owns the FK (FROM child TO parent)
CREATE ASSOCIATION Module.CustomerInfo_OrderResponse
  FROM Module.CustomerInfo
  TO Module.OrderResponse;
/

CREATE ASSOCIATION Module.OrderItem_OrderResponse
  FROM Module.OrderItem
  TO Module.OrderResponse;
/
```

### Simple Import Mapping (flat JSON)

```sql
CREATE IMPORT MAPPING Module.IMM_Pet
  WITH JSON STRUCTURE Module.JSON_Pet
{
  CREATE Module.PetResponse {
    PetId = id,
    Name = name,
    Status = status
  }
};
```

### Nested Import Mapping (objects and arrays)

Arrays map directly to the item entity — no intermediate container needed:

```sql
CREATE IMPORT MAPPING Module.IMM_Order
  WITH JSON STRUCTURE Module.JSON_Order
{
  CREATE Module.OrderResponse {
    OrderId = orderId,
    CREATE Module.CustomerInfo_OrderResponse/Module.CustomerInfo = customer {
      Name = name,
      Email = email
    },
    CREATE Module.OrderItem_OrderResponse/Module.OrderItem = items {
      Sku = sku,
      Quantity = quantity,
      Price = price
    }
  }
};
```

### Object Handling

| Syntax | Meaning |
|--------|---------|
| `CREATE Module.Entity` | Always create a new object (default) |
| `FIND Module.Entity` | Find by KEY attributes, ignore if not found |
| `FIND OR CREATE Module.Entity` | Find by KEY, create if not found |

```sql
CREATE IMPORT MAPPING Module.IMM_UpsertPet
  WITH JSON STRUCTURE Module.JSON_Pet
{
  FIND OR CREATE Module.PetResponse {
    PetId = id KEY,
    Name = name,
    Status = status
  }
};
```

**Note**: `KEY` is only valid with `FIND` or `FIND OR CREATE`, not with `CREATE`.

---

## Export Mappings

### Domain Model for Export

Export mappings require entities that **mirror the JSON structure**. Arrays need an intermediate container entity:

```sql
-- Root entity (matches top-level JSON object)
CREATE NON-PERSISTENT ENTITY Module.ExRoot (
  OrderId: Integer
);
/

-- Nested object entity (1-1 relationship, use OWNER Both)
CREATE NON-PERSISTENT ENTITY Module.ExCustomer (
  Name: String,
  Email: String
);
/

-- Array CONTAINER entity (no attributes, just links parent to items)
CREATE NON-PERSISTENT ENTITY Module.ExItems;
/

-- Array ITEM entity (attributes for each array element)
CREATE NON-PERSISTENT ENTITY Module.ExItemsItem (
  Sku: String,
  Quantity: Integer,
  Price: Decimal
);
/

-- Associations: child FROM, parent TO
CREATE ASSOCIATION Module.ExCustomer_ExRoot
  FROM Module.ExCustomer
  TO Module.ExRoot
  OWNER Both;   -- 1-1 for nested objects
/

CREATE ASSOCIATION Module.ExItems_ExRoot
  FROM Module.ExItems
  TO Module.ExRoot;   -- 1-* for arrays
/

CREATE ASSOCIATION Module.ExItemsItem_ExItems
  FROM Module.ExItemsItem
  TO Module.ExItems;   -- 1-* for array items
/
```

### Simple Export Mapping (flat JSON)

```sql
CREATE EXPORT MAPPING Module.EMM_Pet
  WITH JSON STRUCTURE Module.JSON_Pet
{
  Module.PetResponse {
    id = PetId,
    name = Name,
    status = Status
  }
};
```

### Nested Export Mapping (objects and arrays)

Arrays have TWO levels: container entity + item entity:

```sql
CREATE EXPORT MAPPING Module.EMM_Order
  WITH JSON STRUCTURE Module.JSON_Order
{
  Module.ExRoot {
    orderId = OrderId,
    Module.ExCustomer_ExRoot/Module.ExCustomer AS customer {
      name = Name,
      email = Email
    },
    Module.ExItems_ExRoot/Module.ExItems AS items {
      Module.ExItemsItem_ExItems/Module.ExItemsItem AS ItemsItem {
        sku = Sku,
        quantity = Quantity,
        price = Price
      }
    }
  }
};
```

### NULL VALUES option

```sql
CREATE EXPORT MAPPING Module.EMM_Pet
  WITH JSON STRUCTURE Module.JSON_Pet
  NULL VALUES SendAsNil     -- or LeaveOutElement (default)
{
  ...
};
```

---

## Microflow Actions

### Import from Mapping (JSON → entities)

```sql
-- With result variable (non-persistent entities)
$PetResponse = IMPORT FROM MAPPING Module.IMM_Pet($JsonContent);

-- Without result variable (persistent entities, stores to DB)
IMPORT FROM MAPPING Module.IMM_Pet($JsonContent);
```

### Export to Mapping (entity → JSON)

```sql
$JsonOutput = EXPORT TO MAPPING Module.EMM_Pet($PetResponse);
```

### Complete Pipeline

```sql
CREATE MICROFLOW Module.ProcessData ()
BEGIN
  DECLARE $Json String = $latestHttpResponse/Content;
  $PetResponse = IMPORT FROM MAPPING Module.IMM_Pet($Json);
  -- Process...
  $Output = EXPORT TO MAPPING Module.EMM_Pet($PetResponse);
  LOG INFO NODE 'Integration' 'Result: ' + $Output;
END;
/
```

---

## Browse

```sql
SHOW IMPORT MAPPINGS [IN Module];
SHOW EXPORT MAPPINGS [IN Module];
DESCRIBE IMPORT MAPPING Module.Name;
DESCRIBE EXPORT MAPPING Module.Name;
DROP IMPORT MAPPING Module.Name;
DROP EXPORT MAPPING Module.Name;
```

---

## Export Workflow: PE → NPE → JSON

Export mappings work on non-persistent entity (NPE) structures that mirror the target JSON. When the source data is in persistent entities (PE) in the database, the typical workflow is:

1. **Retrieve** persistent data from the database
2. **Build NPE tree** in a microflow: create NPE objects, set attributes, link via associations to match the JSON structure
3. **Export to mapping** to serialize the NPE tree to JSON

```sql
-- Example: build NPE tree from persistent Order data, then export
CREATE MICROFLOW Module.ExportOrder ($Order: Module.Order)
RETURNS String AS $Json
BEGIN
  -- Build the NPE tree matching the JSON structure
  $Root = CREATE Module.ExRoot (OrderId = $Order/OrderId);

  RETRIEVE $Customer FROM $Order/Module.Order_Customer;
  $ExCust = CREATE Module.ExCustomer (Name = $Customer/Name, Email = $Customer/Email);
  -- Link customer to root...

  -- Export
  $Json = EXPORT TO MAPPING Module.EMM_Order($Root);
  RETURN $Json;
END;
/
```

### Shortcut with View Entities

View Entities (OQL-backed) can retrieve data directly into the export-ready structure, skipping the manual NPE assembly:

```sql
CREATE VIEW ENTITY Module.ExOrderView (
  OrderId: Integer,
  CustomerName: String,
  CustomerEmail: String
) AS SELECT o.OrderId, c.Name, c.Email
   FROM Module.Order o
   JOIN Module.Order_Customer/Module.Customer c;
```

This can reduce the microflow to a single retrieve + export step.

---

## Realistic Example: Countries REST API

A complete example consuming a Countries REST API, importing the response, and
exporting country data back to JSON.

### Step 1: JSON Structures

```sql
-- Single country (flat object)
CREATE JSON STRUCTURE Integration.JSON_Country
  SNIPPET '{"name": "Netherlands", "officialName": "Kingdom of the Netherlands", "capital": "Amsterdam", "region": "Europe", "population": 18100436, "flagUrl": "https://flagcdn.com/w320/nl.png"}';

-- List of countries (array of objects)
CREATE JSON STRUCTURE Integration.JSON_CountryList
  SNIPPET '[{"name": "Netherlands", "capital": "Amsterdam", "region": "Europe", "population": 18100436}]';
```

### Step 2: Import — Single Country

```sql
CREATE NON-PERSISTENT ENTITY Integration.Country (
  Name: String,
  OfficialName: String,
  Capital: String,
  Region: String,
  Population: Integer,
  FlagUrl: String
);
/

CREATE IMPORT MAPPING Integration.IMM_Country
  WITH JSON STRUCTURE Integration.JSON_Country
{
  CREATE Integration.Country {
    Name = name,
    OfficialName = officialName,
    Capital = capital,
    Region = region,
    Population = population,
    FlagUrl = flagUrl
  }
};
```

### Step 3: Import — List of Countries

For a list response, the import mapping maps the array item directly (no container):

```sql
CREATE NON-PERSISTENT ENTITY Integration.CountryListItem (
  Name: String,
  Capital: String,
  Region: String,
  Population: Integer
);
/

CREATE IMPORT MAPPING Integration.IMM_CountryList
  WITH JSON STRUCTURE Integration.JSON_CountryList
{
  CREATE Integration.CountryListItem {
    Name = name,
    Capital = capital,
    Region = region,
    Population = population
  }
};
```

### Step 4: Export — Serialize Country to JSON

For the flat country, the same entity works for both import and export:

```sql
CREATE EXPORT MAPPING Integration.EMM_Country
  WITH JSON STRUCTURE Integration.JSON_Country
{
  Integration.Country {
    name = Name,
    officialName = OfficialName,
    capital = Capital,
    region = Region,
    population = Population,
    flagUrl = FlagUrl
  }
};
```

### Step 5: Export — List of Countries

For exporting a list, the export domain model needs a root container + item entities:

```sql
-- Container entity wrapping the array
CREATE NON-PERSISTENT ENTITY Integration.ExCountryList;
/

-- Item entity for each country in the array
CREATE NON-PERSISTENT ENTITY Integration.ExCountryItem (
  Name: String,
  Capital: String,
  Region: String,
  Population: Integer
);
/

CREATE ASSOCIATION Integration.ExCountryItem_ExCountryList
  FROM Integration.ExCountryItem
  TO Integration.ExCountryList;
/

CREATE EXPORT MAPPING Integration.EMM_CountryList
  WITH JSON STRUCTURE Integration.JSON_CountryList
{
  Integration.ExCountryList {
    Integration.ExCountryItem_ExCountryList/Integration.ExCountryItem AS Root {
      name = Name,
      capital = Capital,
      region = Region,
      population = Population
    }
  }
};
```

### Step 6: Microflow — Fetch, Import, Process, Export

```sql
CREATE MICROFLOW Integration.GetCountryInfo ()
RETURNS String AS $Json
BEGIN
  -- Fetch country data from REST API
  $Response = REST CALL GET 'https://restcountries.com/v3.1/name/netherlands'
    HEADER Accept = 'application/json'
    TIMEOUT 30
    RETURNS String
    ON ERROR CONTINUE;

  -- Import JSON into entity
  $Country = IMPORT FROM MAPPING Integration.IMM_Country($Response);

  -- Export back to our own JSON format
  $Json = EXPORT TO MAPPING Integration.EMM_Country($Country);
  LOG INFO NODE 'Integration' 'Country: ' + $Json;

  RETURN $Json;
END;
/
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Reusing import domain model for export | Export needs separate entities mirroring JSON structure |
| Association direction wrong | Always FROM child TO parent (child owns FK) |
| Using `OWNER Default` for 1-1 nested objects in export | Use `OWNER Both` for 1-1 relationships |
| Missing array container entity in export | Arrays need Container + Item entities |
| Using `KEY` with `CREATE` handling | `KEY` only valid with `FIND` or `FIND OR CREATE` |
| Arrays in import with container entity | Import arrays map directly to item entity, no container |
