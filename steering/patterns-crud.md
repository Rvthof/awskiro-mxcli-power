# CRUD Action Patterns

Standard patterns for Create, Read, Update, Delete operations on entities.

## Naming Conventions

| Prefix | Purpose | Example |
|--------|---------|---------|
| `ACT_` | Action microflow (page button) | `ACT_Customer_Save` |
| `VAL_` | Validation microflow | `VAL_Customer_Save` |
| `DS_` | Data source microflow | `DS_Customer_GetAll` |
| `SUB_` | Sub-microflow (internal) | `SUB_Customer_SendEmail` |

## Save Pattern (Create/Update)

Used for Save buttons on NewEdit pages.

```mdl
/**
 * Save action for Customer NewEdit page
 * Validates, commits, and closes the page
 *
 * @param $Customer The customer to save
 * @returns true if saved successfully
 */
CREATE MICROFLOW Module.ACT_Customer_Save (
  $Customer: Module.Customer
)
RETURNS Boolean
BEGIN
  -- Validate first
  DECLARE $IsValid Boolean = true;
  $IsValid = CALL MICROFLOW Module.VAL_Customer_Save($Customer = $Customer);

  IF $IsValid THEN
    COMMIT $Customer WITH EVENTS;
    CLOSE PAGE;
  END IF;

  RETURN $IsValid;
END;
/
```

## Validation Pattern

Companion validation microflow for Save actions.

```mdl
/**
 * Validate Customer before save
 *
 * @param $Customer The customer to validate
 * @returns true if valid
 */
CREATE MICROFLOW Module.VAL_Customer_Save (
  $Customer: Module.Customer
)
RETURNS Boolean
BEGIN
  DECLARE $IsValid Boolean = true;

  -- Required field validation
  IF $Customer/Name = empty THEN
    VALIDATION FEEDBACK $Customer/Name MESSAGE 'Name is required';
    SET $IsValid = false;
  END IF;

  IF $Customer/Email = empty THEN
    VALIDATION FEEDBACK $Customer/Email MESSAGE 'Email is required';
    SET $IsValid = false;
  END IF;

  -- Business rule validation
  IF $Customer/CreditLimit < 0 THEN
    VALIDATION FEEDBACK $Customer/CreditLimit MESSAGE 'Credit limit cannot be negative';
    SET $IsValid = false;
  END IF;

  RETURN $IsValid;
END;
/
```

## Delete Pattern

Used for Delete buttons with confirmation.

```mdl
/**
 * Delete a customer
 * Called after user confirms deletion
 *
 * @param $Customer The customer to delete
 * @returns true if deleted
 */
CREATE MICROFLOW Module.ACT_Customer_Delete (
  $Customer: Module.Customer
)
RETURNS Boolean
BEGIN
  DELETE $Customer;
  CLOSE PAGE;
  RETURN true;
END;
/
```

## Cancel Pattern

Used for Cancel buttons (discard changes).

```mdl
/**
 * Cancel editing and close page
 * Discards uncommitted changes
 *
 * @param $Customer The customer being edited
 * @returns true
 */
CREATE MICROFLOW Module.ACT_Customer_Cancel (
  $Customer: Module.Customer
)
RETURNS Boolean
BEGIN
  ROLLBACK $Customer;
  CLOSE PAGE;
  RETURN true;
END;
/
```

## Create New Pattern

Used for New/Add buttons on overview pages.

```mdl
/**
 * Create new customer and open edit page
 *
 * @returns true
 */
CREATE MICROFLOW Module.ACT_Customer_New ()
RETURNS Boolean
BEGIN
  DECLARE $NewCustomer AS Module.Customer;

  $NewCustomer = CREATE Module.Customer (
    IsActive = true,
    CreatedDate = [%CurrentDateTime%]
  );

  SHOW PAGE Module.Customer_NewEdit ($Customer = $NewCustomer);
  RETURN true;
END;
/
```

## Data Source Pattern

Used for data grid/list view sources.

```mdl
/**
 * Get all active customers
 * Used as data source for Customer overview
 *
 * @returns List of active customers
 */
CREATE MICROFLOW Module.DS_Customer_GetActive ()
RETURNS List of Module.Customer
BEGIN
  DECLARE $Customers List of Module.Customer = empty;

  RETRIEVE $Customers FROM Module.Customer
    WHERE IsActive = true;

  RETURN $Customers;
END;
/
```

## Edit Pattern

Open existing entity for editing.

```mdl
/**
 * Open customer for editing
 *
 * @param $Customer The customer to edit
 * @returns true
 */
CREATE MICROFLOW Module.ACT_Customer_Edit (
  $Customer: Module.Customer
)
RETURNS Boolean
BEGIN
  SHOW PAGE Module.Customer_NewEdit ($Customer = $Customer);
  RETURN true;
END;
/
```

## Complete CRUD Set

For a typical entity, create these microflows:

| Microflow | Purpose | Parameters |
|-----------|---------|------------|
| `ACT_Entity_New` | Create new | None |
| `ACT_Entity_Edit` | Open for edit | `$Entity` |
| `ACT_Entity_Save` | Save changes | `$Entity` |
| `VAL_Entity_Save` | Validate | `$Entity` |
| `ACT_Entity_Delete` | Delete | `$Entity` |
| `ACT_Entity_Cancel` | Cancel edit | `$Entity` |
| `DS_Entity_GetAll` | List all | None |

## Best Practices

1. **Always validate before commit**: Call VAL_ microflow in ACT_Save
2. **Use WITH EVENTS**: `COMMIT $Entity WITH EVENTS` triggers event handlers
3. **Close page on success**: Use `CLOSE PAGE` after successful save/delete
4. **Rollback on cancel**: Use `ROLLBACK $Entity` to discard changes
5. **Initialize defaults**: Set default values in ACT_New microflow
6. **Return Boolean**: All action microflows should return success status
