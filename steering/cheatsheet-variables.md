# MDL Variable Cheatsheet

Quick reference for variable declarations in MDL microflows.

## Declaration Syntax

| Type | Syntax | Example |
|------|--------|---------|
| String | `DECLARE $name String = 'value';` | `DECLARE $Message String = '';` |
| Integer | `DECLARE $name Integer = 0;` | `DECLARE $Count Integer = 0;` |
| Boolean | `DECLARE $name Boolean = true;` | `DECLARE $IsValid Boolean = true;` |
| Decimal | `DECLARE $name Decimal = 0.0;` | `DECLARE $Amount Decimal = 0;` |
| DateTime | `DECLARE $name DateTime = [%CurrentDateTime%];` | `DECLARE $Now DateTime = [%CurrentDateTime%];` |
| Entity | `DECLARE $name AS Module.Entity;` | `DECLARE $Customer AS Sales.Customer;` |
| List | `DECLARE $name List of Module.Entity = empty;` | `DECLARE $Orders List of Sales.Order = empty;` |

## Key Rules

1. **Primitives**: Use `DECLARE $var Type = value;` (initialization required)
2. **Entities**: Use `DECLARE $var AS Module.Entity;` (use AS keyword, no initialization)
3. **Lists**: Use `DECLARE $var List of Module.Entity = empty;`
4. **SET requires DECLARE**: Always declare variables before using SET
5. **Parameters are pre-declared**: Microflow parameters don't need DECLARE

## Common Mistakes

### Entity Declaration

```mdl
-- WRONG: Missing AS keyword
DECLARE $Product Module.Product = empty;

-- CORRECT: Use AS for entity types
DECLARE $Product AS Module.Product;
```

### SET Without DECLARE

```mdl
-- WRONG: Variable not declared
IF $Value > 10 THEN
  SET $Message = 'High';  -- ERROR!
END IF;

-- CORRECT: Declare first
DECLARE $Message String = '';
IF $Value > 10 THEN
  SET $Message = 'High';
END IF;
```

### List Declaration

```mdl
-- WRONG: Missing 'of' keyword
DECLARE $Items List Module.Item = empty;

-- CORRECT: Use 'List of'
DECLARE $Items List of Module.Item = empty;
```

## Special Values

| Value | Usage |
|-------|-------|
| `empty` | Null/empty value for any type |
| `[%CurrentDateTime%]` | Current date and time |
| `[%CurrentUser%]` | Currently logged in user object |
| `true` / `false` | Boolean literals |

## Parameter vs Variable

```mdl
CREATE MICROFLOW Module.Example (
  $Input: String,              -- Parameter: auto-declared
  $Entity: Module.Customer     -- Parameter: auto-declared
)
RETURNS Boolean
BEGIN
  -- Parameters $Input and $Entity are already available

  DECLARE $Result Boolean = true;  -- Local variable: must declare
  DECLARE $Temp AS Module.Order;   -- Local entity: must declare

  RETURN $Result;
END;
/
```

## Variable Scope

- Parameters: Available throughout the microflow
- DECLARE variables: Available from declaration point forward
- Loop variables: Only available inside the loop body

```mdl
LOOP $Item IN $ItemList
BEGIN
  -- $Item is available here (derived from list type)
  SET $Count = $Count + 1;
END LOOP;
-- $Item is NOT available here
```
