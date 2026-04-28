# Oracle Forms to Mendix Migration Skill

This skill provides comprehensive guidance for migrating Oracle Forms applications to Mendix using MDL (Mendix Definition Language).

## When to Use This Skill

Use this skill when:
- Converting Oracle Forms (.fmb) applications to Mendix
- Translating PL/SQL logic to Mendix microflows
- Mapping Oracle Forms UI elements to Mendix widgets
- Planning a migration strategy for legacy Oracle Forms systems

## Migration Overview

Oracle Forms migration to Mendix involves:
1. **Data Model**: Oracle tables → Mendix entities
2. **Business Logic**: PL/SQL triggers/procedures → Mendix microflows
3. **User Interface**: Forms blocks/items → Mendix pages/widgets
4. **Navigation**: Form canvases → Mendix page navigation

## Reserved Word Conflicts

Most common words (`Check`, `Text`, `Format`, `Value`, `Type`, `Index`, `Status`, `Select`, etc.) now work **unquoted** as attribute names in MDL. Only structural keywords (`Create`, `Delete`, `Begin`, `End`, `Return`, `Entity`, `Module`) need quoting.

### Naming Best Practices

While most words are no longer reserved, using descriptive names is still recommended for clarity:

| Oracle Forms Field | Recommended Mendix Name | Notes |
|-------------------|-------------------------|-------|
| `Check` | `Check` or `CheckStatus` | Works unquoted |
| `Text` | `Text` or `TextContent` | Works unquoted |
| `Format` | `Format` or `FormatType` | Works unquoted |
| `Value` | `Value` or `FieldValue` | Works unquoted |
| `Name` | `Name` or `ItemName` | Works unquoted (not a keyword) |
| `Type` | `Type` or `ItemType` | Works unquoted |
| `Create` | `"Create"` or `CreatedBy` | **Requires quoting** (structural keyword) |
| `Delete` | `"Delete"` or `IsDeleted` | **Requires quoting** (structural keyword) |

### Example

```mdl
CREATE PERSISTENT ENTITY MyModule.FormField (
  Check: Boolean DEFAULT false,  -- Works unquoted
  Text: String(500),             -- Works unquoted
  Format: String(50),            -- Works unquoted
  CheckFlag: Boolean DEFAULT false  -- Renamed alternative (also fine)
  TextContent: String(500), -- Renamed
  FormatType: String(50)    -- Renamed
);
```

## Script Organization

### Execution Order Dependencies

MDL scripts execute statements sequentially. Items created in one statement can be referenced in subsequent statements within the **same script execution**.

**Key Insight**: Microflows and pages created earlier in the script are tracked and can be resolved by later statements.

### Recommended Script Structure

```mdl
-- ============================================
-- PHASE 1: Domain Model (Entities & Associations)
-- ============================================

CREATE PERSISTENT ENTITY MyModule.Customer (
  CustomerCode: String(50),
  CustomerName: String(200),
  Email: String(200),
  IsActive: Boolean DEFAULT true
);

CREATE PERSISTENT ENTITY MyModule.Order (
  OrderNumber: String(50),
  OrderDate: DateTime,
  TotalAmount: Decimal
);

CREATE ASSOCIATION MyModule.Order_Customer (
  MyModule.Order [*] -> MyModule.Customer [1]
);
/

-- ============================================
-- PHASE 2: Microflows (Business Logic)
-- ============================================

/**
 * Validates and saves a customer record
 * Replaces Oracle Forms POST-INSERT/POST-UPDATE triggers
 */
CREATE MICROFLOW MyModule.ACT_Customer_Save ($Customer: MyModule.Customer)
RETURNS Boolean AS $Success
BEGIN
  DECLARE $Success Boolean = false;

  -- Validation (replaces WHEN-VALIDATE-ITEM)
  IF $Customer/CustomerCode = empty THEN
    VALIDATION FEEDBACK $Customer/CustomerCode MESSAGE 'Customer code is required';
    RETURN false;
  END IF;

  COMMIT $Customer WITH EVENTS;
  SET $Success = true;
  RETURN $Success;
END;
/

-- ============================================
-- PHASE 3: Pages (User Interface)
-- ============================================

-- Now this page can reference the microflow created above
CREATE PAGE MyModule.Customer_Edit
LAYOUT Atlas_Default
TITLE 'Edit Customer'
PARAMETER $Customer: MyModule.Customer
WIDGETS (
  DATAVIEW SOURCE $Customer (
    INPUT 'CustomerCode' ATTRIBUTE CustomerCode LABEL 'Customer Code',
    INPUT 'CustomerName' ATTRIBUTE CustomerName LABEL 'Name',
    INPUT 'Email' ATTRIBUTE Email LABEL 'Email',

    CONTAINER 'ButtonBar' (
      -- Reference to microflow created in Phase 2
      BUTTON 'Save' CALL MICROFLOW MyModule.ACT_Customer_Save (
        Customer = $Customer
      ),
      BUTTON 'Cancel' ON CLICK CLOSE PAGE
    )
  )
);
/
```

## Validation Feedback

### VALIDATION FEEDBACK Syntax

**CRITICAL**: VALIDATION FEEDBACK requires an attribute path, not just a message.

**WRONG:**
```mdl
VALIDATION FEEDBACK 'Customer code is required';  -- Missing attribute!
```

**CORRECT:**
```mdl
-- Syntax: VALIDATION FEEDBACK $entity/attribute MESSAGE 'message'
VALIDATION FEEDBACK $Customer/CustomerCode MESSAGE 'Customer code is required';
VALIDATION FEEDBACK $Order/OrderDate MESSAGE 'Order date cannot be in the future';
```

### Mapping Oracle Forms Validation

| Oracle Forms | Mendix MDL |
|--------------|------------|
| `WHEN-VALIDATE-ITEM` trigger | `IF ... VALIDATION FEEDBACK` in microflow |
| `RAISE FORM_TRIGGER_FAILURE` | `VALIDATION FEEDBACK` + `RETURN false` |
| `MESSAGE('error text')` | `VALIDATION FEEDBACK $Entity/Attribute MESSAGE 'error text'` |

### Complete Validation Pattern

```mdl
/**
 * Validates order before save
 * Replaces Oracle Forms WHEN-VALIDATE-RECORD trigger
 */
CREATE MICROFLOW MyModule.ACT_Order_Validate ($Order: MyModule.Order)
RETURNS Boolean AS $IsValid
BEGIN
  DECLARE $IsValid Boolean = true;

  -- Required field validation
  IF $Order/OrderNumber = empty THEN
    VALIDATION FEEDBACK $Order/OrderNumber MESSAGE 'Order number is required';
    SET $IsValid = false;
  END IF;

  -- Date validation
  IF $Order/OrderDate > [%CurrentDateTime%] THEN
    VALIDATION FEEDBACK $Order/OrderDate MESSAGE 'Order date cannot be in the future';
    SET $IsValid = false;
  END IF;

  -- Cross-field validation
  IF $Order/TotalAmount < 0 THEN
    VALIDATION FEEDBACK $Order/TotalAmount MESSAGE 'Total amount cannot be negative';
    SET $IsValid = false;
  END IF;

  RETURN $IsValid;
END;
/
```

## PL/SQL to Microflow Mapping

### Data Manipulation

| Oracle PL/SQL | Mendix MDL |
|---------------|------------|
| `INSERT INTO table ...` | `$var = CREATE Module.Entity (...)` |
| `UPDATE table SET ...` | `CHANGE $var (...)` + `COMMIT $var` |
| `DELETE FROM table ...` | `DELETE $var` |
| `SELECT ... INTO ...` | `RETRIEVE $var FROM Module.Entity WHERE ...` |
| `COMMIT` | `COMMIT $var` |
| `ROLLBACK` | Built-in with error handlers |

### Control Flow

| Oracle PL/SQL | Mendix MDL |
|---------------|------------|
| `IF ... THEN ... ELSIF ... ELSE ... END IF` | `IF ... THEN ... ELSE ... END IF` |
| `FOR ... LOOP ... END LOOP` | `LOOP $item IN $list BEGIN ... END LOOP` |
| `WHILE ... LOOP ... END LOOP` | Not directly supported; use recursive microflow |
| `CURSOR` | `RETRIEVE $list FROM ...` then `LOOP` |
| `EXCEPTION WHEN ... THEN` | `ON ERROR { ... }` |

### Example: PL/SQL to MDL

**Oracle PL/SQL:**
```sql
DECLARE
  v_count NUMBER := 0;
  v_total NUMBER := 0;
BEGIN
  FOR rec IN (SELECT * FROM orders WHERE status = 'PENDING') LOOP
    v_count := v_count + 1;
    v_total := v_total + rec.amount;

    UPDATE orders SET status = 'PROCESSED' WHERE id = rec.id;
  END LOOP;

  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Processed ' || v_count || ' orders, total: ' || v_total);
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    RAISE;
END;
```

**Mendix MDL:**
```mdl
CREATE MICROFLOW MyModule.ACT_ProcessPendingOrders ()
RETURNS String AS $Result
BEGIN
  DECLARE $OrderList List of MyModule.Order = empty;
  DECLARE $Count Integer = 0;
  DECLARE $Total Decimal = 0;
  DECLARE $Result String = '';

  -- Retrieve pending orders (replaces CURSOR)
  RETRIEVE $OrderList FROM MyModule.Order
    WHERE Status = 'PENDING';

  -- Process each order (replaces FOR LOOP)
  LOOP $Order IN $OrderList
  BEGIN
    SET $Count = $Count + 1;
    SET $Total = $Total + $Order/Amount;

    CHANGE $Order (Status = 'PROCESSED');
    COMMIT $Order ON ERROR {
      LOG ERROR 'Failed to process order: ' + $Order/OrderNumber;
    };
  END LOOP;

  LOG INFO 'Processed ' + toString($Count) + ' orders, total: ' + toString($Total);
  SET $Result = 'Processed ' + toString($Count) + ' orders';
  RETURN $Result;
END;
/
```

## UI Component Mapping

### Oracle Forms Items to Mendix Widgets

| Oracle Forms Item | Mendix Widget | MDL Syntax |
|-------------------|---------------|------------|
| Text Item | Text Input | `INPUT 'name' ATTRIBUTE attr` |
| Display Item | Text | `TEXT 'content'` |
| Check Box | Check Box | `CHECKBOX 'name' ATTRIBUTE attr` |
| Radio Group | Radio Buttons | `RADIO 'name' ATTRIBUTE attr` |
| List Item (LOV) | Drop-down | `DROPDOWN 'name' ATTRIBUTE attr` |
| Push Button | Button | `BUTTON 'name' ON CLICK ...` |
| Tab Canvas | Tab Container | `TAB_CONTAINER (TAB 'name' (...))` |

### Oracle Forms Blocks to Mendix DataViews

**Oracle Forms Block → Mendix DataView:**
```mdl
-- Single-record block
DATAVIEW SOURCE $Customer (
  INPUT 'Code' ATTRIBUTE CustomerCode,
  INPUT 'Name' ATTRIBUTE CustomerName
)

-- Multi-record block (tabular)
DATAGRID SOURCE $OrderList (
  COLUMN 'OrderNumber' ATTRIBUTE OrderNumber,
  COLUMN 'OrderDate' ATTRIBUTE OrderDate,
  COLUMN 'Amount' ATTRIBUTE TotalAmount
)
```

### Master-Detail Pattern

**Oracle Forms Master-Detail → Mendix:**
```mdl
CREATE PAGE MyModule.CustomerOrders
LAYOUT Atlas_Default
TITLE 'Customer Orders'
PARAMETER $Customer: MyModule.Customer
WIDGETS (
  -- Master block
  DATAVIEW SOURCE $Customer (
    INPUT 'Code' ATTRIBUTE CustomerCode READONLY,
    INPUT 'Name' ATTRIBUTE CustomerName READONLY
  ),

  -- Detail block (orders for this customer)
  DATAGRID 'OrderGrid' SOURCE DATABASE MyModule.Order
    WHERE '[MyModule.Order_Customer = $Customer]' (
    COLUMN 'OrderNumber' ATTRIBUTE OrderNumber,
    COLUMN 'OrderDate' ATTRIBUTE OrderDate,
    COLUMN 'Amount' ATTRIBUTE TotalAmount
  )
);
/
```

## Triggers to Microflows

### Common Trigger Mappings

| Oracle Forms Trigger | Mendix Implementation |
|---------------------|----------------------|
| `WHEN-NEW-FORM-INSTANCE` | Page load microflow (data source) |
| `WHEN-NEW-RECORD-INSTANCE` | OnChange microflow on data source |
| `WHEN-VALIDATE-ITEM` | OnChange microflow or validation in save |
| `WHEN-VALIDATE-RECORD` | Validation microflow before save |
| `POST-QUERY` | Microflow data source with transformation |
| `PRE-INSERT` / `PRE-UPDATE` | Before commit event handler |
| `POST-INSERT` / `POST-UPDATE` | After commit event handler |
| `KEY-COMMIT` | Save button action microflow |
| `ON-ERROR` | `ON ERROR { ... }` blocks |

## Migration Checklist

Before starting migration:
- [ ] Export Oracle Forms XML (.xml) or use Forms2XML utility
- [ ] Document all triggers and their purposes
- [ ] Map database tables to Mendix entities
- [ ] Identify LOVs and map to enumerations
- [ ] Check for structural keyword conflicts (Create, Delete, Begin, End, Return)

During migration:
- [ ] Create entities first (Phase 1)
- [ ] Create microflows second (Phase 2)
- [ ] Create pages last (Phase 3) - they can reference microflows
- [ ] Test validation patterns thoroughly
- [ ] Use `VALIDATION FEEDBACK $Entity/Attribute MESSAGE 'message'` for all validations

After migration:
- [ ] Run `mxcli check script.mdl -p app.mpr --references`
- [ ] Open in Mendix Studio Pro to verify
- [ ] Test all validation scenarios
- [ ] Verify master-detail relationships work correctly

## Common Migration Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Parse error: mismatched input 'Create'" | Structural keyword as attribute | Use `"Create"` (quoted) or rename |
| "microflow not found" | Referenced before created | Move microflow definition before page |
| "page not found" | Referenced before created | Move page definition earlier |
| "VALIDATION FEEDBACK requires attribute" | Missing attribute path | Use `VALIDATION FEEDBACK $Entity/Attribute MESSAGE 'msg'` |
| CE0117 "Error in expression" | Missing module prefix | Use fully qualified names |

## Tips for Success

1. **Plan attribute names carefully**: Most words work unquoted; only structural keywords (`Create`, `Delete`, `Begin`, `End`, `Return`) need quoting
2. **Organize scripts by phase**: Entities → Microflows → Pages
3. **Test incrementally**: Migrate one form at a time
4. **Keep validation close to logic**: Embed validation in save microflows
5. **Document mappings**: Track which Oracle Forms items map to which Mendix elements
6. **Use meaningful names**: `ACT_Customer_Save` not `SUB_SAVE`
7. **Leverage CRUD generation**: Use `/create-crud` skill for standard operations

## Related Skills

- [/write-microflows](./write-microflows.md) - Detailed microflow syntax
- [/create-crud](./create-crud.md) - Generate CRUD operations
- [/overview-pages](./overview-pages.md) - Page building patterns
- [/master-detail-pages](./master-detail-pages.md) - Master-detail layouts
