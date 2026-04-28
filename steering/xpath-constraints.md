# XPath Constraints in MDL

This skill provides reference for writing XPath constraint expressions in MDL RETRIEVE statements, page data sources, and security rules.

## When to Use This Skill

- Writing `RETRIEVE ... WHERE [xpath]` statements in microflows
- Writing `DATABASE FROM Entity WHERE [xpath]` in page data sources
- Writing `GRANT ... WHERE 'xpath'` for row-level entity access
- Debugging XPath parsing or serialization issues

## XPath vs Mendix Expressions

**Critical distinction**: XPath constraints (inside `[...]`) use different syntax from Mendix expressions (in SET, IF, DECLARE, etc.):

| Feature | XPath `[...]` | Mendix Expression |
|---------|---------------|-------------------|
| Path separator | `/` (always path traversal) | `/` (also division) |
| Boolean ops | lowercase: `and`, `or`, `not()` | `AND`, `OR`, `NOT` |
| Negation | `not(expr)` function | `NOT expr` |
| Empty check | `= empty`, `!= empty` | `= empty` |
| Token quoting | `'[%CurrentUser%]'` (quoted) | `[%CurrentUser%]` (unquoted) |
| Nested filter | `Assoc/Entity[pred]` | Not applicable |

## Syntax Reference

### Simple Comparisons

```mdl
RETRIEVE $Orders FROM Module.Order
  WHERE [State = 'Completed'];

RETRIEVE $Active FROM Module.Customer
  WHERE [IsActive = true];

RETRIEVE $Recent FROM Module.Order
  WHERE [OrderDate != empty];

RETRIEVE $HighValue FROM Module.Order
  WHERE [TotalAmount >= $MinAmount];
```

Operators: `=`, `!=`, `<`, `>`, `<=`, `>=`

### Boolean Logic

```mdl
-- AND
WHERE [State = 'Completed' and IsPaid = true]

-- OR
WHERE [State = 'Pending' or State = 'Processing']

-- Grouped
WHERE [State = 'Completed' and ($IgnorePaid or IsPaid = true)]

-- NOT
WHERE [not(IsPaid)]
WHERE [not(contains(Name, 'demo'))]
```

### Association Path Traversal

Bare association paths (without `$variable` prefix) navigate through the domain model:

```mdl
-- Single-hop: filter by associated object
WHERE [Module.Order_Customer = $Customer]

-- Multi-hop: traverse through associations
WHERE [Module.Order_Customer/Module.Customer/Name = $CustomerName]

-- Existence check: has an associated object
WHERE [Module.Order_Customer/Module.Customer]

-- Negated existence: has NO associated object
WHERE [not(Module.Order_Customer/Module.Customer)]
```

**Rule**: Always use the fully qualified association name (`Module.AssociationName`).

### Variable Paths

```mdl
-- Compare attribute via variable path
WHERE [Module.Assoc/Module.Entity/Name = $Variable/Name]

-- Variable on right side
WHERE [Name = $currentObject/SearchString]
```

### Nested Predicates

Filter intermediate path steps with inline `[predicate]`:

```mdl
-- Only lines of completed orders
WHERE [Module.OrderLine_Order/Module.Order[State = 'Completed']]

-- Nested predicate with further traversal
WHERE [Module.OrderLine_Order/Module.Order[State = 'Active']/Module.Order_Category/Module.Category/Name = $CategoryName]

-- reversed() path modifier (traverse association in reverse direction)
WHERE [System.grantableRoles[reversed()]/System.UserRole/System.UserRoles = '[%CurrentUser%]']
```

### Functions

```mdl
-- String search
WHERE [contains(Name, $SearchStr)]
WHERE [starts-with(Name, $Prefix)]
WHERE [not(contains(Name, 'demo'))]

-- Boolean functions
WHERE [IsActive = true()]
WHERE [Displayed = false()]
```

Supported functions: `contains()`, `starts-with()`, `not()`, `true()`, `false()`

### Tokens

Mendix tokens provide runtime values. In XPath, tokens used as values must be quoted:

```mdl
-- Unquoted token (parsed by MDL, auto-quoted in BSON)
WHERE [OrderDate < [%CurrentDateTime%]]
WHERE [System.owner = [%CurrentUser%]]

-- Quoted token in string literal (passed through as-is)
WHERE [System.owner = '[%CurrentUser%]']
```

Common tokens: `[%CurrentUser%]`, `[%CurrentDateTime%]`, `[%CurrentObject%]`, `[%UserRole_RoleName%]`, `[%DayLength%]`

### ID Pseudo-Attribute

The `id` pseudo-attribute compares object identity (GUID):

```mdl
WHERE [id = $currentUser]
WHERE [id != $existingObject]
WHERE [id = '[%CurrentUser%]']
```

## Usage Contexts

### RETRIEVE in Microflows

```mdl
RETRIEVE $Results FROM Module.Entity
  WHERE [IsActive = true and State = 'Ready']
  SORT BY Name ASC
  LIMIT 100;
```

The expression inside `[...]` is parsed as XPath and stored in BSON as the `XpathConstraint` field.

### Page Data Sources

```mdl
DATAGRID dg (
  DataSource: DATABASE FROM Module.Entity WHERE [State != 'Cancelled'] SORT BY Name ASC
) {
  COLUMN col1 (Attribute: Name, Caption: 'Name')
}
```

Multiple bracket constraints can be chained. Consecutive brackets without an operator are treated as AND (standard Mendix XPath):

```mdl
-- Consecutive brackets (implicit AND) — standard Mendix XPath syntax
DataSource: DATABASE FROM Module.Entity WHERE [IsActive = true][Stock > 0]

-- Explicit AND: same result
DataSource: DATABASE FROM Module.Entity WHERE [IsActive = true] AND [Stock > 0]

-- Mix with OR: combines into single bracket
DataSource: DATABASE FROM Module.Entity WHERE [IsActive = true] OR [Stock > 10]
```

### GRANT Entity Access (Security)

For security rules, XPath is passed as a **string literal** (not parsed):

```mdl
GRANT Module.Role ON Module.Entity (
  READ *,
  WRITE *
) WHERE '[System.owner = ''[%CurrentUser%]'']';
```

Note the double single-quotes for escaping inside the string literal.

## Common Patterns

### Parameterized Search

```mdl
CREATE MICROFLOW Module.Search ($Query: String, $ActiveOnly: Boolean)
RETURNS Boolean
BEGIN
  RETRIEVE $Results FROM Module.Customer
    WHERE [($ActiveOnly = false or IsActive = true)
      and (contains(Name, $Query) or contains(Email, $Query))];
  RETURN true;
END;
```

### Date Range Filter

```mdl
RETRIEVE $Orders FROM Module.Order
  WHERE [OrderDate >= $StartDate and OrderDate <= $EndDate];
```

### Optional Filters (empty = skip)

```mdl
RETRIEVE $Orders FROM Module.Order
  WHERE [($Category = empty or Module.Order_Category = $Category)
    and ($State = empty or State = $State)];
```

### Owner-Based Security

```mdl
-- In microflow
RETRIEVE $MyItems FROM Module.Item
  WHERE [System.owner = '[%CurrentUser%]'];

-- In security rule
GRANT Module.User ON Module.Item (READ ALL) WHERE '[System.owner = ''[%CurrentUser%]'']';
```

## Validation

Always validate XPath syntax before execution:

```bash
# Syntax check (no project needed)
./bin/mxcli check script.mdl

# With reference validation (needs project)
./bin/mxcli check script.mdl -p app.mpr --references
```

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `mismatched input` on keyword | Attribute name is a reserved word | This is handled — `xpathWord` accepts any keyword as identifier |
| Token not quoted in BSON | Token in Mendix expression context | Use `[...]` bracket syntax for XPath, not bare expression |
| `CE0111` path error | Missing module prefix on association | Use `Module.AssociationName`, not just `AssociationName` |
| `not` parsed as keyword | Using `NOT` (uppercase) in XPath | XPath uses lowercase `not()` as a function |
