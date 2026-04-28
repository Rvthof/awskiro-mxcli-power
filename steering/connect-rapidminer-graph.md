# Connecting Mendix to RapidMiner / AnzoGraph via SPARQL

Use this skill when you need to fetch data from a RapidMiner graph mart (or any SPARQL 1.1 HTTP endpoint like AnzoGraph) and surface it in a Mendix app.

## When to Use

- An external graph database exposes a SPARQL HTTP endpoint with Basic Auth
- You want the graph results to become Mendix entities (for display, search, further processing)
- You have read-only needs — the pattern fits SELECT queries that return tabular results

## Endpoint shape

A RapidMiner / AnzoGraph graphmart endpoint looks like:

```
https://<host>/sparql/graphmart/<url-encoded-graphmart-uri>
```

Example:
```
https://graphstudio.mendixdemo.com/sparql/graphmart/http%3A%2F%2Fcambridgesemantics.com%2FGraphmart%2F3617250aca6a40d88972c1c0de38f86a
```

Two things to note:
1. The graphmart URI is **URL-encoded and embedded in the path** (colons and slashes become `%3A` / `%2F`).
2. SPARQL queries are sent as the **POST body** with `Content-Type: application/sparql-query`, and the response is JSON when `Accept: application/sparql-results+json`.

Verify with curl first:

```bash
curl -u 'user@example.com:password' \
  -H 'Accept: application/sparql-results+json' \
  -H 'Content-Type: application/sparql-query' \
  --data-binary 'SELECT ?s WHERE { ?s a <http://example.com/Foo> } LIMIT 10' \
  'https://host/sparql/graphmart/<encoded-uri>'
```

If curl returns `200` and a JSON `results.bindings` array, you're ready to wire it into Mendix.

## SPARQL JSON result shape

Every SPARQL HTTP result looks like this:

```json
{
  "head": { "vars": ["customer", "customerId", "customerName"] },
  "results": {
    "bindings": [
      {
        "customer":     {"type": "uri",     "value": "http://.../Customer/0000000"},
        "customerId":   {"type": "literal", "value": "CUST001"},
        "customerName": {"type": "literal", "value": "Global Tech Solutions Inc."}
      }
    ]
  }
}
```

Each row in `bindings` is an object of `{var: {type, value}}`. A JSLT transformer flattens this to something directly mappable into Mendix entities.

## The full pipeline

```
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐   ┌────────────────┐
│  Inline REST CALL  │─▶│ Data Transformer   │─▶│  Import Mapping    │─▶│ Mendix Entity  │
│  POST + Basic Auth │   │ JSLT: flatten      │   │ JSON → entities    │   │ (persistent)   │
│  SPARQL as body    │   │ results.bindings   │   │                    │   │                │
└────────────────────┘   └────────────────────┘   └────────────────────┘   └────────────────┘
```

**Why inline `REST CALL` rather than `CREATE REST CLIENT` + `SEND REST REQUEST`?**

- At the time of writing, REST Client `AUTHENTICATION: BASIC (Username: '...', Password: '...')` silently fails to attach the `Authorization` header when the password contains special characters (e.g. `!`). Result: `401 Unauthorized`.
- Inline `REST CALL ... AUTH BASIC '<user>' PASSWORD '<pass>'` handles the same credentials correctly.

**Why persistent entities for the final list?**

- Non-persistent `ReferenceSet` children can't be extracted as a `List` in MDL microflows (no documented `RETRIEVE ... BY ASSOCIATION` syntax), and `LOOP $c IN $Parent/Assoc` fails at build time.
- Persistent entities work with `DataSource: DATABASE` on a DataGrid — the standard happy path.

## Step-by-step template

### 1. Persistent target entity

```sql
@Position(100, 100)
CREATE PERSISTENT ENTITY MyModule.Customer (
  CustomerUri:  String(500),
  CustomerId:   String(50),
  CustomerName: String(200)
);
/
```

### 2. Non-persistent wrapper (for the import mapping only)

Import mappings need a single root entity. A tiny non-persistent wrapper with one dummy attribute is enough:

```sql
@Position(400, 100)
CREATE NON-PERSISTENT ENTITY MyModule.CustomerImport (
  DummyAttr: String(10)
);
/

CREATE ASSOCIATION MyModule.CustomerImport_Customer
  FROM MyModule.CustomerImport
  TO   MyModule.Customer
  TYPE ReferenceSet;
/
```

### 3. Data Transformer (JSLT) — flatten SPARQL response

Take the nested `results.bindings[].*.value` shape and emit a flat `customers[]` array:

```sql
CREATE DATA TRANSFORMER MyModule.SimplifyCustomers
SOURCE JSON '{"head":{"vars":["customer","customerId","customerName"]},"results":{"bindings":[{"customer":{"type":"uri","value":"http://.../Customer/0"},"customerId":{"type":"literal","value":"CUST001"},"customerName":{"type":"literal","value":"Global Tech Solutions Inc."}}]}}'
{
  JSLT $$
{
  "customers": [for (.results.bindings)
    {
      "customerUri":  .customer.value,
      "customerId":   .customerId.value,
      "customerName": .customerName.value
    }
  ]
}
  $$;
};
```

**JSLT notes for this runtime:**

- `[for (.path.to.array) <expr>]` works for iteration.
- `.field.subfield` path access works.
- `[N]` array indexing works.
- `$var[start : end]` slice works for strings — **do not use `substring(...)`** (it silently drops the field from the output).
- `let` variables and `if/else` expressions work.
- `def fn(arg) ...` helper functions work.

### 4. JSON structure + Import Mapping

The JSON structure represents the **transformed** shape (after JSLT), not the raw SPARQL response:

```sql
CREATE JSON STRUCTURE MyModule.JSON_Customers
SNIPPET '{"customers":[{"customerUri":"http://example.com/Customer/0","customerId":"CUST001","customerName":"Global Tech Solutions Inc."}]}';

CREATE IMPORT MAPPING MyModule.IMM_Customers
  WITH JSON STRUCTURE MyModule.JSON_Customers
{
  CREATE MyModule.CustomerImport {
    CREATE MyModule.CustomerImport_Customer/MyModule.Customer = customers {
      CustomerUri  = customerUri,
      CustomerId   = customerId,
      CustomerName = customerName
    }
  }
};
```

### 5. Microflow — the actual API call

```sql
CREATE MICROFLOW MyModule.ACT_RefreshCustomers ()
RETURNS Boolean AS $Success
BEGIN
  LOG INFO NODE 'MyModule' '=== Refresh start ===';

  -- Clear existing persistent records (full replace)
  RETRIEVE $Existing FROM MyModule.Customer;
  LOOP $C IN $Existing BEGIN
    DELETE $C;
  END LOOP;

  -- Inline REST CALL — NOT the REST Client (see notes)
  $RawJson = REST CALL POST 'https://graphstudio.mendixdemo.com/sparql/graphmart/http%3A%2F%2Fcambridgesemantics.com%2FGraphmart%2F3617250aca6a40d88972c1c0de38f86a'
    HEADER 'Accept'       = 'application/sparql-results+json'
    HEADER 'Content-Type' = 'application/sparql-query'
    AUTH BASIC '<username>' PASSWORD '<password>'
    BODY 'PREFIX model: <http://cambridgesemantics.com/SourceLayer/c4ce0eca2e7241f2aee13b46fbdca3f8/Model#> SELECT ?customer ?customerId ?customerName FROM <http://cambridgesemantics.com/SourceLayer/c4ce0eca2e7241f2aee13b46fbdca3f8/Model> WHERE {1} ?customer a model:ExamplePlmBom.Customer; model:ExamplePlmBom.Customer.id ?customerId; model:ExamplePlmBom.Customer.name ?customerName; {2}'
    WITH ({1} = '{', {2} = '}')
    TIMEOUT 60
    RETURNS String
    ON ERROR CONTINUE;

  LOG INFO NODE 'MyModule' '{1}' WITH ({1} = 'HTTP status: ' + toString($latestHttpResponse/StatusCode));

  IF $latestHttpResponse/StatusCode = 200 THEN
    $SimplifiedJson = TRANSFORM $RawJson WITH MyModule.SimplifyCustomers;
    $ImportResult   = IMPORT FROM MAPPING MyModule.IMM_Customers($SimplifiedJson);
    LOG INFO NODE 'MyModule' '=== Done ===';
  END IF;

  RETURN true;
END;
/
```

### 6. Page

```sql
CREATE PAGE MyModule.Customer_Overview (
  Title:  'Customers (from Graph Mart)',
  Layout: Atlas_Core.Atlas_Default
) {
  DYNAMICTEXT heading (Content: 'Customers', RenderMode: H2)
  ACTIONBUTTON btnRefresh (Caption: 'Refresh', Action: MICROFLOW MyModule.ACT_RefreshCustomers, ButtonStyle: Primary)
  DATAGRID gridCustomers (DataSource: DATABASE MyModule.Customer SORT BY CustomerId ASC) {
    COLUMN colId   (Attribute: CustomerId,   Caption: 'ID')
    COLUMN colName (Attribute: CustomerName, Caption: 'Name')
    COLUMN colUri  (Attribute: CustomerUri,  Caption: 'URI')
  }
}
/
```

## Gotchas (things that burned an hour during development)

### `!` in Basic Auth password → 401

REST Client `AUTHENTICATION: BASIC (...)` with a literal password containing `!` sends no auth header at runtime. Workaround: use inline `REST CALL ... AUTH BASIC '<user>' PASSWORD '<pass>'`. The inline form works with the same literal credentials.

### SPARQL `{` braces in `BODY` templates are consumed as placeholder escapes

In `REST CALL ... BODY '...'`, the body is a template string where `{1}`, `{2}` are placeholders. A literal `{` must be escaped as `{{`, but in this runtime `{{` is sent **literally** rather than being converted to `{` → server returns `400 Bad Request`.

**Solution:** pass literal braces as placeholder values:

```sql
BODY '... WHERE {1} ... {2}'
WITH ({1} = '{', {2} = '}')
```

### JSON structure auto-detects ISO strings as DateTime

If your JSLT emits ISO 8601 timestamps (`"2026-04-13T14:00"`) and the target Mendix attribute is `String`, `CREATE JSON STRUCTURE ... SNIPPET '...'` will infer `DateTime` from the sample and mxbuild fails with `CE5015` ("schema type DateTime doesn't match attribute type String").

**Solutions:**
- Use a non-ISO sample value in the snippet (e.g. `"2026-04-13 14:00 CET"`).
- Or slice/format the timestamp in JSLT so it doesn't look like ISO 8601 (`$rawTime[11 : 16]` for `HH:MM`).
- Or change the target attribute to `DateTime`.

### Non-persistent child lists can't be extracted in microflows

The import mapping happily populates `CustomerImport` with a `ReferenceSet` of `Customer` children, but:
- `RETURN $Root/MyModule.CustomerImport_Customer` → "Error(s) in expression" at build
- `DECLARE $C List of MyModule.Customer = $Root/...` → "Error(s) in expression"
- `LOOP $c IN $Root/MyModule.CustomerImport_Customer` → "The 'Iterate over' property is required"
- DataGrid `DataSource: $currentObject/MyModule.CustomerImport_Customer` → BSON serializer drops the datasource

**Solution:** Make the target entity **persistent**. The import mapping commits them automatically, and the page uses the standard `DataSource: DATABASE MyModule.Customer` for the grid. A full replace on each refresh (delete-all-then-import) keeps data consistent with the graph.

### Rapid drop/create cycles on the same entity can corrupt the MPR

If you `DROP ENTITY X` then `CREATE ENTITY X` repeatedly while associations referencing `X` exist, the associations may hold the **old** entity GUID → mxbuild fails with `KeyNotFoundException`. Fix by dropping/recreating the broken association after the entity change.

## Exploring the graph

Before building the pipeline, explore the graph to understand what's there. Useful SPARQL queries (send via curl):

**List all classes with counts:**
```sparql
SELECT DISTINCT ?class (COUNT(?s) AS ?count)
FROM <http://.../Model>
WHERE { ?s a ?class }
GROUP BY ?class
ORDER BY DESC(?count)
```

**List properties used by a given class:**
```sparql
PREFIX model: <http://.../Model#>
SELECT DISTINCT ?property
FROM <http://.../Model>
WHERE {
  ?s a model:ExamplePlmBom.Customer ;
     ?property ?o .
}
ORDER BY ?property
```

**Filter to a single namespace (skip rdf/owl noise):**
```sparql
SELECT DISTINCT ?class ?property
FROM <http://.../Model>
WHERE {
  ?s a ?class ;
     ?property ?o .
  FILTER(STRSTARTS(STR(?class), "http://.../Model#MyPrefix"))
}
```

## Credential management

For demos, literal credentials inline in the microflow are the simplest and most reliable. For anything else, put them in a project constant and reference it from the microflow via `$ConstantName` (requires a non-trivial amount of setup — see the project settings skill).

**Do not** use `$ConstantName` in `CREATE REST CLIENT ... AUTHENTICATION: BASIC (Username: $C, Password: $C)` — the MDL parser rejects the `$` prefix there, and the skill files' claim of `Rest$ConstantValue` serialization isn't reachable.

## Related skills

- [rest-client.md](./rest-client.md) — REST Client + SEND REST REQUEST pattern (preferred when Basic Auth is not needed or uses simple passwords)
- [json-structures-and-mappings.md](./json-structures-and-mappings.md) — JSON structure / import mapping details
- [rest-call-from-json.md](./rest-call-from-json.md) — inline REST CALL + mapping pipeline
- [write-microflows.md](./write-microflows.md) — microflow syntax reference
