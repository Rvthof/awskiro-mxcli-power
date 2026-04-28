# Validation Microflows Skill

This skill provides guidance for creating validation microflows in MDL that validate user input on NewEdit pages and provide feedback to users.

## When to Use This Skill

Use this skill when:
- Creating validation logic for NewEdit pages
- Implementing attribute validation with feedback messages
- Building conditional validation chains
- Creating action microflows that call validation microflows

## The Validation Pattern

Mendix validation follows a two-microflow pattern:

1. **VAL_Entity_Action** - The validation microflow that:
   - Takes an entity object as parameter
   - Validates each required field
   - Shows validation feedback on invalid fields
   - Returns a Boolean indicating overall validity

2. **ACT_Entity_Action** - The action microflow that:
   - Calls the validation microflow
   - Only proceeds with save/commit if validation passes
   - Closes the page on success

## MDL Syntax

### VALIDATION FEEDBACK Statement

```mdl
VALIDATION FEEDBACK $VariableName/AttributeName MESSAGE 'Error message';
```

With template arguments (for dynamic messages):
```mdl
VALIDATION FEEDBACK $VariableName/AttributeName MESSAGE '{1}' OBJECTS [$MessageVariable];
```

### CLOSE PAGE Statement

```mdl
CLOSE PAGE;
```

Or to close multiple pages:
```mdl
CLOSE PAGE 2;
```

## Complete Example

### Validation Microflow (VAL_Car_NewEdit)

```mdl
/**
 * Validates a Car entity for NewEdit operations
 *
 * Performs validation on all required fields and displays
 * appropriate error messages to the user.
 *
 * @param $Car The Car entity to validate
 * @returns Boolean indicating if all validations passed
 */
CREATE MICROFLOW MdlTemplates.VAL_Car_NewEdit (
  $Car: MdlTemplates.Car
)
RETURNS Boolean AS $IsValid
FOLDER 'OverviewPages'
BEGIN
  -- Initialize validation flag
  DECLARE $IsValid Boolean = true;

  -- Validate Brand (required text field)
  IF trim($Car/Brand) = '' THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Car/Brand MESSAGE 'Brand is required';
  END IF;

  -- Validate Model (required text field)
  IF trim($Car/Model) = '' THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Car/Model MESSAGE 'Model is required';
  END IF;

  -- Validate Price (required, must be positive)
  IF $Car/Price = empty THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Car/Price MESSAGE 'Price is required';
  ELSE
    IF $Car/Price <= 0 THEN
      SET $IsValid = false;
      VALIDATION FEEDBACK $Car/Price MESSAGE 'Price must be greater than 0';
    END IF;
  END IF;

  -- Validate enumeration (required)
  IF $Car/CarType = empty THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Car/CarType MESSAGE 'Car type is required';
  END IF;

  RETURN $IsValid;
END;
/
```

### Action Microflow (ACT_Car_NewEdit)

```mdl
/**
 * Handles the Save action for Car NewEdit page
 *
 * Validates the Car, commits it if valid, and closes the page.
 *
 * @param $Car The Car entity to save
 * @returns Boolean indicating success
 */
CREATE MICROFLOW MdlTemplates.ACT_Car_NewEdit (
  $Car: MdlTemplates.Car
)
RETURNS Boolean AS $IsValid
FOLDER 'OverviewPages'
BEGIN
  -- Call validation microflow
  $IsValid = CALL MICROFLOW MdlTemplates.VAL_Car_NewEdit($param = $Car);

  -- Only save if validation passed
  IF $IsValid THEN
    COMMIT $Car;
    CLOSE PAGE;
  END IF;

  RETURN $IsValid;
END;
/
```

## Validation Patterns

### Simple Required Field Validation

```mdl
IF trim($Entity/TextField) = '' THEN
  SET $IsValid = false;
  VALIDATION FEEDBACK $Entity/TextField MESSAGE 'This field is required';
END IF;
```

### Numeric Range Validation

```mdl
IF $Entity/Amount != empty THEN
  IF $Entity/Amount < 0 OR $Entity/Amount > 1000 THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Entity/Amount MESSAGE 'Amount must be between 0 and 1000';
  END IF;
END IF;
```

### Enumeration Required Validation

```mdl
IF $Entity/Status = empty THEN
  SET $IsValid = false;
  VALIDATION FEEDBACK $Entity/Status MESSAGE 'Status is required';
END IF;
```

### Enumeration Value Comparison

**IMPORTANT**: When comparing enumeration values, use the fully qualified enumeration value, NOT a string literal.

```mdl
-- CORRECT: Use fully qualified enumeration value
IF $Task/TaskStatus = Module.TaskStatus.Completed THEN
  -- Task is completed
END IF;

IF $Task/TaskStatus != Module.TaskStatus.Cancelled THEN
  -- Task is not cancelled
END IF;

-- WRONG: Do NOT use string literals for enumeration comparison
-- IF $Task/TaskStatus = 'Completed' THEN  -- This is incorrect!
```

### Conditional Validation Based on Enumeration

```mdl
-- Validate CompletedDate only when status is Completed
IF $Task/TaskStatus = Module.TaskStatus.Completed THEN
  IF $Task/CompletedDate = empty THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Task/CompletedDate MESSAGE 'Completed date is required';
  END IF;
END IF;

-- Validate DueDate for active (non-completed, non-cancelled) tasks
IF $Task/TaskStatus != Module.TaskStatus.Completed AND $Task/TaskStatus != Module.TaskStatus.Cancelled THEN
  IF $Task/DueDate = empty THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Task/DueDate MESSAGE 'Due date is required for active tasks';
  END IF;
END IF;
```

### Date Validation

```mdl
IF $Entity/StartDate != empty AND $Entity/EndDate != empty THEN
  IF $Entity/EndDate < $Entity/StartDate THEN
    SET $IsValid = false;
    VALIDATION FEEDBACK $Entity/EndDate MESSAGE 'End date must be after start date';
  END IF;
END IF;
```

### Dynamic Message with Template Arguments

```mdl
-- Build validation message with dynamic content
DECLARE $ValidationMessage String = '';
IF $Entity/Value < $Entity/MinValue THEN
  SET $ValidationMessage = 'Value must be at least ' + toString($Entity/MinValue);
END IF;
IF $Entity/Value > $Entity/MaxValue THEN
  SET $ValidationMessage = if trim($ValidationMessage) = ''
    then 'Value must be at most ' + toString($Entity/MaxValue)
    else $ValidationMessage + '. Value must be at most ' + toString($Entity/MaxValue);
END IF;

IF trim($ValidationMessage) != '' THEN
  SET $IsValid = false;
  VALIDATION FEEDBACK $Entity/Value MESSAGE '{1}' OBJECTS [$ValidationMessage];
END IF;
```

### Association Validation

```mdl
-- Validate that an association is set
IF $Order/Order_Customer = empty THEN
  SET $IsValid = false;
  VALIDATION FEEDBACK $Order/Module.Order_Customer MESSAGE 'Customer is required';
END IF;
```

## Implementation Checklist

When implementing validation microflows:

1. **Initialize the validation flag**: Always start with `DECLARE $IsValid Boolean = true;`

2. **Declare all variables before using SET**: You must use `DECLARE` before `SET` for primitive variables. Parameters are automatically available but local variables require declaration.

3. **Validate all required fields**: Check each attribute that needs validation

4. **Set flag to false on error**: `SET $IsValid = false;` before showing feedback

5. **Show clear error messages**: Use `VALIDATION FEEDBACK` with descriptive messages

6. **Return the validation flag**: End with `RETURN $IsValid;`

7. **Handle nullable fields**: Check for `empty` before validating nullable fields

8. **Use appropriate validation order**: Validate presence before other constraints

**Important**: The script executor validates that all variables used with `SET` are declared. If you use `SET $Var = ...` without a prior `DECLARE $Var Type = ...`, you will receive an error like:
```
variable '$Var' is not declared. Use DECLARE $Var: <Type> before using SET
```

## Files Modified

This feature is implemented in:
- `mdl/grammar/MDL.g4` - ANTLR4 grammar with VALIDATION FEEDBACK tokens
- `mdl/ast/ast_microflow.go` - AST type definitions (MfValidationFeedbackStmt)
- `mdl/visitor/visitor_microflow_statements.go` - ANTLR listener to build AST
- `mdl/executor/cmd_microflows_builder.go` - Flow builder with variable validation
- `mdl/executor/cmd_microflows_show.go` - DESCRIBE formatter for MDL output
- `sdk/mpr/writer_microflow.go` - BSON serialization for ValidationFeedbackAction
- `sdk/microflows/microflows_actions.go` - ValidationFeedbackAction struct
