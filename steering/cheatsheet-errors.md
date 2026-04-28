# MDL Common Errors Cheatsheet

Quick fixes for common MDL syntax errors.

## Variable Errors

### "Variable 'X' is not declared"

**Problem**: Using SET on a variable that wasn't declared.

```mdl
-- WRONG
IF $Value > 10 THEN
  SET $IsValid = false;  -- ERROR: $IsValid not declared
END IF;
```

**Fix**: Add DECLARE before SET.

```mdl
-- CORRECT
DECLARE $IsValid Boolean = true;
IF $Value > 10 THEN
  SET $IsValid = false;
END IF;
```

### "Selected type is not allowed" (CE0053)

**Problem**: Wrong syntax for entity type declaration.

```mdl
-- WRONG: Missing AS keyword
DECLARE $Product Module.Product = empty;
```

**Fix**: Use AS keyword for entity types.

```mdl
-- CORRECT
DECLARE $Product AS Module.Product;
```

## Expression Errors

### "Error in expression" (CE0117)

**Problem**: Using unqualified association name in XPath.

```mdl
-- WRONG: Missing module qualification
SET $Name = $Order/Customer/Name;
```

**Fix**: Use fully qualified association name.

```mdl
-- CORRECT: Module.AssociationName
SET $Name = $Order/Shop.Order_Customer/Name;
```

### "Type mismatch" in enum comparison

**Problem**: Comparing enumeration with string literal.

```mdl
-- WRONG: String literal instead of enum value
IF $Task/Status = 'Completed' THEN
```

**Fix**: Use qualified enumeration value.

```mdl
-- CORRECT: Module.Enumeration.Value
IF $Task/Status = Module.TaskStatus.Completed THEN
```

## Control Flow Errors

### "Activity cannot be the last object" (CE0105)

**Problem**: Missing RETURN statement.

```mdl
-- WRONG: No RETURN
BEGIN
  DECLARE $Result Boolean = true;
  LOG INFO 'Done';
  -- Missing RETURN!
END;
```

**Fix**: Add RETURN statement.

```mdl
-- CORRECT
BEGIN
  DECLARE $Result Boolean = true;
  LOG INFO 'Done';
  RETURN $Result;
END;
```

### "Action activity is unreachable" (CE0104)

**Problem**: Code after RETURN statement.

```mdl
-- WRONG: Code after RETURN
IF $Value < 0 THEN
  RETURN false;
  LOG INFO 'Negative';  -- Unreachable!
END IF;
```

**Fix**: Move code before RETURN.

```mdl
-- CORRECT
IF $Value < 0 THEN
  LOG INFO 'Negative';
  RETURN false;
END IF;
```

## Syntax Errors

### Division operator

```mdl
-- WRONG: Using / for division
SET $Average = $Total / $Count;

-- CORRECT: Use 'div' keyword
SET $Average = $Total div $Count;
```

### Missing END IF / END LOOP

```mdl
-- WRONG: Missing END IF
IF $Value > 0 THEN
  SET $Positive = true;
-- Missing END IF!

-- CORRECT
IF $Value > 0 THEN
  SET $Positive = true;
END IF;
```

### Missing semicolons

```mdl
-- WRONG: Missing semicolon
DECLARE $Count Integer = 0
SET $Count = 1

-- CORRECT
DECLARE $Count Integer = 0;
SET $Count = 1;
```

## Reference Errors

### "Module not found"

**Problem**: Using non-existent module name.

**Fix**: Check module exists with `SHOW MODULES`.

### "Entity not found"

**Problem**: Using non-existent entity name.

**Fix**:
1. Check entity exists: `SHOW ENTITIES IN ModuleName`
2. Use fully qualified name: `Module.EntityName`

### "Microflow not found"

**Problem**: Calling non-existent microflow.

**Fix**:
1. Check microflow exists: `SHOW MICROFLOWS IN ModuleName`
2. Use fully qualified name: `Module.MicroflowName`

## Studio Pro Error Code Reference

| Code | Message | Common Cause |
|------|---------|--------------|
| CE0053 | Selected type is not allowed | Missing AS for entity type |
| CE0104 | Action activity is unreachable | Code after RETURN |
| CE0105 | Must end with end event | Missing RETURN |
| CE0117 | Error in expression | Unqualified association path |
| CW0094 | Variable never used | Unused parameter/variable |

## Quick Validation Checklist

Before executing MDL:

- [ ] All entity types use `DECLARE $var AS Module.Entity`
- [ ] All SET targets have prior DECLARE
- [ ] Association paths are qualified: `$var/Module.Assoc/Attr`
- [ ] Enum comparisons use `Module.Enum.Value`
- [ ] Every flow path ends with RETURN
- [ ] Division uses `div` not `/`
- [ ] All statements end with `;`
- [ ] IF/LOOP properly closed with END IF/END LOOP
