# Bulk Widget Property Updates

> **EXPERIMENTAL**: These commands are an untested proof-of-concept.
> Always use `DRY RUN` first and backup your project before applying changes.

Use `SHOW WIDGETS` and `UPDATE WIDGETS` to discover and modify widget properties across pages and snippets in bulk.

## Prerequisites

Widget commands require a full catalog build:
```sql
REFRESH CATALOG FULL;
```

## SHOW WIDGETS - Discover Widgets

### Basic Usage

```sql
-- Show all widgets
SHOW WIDGETS;

-- Filter by module
SHOW WIDGETS IN MyModule;

-- Filter by widget type (case-insensitive LIKE)
SHOW WIDGETS WHERE WidgetType LIKE '%combobox%';

-- Filter by name
SHOW WIDGETS WHERE Name = 'myGrid';

-- Combine filters
SHOW WIDGETS WHERE WidgetType LIKE '%DataGrid%' AND Name LIKE '%Overview%' IN MyModule;
```

### Output Columns

| Column | Description |
|--------|-------------|
| NAME | Widget name (may be auto-generated) |
| WIDGET TYPE | Full widget type ID (e.g., `com.mendix.widget.web.combobox.Combobox`) |
| CONTAINER | Page or snippet qualified name |
| MODULE | Module name |

### Common Widget Type Patterns

| Pattern | Matches |
|---------|---------|
| `%combobox%` | ComboBox widgets |
| `%datagrid%` | DataGrid2 and related widgets |
| `%textbox%` | TextBox widgets |
| `%dropdown%` | DropDown widgets |
| `%gallery%` | Gallery widgets |

## UPDATE WIDGETS - Modify Properties

### Syntax

```sql
UPDATE WIDGETS
  SET 'propertyName' = value [, 'propertyName' = value ...]
  WHERE condition [AND condition ...]
  [IN Module]
  [DRY RUN];
```

### Dry Run (Preview Changes)

Always preview changes first:

```sql
-- See what would change without modifying
UPDATE WIDGETS
  SET 'showLabel' = false
  WHERE WidgetType LIKE '%combobox%'
  DRY RUN;
```

Output shows:
```
Found 5 widget(s) in 3 container(s) matching the criteria

[DRY RUN] The following changes would be made:
  Would set 'showLabel' = false on combobox1 (ComboBox) in MyModule.OrderForm
  Would set 'showLabel' = false on combobox2 (ComboBox) in MyModule.CustomerPage
  ...

[DRY RUN] Would update 5 widget(s)

Run without DRY RUN to apply changes.
```

### Apply Changes

Remove `DRY RUN` to apply:

```sql
UPDATE WIDGETS
  SET 'showLabel' = false
  WHERE WidgetType LIKE '%combobox%'
  IN MyModule;
```

### Property Value Types

| Type | Examples |
|------|----------|
| String | `'contains'`, `'above'` |
| Number | `4`, `100`, `3.14` |
| Boolean | `true`, `false` |
| Null | `null` |

### Examples

```sql
-- Hide labels on all comboboxes
UPDATE WIDGETS
  SET 'showLabel' = false
  WHERE WidgetType LIKE '%combobox%';

-- Set multiple properties
UPDATE WIDGETS
  SET 'showLabel' = false, 'labelWidth' = 4
  WHERE WidgetType LIKE '%textbox%'
  IN MyModule;

-- Change filter mode on DataGrid filters
UPDATE WIDGETS
  SET 'filterMode' = 'contains'
  WHERE WidgetType LIKE '%DatagridTextFilter%';
```

## Important Notes

### Known Limitations (Experimental)

**UPDATE WIDGETS functionality is not fully implemented.**
- The DRY RUN mode shows which widgets would be matched
- Actual property updates require additional implementation work
- Use SHOW WIDGETS for discovery, then manually update properties in Studio Pro

### After Making Changes

1. Refresh the catalog to see updated data:
   ```sql
   REFRESH CATALOG FULL FORCE;
   ```

2. Open the project in Studio Pro to verify changes

### Supported Properties

Only primitive properties (string, number, boolean) are supported:
- `showLabel`, `labelWidth`, `placeholder`
- `filterMode`, `defaultValue`
- Widget-specific configuration properties

NOT supported:
- DataSource properties
- Action properties (onClick, etc.)
- Nested object properties
- Expression properties

### Finding Property Names

To find the correct property names:
1. Create a widget in Studio Pro
2. Use `DESCRIBE PAGE Module.PageName` to see widget structure
3. Or check the Mendix widget documentation

## Workflow Example

```sql
-- 1. Build catalog
REFRESH CATALOG FULL;

-- 2. Discover widgets
SHOW WIDGETS WHERE WidgetType LIKE '%combobox%';

-- 3. Preview changes
UPDATE WIDGETS SET 'showLabel' = false WHERE WidgetType LIKE '%combobox%' DRY RUN;

-- 4. Apply changes
UPDATE WIDGETS SET 'showLabel' = false WHERE WidgetType LIKE '%combobox%';

-- 5. Rebuild catalog
REFRESH CATALOG FULL FORCE;

-- 6. Verify
SHOW WIDGETS WHERE WidgetType LIKE '%combobox%';
```
