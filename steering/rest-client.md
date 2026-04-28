# REST Integration Skill

Use this skill when integrating with external REST APIs from Mendix.

## Two Approaches

Mendix offers two ways to call REST APIs from microflows. Choose based on the use case:

| Approach | When to Use | Artifacts |
|----------|-------------|-----------|
| **REST Client + SEND REST REQUEST** | Structured APIs with multiple operations, reusable across microflows, Studio Pro UI support | REST client document + microflow |
| **REST CALL (inline)** | One-off calls, quick prototyping, dynamic URLs, low-level HTTP control | Microflow only |

Both can be combined with **Data Transformers** (Mendix 11.9+) and **Import/Export Mappings** to map between JSON and entities.

---

## Approach 1: REST Client (Recommended)

Define the API once as a REST client document, then call its operations from microflows.

### Step 1 — Create the REST Client

```sql
CREATE REST CLIENT Module.OpenMeteoAPI (
  BaseUrl: 'https://api.open-meteo.com/v1',
  Authentication: NONE
)
{
  OPERATION GetForecast {
    Method: GET,
    Path: '/forecast',
    Query: ($latitude: Decimal, $longitude: Decimal, $current: String),
    Headers: ('Accept' = 'application/json'),
    Timeout: 30,
    Response: JSON AS $WeatherJson
  }

  OPERATION PostData {
    Method: POST,
    Path: '/submit',
    Headers: ('Content-Type' = 'application/json'),
    Body: JSON FROM $JsonPayload,
    Response: NONE
  }
};
```

### Authentication

```sql
-- No authentication
Authentication: NONE

-- Basic auth
Authentication: BASIC (Username: 'user', Password: 'secret')
```

### Body Types

```sql
-- JSON variable (Mendix expression stored on the operation)
Body: JSON FROM $JsonPayload

-- String template with parameter placeholders
Body: TEMPLATE '{ "name": "{name}", "value": {value} }'

-- Export mapping (entity → JSON, aligned with export mapping syntax)
Body: MAPPING Module.RequestEntity {
  name = Name,
  email = Email,
}
```

### Response Types

```sql
-- Simple types (variable binding is at the call site, not stored on the operation)
Response: JSON AS $Result
Response: STRING AS $Text
Response: FILE AS $Document
Response: STATUS AS $Code
Response: NONE

-- Import mapping (JSON → entity, aligned with import mapping syntax)
Response: MAPPING Module.ResponseEntity {
  "Id" = "id",
  "Status" = "status",
  CREATE Module.Items_Response/Module.Item = items {
    "Sku" = "sku",
    "Quantity" = "quantity",
  }
}
```

### Step 2 — Call from a Microflow

```sql
CREATE MICROFLOW Module.ACT_GetWeather ()
RETURNS Module.WeatherInfo AS $Weather
BEGIN
  -- Call the REST client operation
  SEND REST REQUEST Module.OpenMeteoAPI.GetForecast;

  -- Extract response body from system variable
  DECLARE $RawJson String = $latestHttpResponse/Content;

  -- (Optional) Transform with JSLT
  $SimplifiedJson = TRANSFORM $RawJson WITH Module.SimplifyWeather;

  -- Import into entity
  $Weather = IMPORT FROM MAPPING Module.IMM_Weather($SimplifiedJson);

  RETURN $Weather;
END;
/
```

**CRITICAL**: After `SEND REST REQUEST`, the response is in `$latestHttpResponse` (System.HttpResponse):
- `$latestHttpResponse/Content` — response body (String)
- `$latestHttpResponse/StatusCode` — HTTP status (Integer)

### Show / Describe / Drop

```sql
SHOW REST CLIENTS [IN Module];
DESCRIBE REST CLIENT Module.ClientName;
DROP REST CLIENT Module.ClientName;
CREATE OR MODIFY REST CLIENT Module.ClientName ...  -- idempotent
```

---

## Approach 2: REST CALL (Inline HTTP)

Call an HTTP endpoint directly from a microflow — no REST client document needed. Best for one-off calls, dynamic URLs, or low-level control.

```sql
-- Simple GET returning a string
$Response = REST CALL GET 'https://api.example.com/data'
  HEADER Accept = 'application/json'
  TIMEOUT 30
  RETURNS String;

-- GET with URL template parameters
$Response = REST CALL GET 'https://api.example.com/users/{1}' WITH (
  {1} = toString($UserId)
)
  HEADER Accept = 'application/json'
  RETURNS String;

-- POST with body
$Response = REST CALL POST 'https://api.example.com/items'
  HEADER 'Content-Type' = 'application/json'
  BODY '{"name": "test"}'
  RETURNS String;

-- With basic auth
$Response = REST CALL GET 'https://api.example.com/secure'
  AUTH BASIC 'username' PASSWORD 'password'
  RETURNS String;

-- With import mapping (JSON → entity)
$Item = REST CALL GET 'https://api.example.com/item/1'
  HEADER Accept = 'application/json'
  RETURNS MAPPING Module.IMM_Item AS Module.Item;

-- Fire and forget
REST CALL DELETE 'https://api.example.com/item/1'
  RETURNS Nothing;

-- Error handling
$Response = REST CALL GET 'https://api.example.com/data'
  RETURNS String
  ON ERROR CONTINUE;
```

---

## Data Transformers (JSLT — Mendix 11.9+)

Transform complex JSON responses into simpler structures before import mapping.

```sql
-- Define the transformer
CREATE DATA TRANSFORMER Module.SimplifyWeather
SOURCE JSON '{"latitude": 52.52, "current": {"temperature_2m": 12.8, "wind_speed_10m": 18.3}}'
{
  JSLT $$
{
  "temp": .current.temperature_2m,
  "wind": .current.wind_speed_10m,
  "lat": .latitude
}
  $$;
};

-- Use in a microflow
$SimplifiedJson = TRANSFORM $RawJson WITH Module.SimplifyWeather;
```

Single-line JSLT: `JSLT '{ "temp": .current.temperature_2m }';`
Multi-line JSLT: `JSLT $$ { ... } $$;` (dollar-quoting, same as Java actions)

```sql
LIST DATA TRANSFORMERS [IN Module];
DESCRIBE DATA TRANSFORMER Module.Name;
DROP DATA TRANSFORMER Module.Name;
```

---

## JSON Structures & Mappings

See [json-structures-and-mappings.md](json-structures-and-mappings.md) for full reference. Quick summary:

```sql
-- JSON structure from snippet
CREATE JSON STRUCTURE Module.JSON_Weather
SNIPPET '{"temp": 12.8, "wind": 18.3, "lat": 52.52}';

-- Non-persistent entity
CREATE NON-PERSISTENT ENTITY Module.WeatherInfo (
  Temperature: Decimal,
  WindSpeed: Decimal,
  Latitude: Decimal
);
/

-- Import mapping (JSON → entity)
CREATE IMPORT MAPPING Module.IMM_Weather
  WITH JSON STRUCTURE Module.JSON_Weather
{
  CREATE Module.WeatherInfo {
    Temperature = temp,
    WindSpeed = wind,
    Latitude = lat
  }
};

-- Use in microflow
$Weather = IMPORT FROM MAPPING Module.IMM_Weather($JsonString);
$JsonOutput = EXPORT TO MAPPING Module.EMM_Weather($WeatherEntity);
```

---

## Complete Pipeline Example

Full example: call weather API → transform → import → show on page.

```sql
-- 1. Entity
CREATE NON-PERSISTENT ENTITY Module.CurrentWeather (
  Temperature: Decimal,
  WindSpeed: Decimal,
  Latitude: Decimal,
  ObservationTime: DateTime
);
/

-- 2. Data Transformer (simplify API response)
CREATE DATA TRANSFORMER Module.SimplifyWeather
SOURCE JSON '{"latitude":52.52,"current":{"time":"2024-01-15T14:00","temperature_2m":12.8,"wind_speed_10m":18.3}}'
{
  JSLT $$
{
  "temperature": .current.temperature_2m,
  "windSpeed": .current.wind_speed_10m,
  "latitude": .latitude,
  "observationTime": .current.time
}
  $$;
};

-- 3. JSON Structure + Import Mapping (for transformed output)
CREATE JSON STRUCTURE Module.JSON_Weather
SNIPPET '{"temperature":12.8,"windSpeed":18.3,"latitude":52.52,"observationTime":"2024-01-15T14:00"}';

CREATE IMPORT MAPPING Module.IMM_Weather
  WITH JSON STRUCTURE Module.JSON_Weather
{
  CREATE Module.CurrentWeather {
    Temperature = temperature,
    WindSpeed = windSpeed,
    Latitude = latitude,
    ObservationTime = observationTime
  }
};

-- 4. REST Client
CREATE REST CLIENT Module.WeatherAPI (
  BaseUrl: 'https://api.open-meteo.com/v1',
  Authentication: NONE
)
{
  OPERATION GetCurrent {
    Method: GET,
    Path: '/forecast',
    Query: ($latitude: Decimal, $longitude: Decimal, $current: String),
    Headers: ('Accept' = 'application/json'),
    Response: JSON AS $Result
  }
};

-- 5. Microflow (REST Client → Transform → Import)
CREATE MICROFLOW Module.ACT_GetWeather ()
RETURNS Module.CurrentWeather AS $Weather
BEGIN
  SEND REST REQUEST Module.WeatherAPI.GetCurrent;
  DECLARE $RawJson String = $latestHttpResponse/Content;
  $SimplifiedJson = TRANSFORM $RawJson WITH Module.SimplifyWeather;
  $Weather = IMPORT FROM MAPPING Module.IMM_Weather($SimplifiedJson);
  RETURN $Weather;
END;
/
```

---

## Mendix Validation Rules

| Rule | Error | Fix |
|------|-------|-----|
| Every operation MUST have an Accept header | CE7062 | Auto-added by serializer if missing |
| POST/PUT/PATCH MUST have a body | CE7064 | Auto-added by serializer (empty JSON body) |
| Template placeholders must match parameters | CE7056 | `{name}` requires a parameter named `name` |
| No custom error handling on SEND REST REQUEST | CE6035 | Always uses abort-on-error semantics |
| Data Transformer requires 11.9+ | version check | `checkFeature("integration", "data_transformer", ...)` |

## BSON Types Reference

| MDL Concept | BSON Type |
|-------------|-----------|
| REST client document | `Rest$ConsumedRestService` |
| Operation | `Rest$RestOperation` |
| GET/DELETE method | `Rest$RestOperationMethodWithoutBody` |
| POST/PUT/PATCH method | `Rest$RestOperationMethodWithBody` |
| JSON body | `Rest$JsonBody` |
| Template body | `Rest$StringBody` |
| Export mapping body | `Rest$ImplicitMappingBody` |
| No response | `Rest$NoResponseHandling` |
| Import mapping response | `Rest$ImplicitMappingResponseHandling` |
| Header | `Rest$HeaderWithValueTemplate` |
| Path parameter | `Rest$OperationParameter` |
| Query parameter | `Rest$QueryParameter` |
| Data transformer | `DataTransformers$DataTransformer` |
| JSLT step | `DataTransformers$JsltAction` |
| Transform action | `Microflows$TransformJsonAction` |
| Send request action | `Microflows$RestOperationCallAction` |
| Inline HTTP action | `Microflows$RestCallAction` |
