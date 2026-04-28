# MDL Syntax Validation Skill

This skill ensures MDL scripts are validated before presenting them to users or executing them.

## When to Use This Skill

**ALWAYS** use this skill before:
- Presenting MDL code to users
- Executing MDL scripts via `mxcli exec`
- Committing MDL files to version control

## Pre-Flight Validation Checklist

Before writing any MDL, verify these requirements:

### 1. Check Supported Syntax

**Supported in Microflows:**
- `DECLARE $Var Type = value;` (primitives)
- `DECLARE $Entity Module.Entity;` (entities - no AS keyword, no = empty)
- `DECLARE $List List of Module.Entity = empty;` (lists)
- `SET $Var = expression;`
- `$Var = CREATE Module.Entity (Attr = value);`
- `CHANGE $Entity (Attr = value);`
- `COMMIT $Entity [WITH EVENTS] [REFRESH];`
- `DELETE $Entity;`
- `RETRIEVE $Var FROM Module.Entity [WHERE condition];`
- `$Result = CALL MICROFLOW Module.Name (Param = $value);` (NOT `SET $Result = ...`)
- `$Result = CALL NANOFLOW Module.Name (Param = $value);`
- `SHOW PAGE Module.PageName ($Param = $value);`
- `CLOSE PAGE;`
- `VALIDATION FEEDBACK $Entity/Attribute MESSAGE 'message';`
- `LOG INFO|WARNING|ERROR [NODE 'name'] 'message';`
- `IF condition THEN ... [ELSE ...] END IF;`
- `LOOP $Item IN $List BEGIN ... END LOOP;`
- `RETURN $value;`
- `ON ERROR CONTINUE|ROLLBACK|{ handler };`

**Now Supported (previously not):**
- `ROLLBACK $Entity [REFRESH];` - Reverts uncommitted changes
- `RETRIEVE ... LIMIT n` - Returns single entity when `LIMIT 1`
- `Boolean` without `DEFAULT` - Auto-defaults to `false`
- `ButtonStyle: Warning` and `ButtonStyle: Info` - Now parse correctly
- Keywords as attribute names - `Caption`, `Label`, `Title`, `Text`, `Content`, `Format`, `Range`, `Source`, `Check`, etc. all work unquoted

**NOT Supported (will cause errors):**
- `SET $var = CALL MICROFLOW ...` - Use `$var = CALL MICROFLOW ...` (no SET)
- `WHILE ... END WHILE` - Use `LOOP` with lists
- `CASE ... WHEN ... END CASE` - Use nested `IF`
- `TRY ... CATCH` - Use `ON ERROR` blocks
- `BREAK` / `CONTINUE` - Not implemented
- `COMMIT MESSAGE 'text'` - Not in current grammar (session command only)

### 2. Quote All Identifiers

**Best practice: Always quote all identifiers** (entity names, attribute names, parameter names) with double quotes. This eliminates all reserved keyword conflicts and is always safe — quotes are stripped automatically by the parser.

```sql
CREATE PERSISTENT ENTITY Module."Customer" (
  "Name": String(200),
  "Status": String(50),
  "Create": DateTime
);
```

Both `"Name"` and `` `Name` `` syntax are supported. Prefer double quotes for consistency.

Run `mxcli syntax keywords` for the full list of 320+ reserved keywords.

### 3. Validate with mxcli

**Always run these checks:**

```bash
# Step 1: Syntax check (no project needed)
./bin/mxcli check script.mdl

# Step 2: Reference validation (needs project)
# Validates microflow bodies, entity/enum references, AND widget tree references
# (DataSource microflow/nanoflow/entity, Action page/microflow, Snippet refs)
./bin/mxcli check script.mdl -p app.mpr --references
```

### 4. Common Error Patterns

| Error Message | Likely Cause | Fix |
|---------------|--------------|-----|
| `mismatched input 'SET'` after `CALL MICROFLOW` | SET not valid with CALL | Use `$var = CALL MICROFLOW ...` |
| `mismatched input 'Create'` | Structural keyword as identifier | Use `"Create"` (quoted) or rename |
| `no viable alternative at input` | Unsupported syntax | Check supported statements list |
| `microflow not found` | Referenced before created | Move microflow definition earlier or check spelling |
| `page not found` | Page doesn't exist | Check qualified name with `--references` |
| `entity not found` | Typo or wrong module | Use fully qualified name |

## Validation Workflow

### Before Writing MDL

1. **Read the skill files:**
   ```bash
   cat .claude/skills/write-microflows.md
   cat .claude/skills/overview-pages.md
   ```

2. **Check help for specific syntax:**
   ```bash
   ./bin/mxcli syntax microflow
   ./bin/mxcli syntax page
   ./bin/mxcli syntax entity
   ```

### After Writing MDL

1. **Save to a file:**
   ```bash
   cat > script.mdl << 'EOF'
   -- Your MDL here
   EOF
   ```

2. **Run syntax check:**
   ```bash
   ./bin/mxcli check script.mdl
   ```

3. **If errors, check specific syntax:**
   ```bash
   ./bin/mxcli syntax keywords    # Reserved words
   ./bin/mxcli syntax microflow   # Microflow syntax
   ```

4. **Run reference check (with project):**
   ```bash
   ./bin/mxcli check script.mdl -p app.mpr --references
   ```

5. **Execute only after all checks pass:**
   ```bash
   ./bin/mxcli exec script.mdl -p app.mpr
   ```

## Script Execution Behavior

**IMPORTANT: Script execution is atomic per statement, NOT per script.**

When a script fails on statement N, statements 1 through N-1 have already been committed:

```
Statement 1: CREATE MODULE ✓ (committed)
Statement 2: CREATE ENTITY ✓ (committed)
Statement 3: CREATE ASSOCIATION ✓ (committed)
Statement 4: CREATE VIEW ENTITY ✗ (failed - execution stops here)
Statement 5: CREATE PAGE (never executed)
```

**Recommendations:**
1. Split scripts into phases when experimenting with uncertain syntax
2. Use `CREATE OR REPLACE` to make scripts idempotent
3. Test new syntax patterns with minimal scripts first
4. Keep a backup of your project before running large scripts

## Script Organization

Organize scripts in dependency order:

```mdl
-- ============================================
-- PHASE 1: Enumerations (no dependencies)
-- ============================================
CREATE ENUMERATION Module.Status (
  Active = 'Active',
  Inactive = 'Inactive'
);
/

-- ============================================
-- PHASE 2: Entities (depend on enumerations)
-- ============================================
CREATE PERSISTENT ENTITY Module.Customer (
  Name: String(200),
  Status: Module.Status
);
/

-- ============================================
-- PHASE 3: Associations (depend on entities)
-- ============================================
CREATE ASSOCIATION Module.Order_Customer (
  Module.Order [*] -> Module.Customer [1]
);
/

-- ============================================
-- PHASE 4: Microflows (depend on entities)
-- ============================================
CREATE MICROFLOW Module.ACT_Save ($Customer: Module.Customer)
RETURNS Boolean AS $Success
BEGIN
  DECLARE $Success Boolean = false;
  COMMIT $Customer;
  SET $Success = true;
  RETURN $Success;
END;
/

-- ============================================
-- PHASE 5: Pages (depend on microflows)
-- ============================================
CREATE PAGE Module.Customer_Edit
LAYOUT Atlas_Default
TITLE 'Edit Customer'
PARAMETER $Customer: Module.Customer
WIDGETS (
  -- Can reference microflows created in Phase 4
  BUTTON 'Save' CALL MICROFLOW Module.ACT_Save (Customer = $Customer)
);
/
```

## Troubleshooting Parse Errors

### Error: "mismatched input 'X'"

The word `X` is either:
1. A reserved word - rename the identifier
2. Unsupported syntax - check the supported statements list
3. A typo - check spelling

### Error: "no viable alternative at input"

The parser expected something different:
1. Check for missing semicolons
2. Check for missing `END IF`, `END LOOP`, etc.
3. Verify statement syntax against the reference

### Error: "extraneous input"

Extra tokens found:
1. Check for stray characters
2. Check for duplicate semicolons
3. Verify string quotes are balanced

## Related Skills

- [/write-microflows](./write-microflows.md) - Detailed microflow syntax
- [/overview-pages](./overview-pages.md) - Page building syntax
- [/migrate-oracle-forms](./migrate-oracle-forms.md) - Migration-specific guidance
