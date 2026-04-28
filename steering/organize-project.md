# Project Organization: Folders and Moving Documents

This skill covers organizing Mendix project documents (pages, microflows, snippets, nanoflows) into folders and moving them between folders and modules.

## When to Use This Skill

Use this skill when:
- Organizing documents into folder hierarchies within a module
- Moving documents between folders
- Moving documents between modules
- Restructuring a project for better maintainability
- Setting up folder conventions for a new module

## Folder Conventions

Organize by **functional grouping** — keep all artifacts for a feature together, not separated by document type. This way, a developer working on "Customer" finds everything in one place: pages, microflows, snippets, and validation logic.

Recommended folder structure within a module:

```
CRM/
├── Customer/
│   ├── Customer_Overview        -- Overview page
│   ├── Customer_NewEdit         -- Edit page
│   ├── CustomerCard             -- Snippet
│   ├── ACT_Customer_Save        -- Save microflow
│   ├── ACT_Customer_Delete      -- Delete microflow
│   ├── ACT_Customer_New         -- New microflow
│   ├── VAL_Customer             -- Validation microflow
│   └── DS_Customer_Filter       -- Data source microflow
├── Order/
│   ├── Order_Overview
│   ├── Order_NewEdit
│   ├── ACT_Order_Save
│   └── VAL_Order
└── Shared/                      -- Cross-cutting concerns
    ├── SUB_SendNotification
    └── Navigation_Snippet
```

**Why functional grouping over type grouping:**
- All related artifacts are in one place — easier to navigate and review
- Adding or removing a feature is a single folder operation
- Naming prefixes (ACT_, VAL_, SUB_, DS_) already indicate document type
- Mirrors how developers think: "I'm working on Customer" not "I'm working on microflows"

Adapt to your project's conventions. The key is consistency across modules.

## Creating Documents in Folders

### Microflows

Use the `FOLDER` keyword after the return type, before `BEGIN`:

```mdl
CREATE MICROFLOW MyModule.ACT_ProcessOrder ($Order: MyModule.Order)
RETURNS Boolean AS $Success
FOLDER 'Order'
BEGIN
  COMMIT $Order;
  RETURN true;
END;
```

### Pages

Use the `Folder` property inside the page properties:

```sql
CREATE PAGE MyModule.Customer_Overview
(
  Title: 'Customer Overview',
  Layout: Atlas_Core.Atlas_Default,
  Folder: 'Customer'
)
{
  -- widgets
}
```

### Snippets

```sql
CREATE SNIPPET MyModule.CustomerCard
(
  Folder: 'Customer'
)
{
  -- widgets
}
```

### Nested Folders

Use `/` to create nested folder paths. Missing folders are created automatically:

```mdl
-- Creates 'Order', then 'Order/Batch' if they don't exist
CREATE MICROFLOW MyModule.ACT_BatchProcess ($List: List of MyModule.Order)
FOLDER 'Order/Batch'
BEGIN
  LOOP $Order IN $List BEGIN
    COMMIT $Order;
  END LOOP;
  RETURN;
END;
```

## Moving Documents

The `MOVE` command relocates existing documents between folders and modules.

### Move to a Folder (Same Module)

```mdl
MOVE PAGE MyModule.CustomerEdit TO FOLDER 'Customer';
MOVE MICROFLOW MyModule.ACT_ProcessOrder TO FOLDER 'Order';
MOVE SNIPPET MyModule.NavigationMenu TO FOLDER 'Shared';
MOVE NANOFLOW MyModule.NAV_OpenCustomer TO FOLDER 'Customer';
MOVE ENUMERATION MyModule.OrderStatus TO FOLDER 'Shared';
```

### Move to Module Root (Out of Folder)

```mdl
MOVE PAGE MyModule.CustomerEdit TO MyModule;
```

### Move Across Modules

```mdl
-- Move to another module's root
MOVE PAGE OldModule.CustomerPage TO NewModule;

-- Move to a folder in another module
MOVE PAGE OldModule.CustomerPage TO FOLDER 'Pages' IN NewModule;
```

### Cross-Module Move Warning

Cross-module moves change the qualified name (e.g., `OldModule.CustomerPage` becomes `NewModule.CustomerPage`). This **breaks by-name references** such as:
- Microflows calling `SHOW PAGE OldModule.CustomerPage`
- Other microflows calling `CALL MICROFLOW OldModule.SomeMicroflow`
- Widget actions referencing the old qualified name

**Always check impact before cross-module moves:**

```mdl
SHOW IMPACT OF OldModule.CustomerPage;
-- Review the output, then move if safe:
MOVE PAGE OldModule.CustomerPage TO NewModule;
```

## Folder Rules

- Folder names are **case-sensitive**
- Use `/` as separator for nested folders: `'Parent/Child/Grandchild'`
- Folders are **created automatically** if they don't exist
- Moving to a folder that doesn't exist creates it
- Empty folders are preserved in the project

## Supported Document Types

| Document Type | FOLDER on Create | MOVE Command |
|---------------|-----------------|--------------|
| Page          | `Folder: 'path'` (property) | `MOVE PAGE ...` |
| Microflow     | `FOLDER 'path'` (keyword) | `MOVE MICROFLOW ...` |
| Nanoflow      | `FOLDER 'path'` (keyword) | `MOVE NANOFLOW ...` |
| Snippet       | `Folder: 'path'` (property) | `MOVE SNIPPET ...` |
| Enumeration   | N/A | `MOVE ENUMERATION ...` |
| Entity        | N/A | `MOVE ENTITY ...` (module only, no folders) |

**Note:** Pages and snippets use property syntax (`Folder: 'path'` inside parentheses). Microflows and nanoflows use keyword syntax (`FOLDER 'path'` before `BEGIN`). Entities are embedded in domain models and can only be moved to a different module (no folder support).

## Example: Reorganize a Module

```mdl
-- Group all Customer artifacts together
MOVE PAGE CRM.Customer_Overview TO FOLDER 'Customer';
MOVE PAGE CRM.Customer_NewEdit TO FOLDER 'Customer';
MOVE MICROFLOW CRM.ACT_Customer_Save TO FOLDER 'Customer';
MOVE MICROFLOW CRM.ACT_Customer_Delete TO FOLDER 'Customer';
MOVE MICROFLOW CRM.ACT_Customer_New TO FOLDER 'Customer';
MOVE MICROFLOW CRM.VAL_Customer TO FOLDER 'Customer';
MOVE SNIPPET CRM.CustomerCard TO FOLDER 'Customer';

-- Group all Order artifacts together
MOVE PAGE CRM.Order_Overview TO FOLDER 'Order';
MOVE PAGE CRM.Order_NewEdit TO FOLDER 'Order';
MOVE MICROFLOW CRM.ACT_Order_Save TO FOLDER 'Order';
MOVE MICROFLOW CRM.ACT_Order_Process TO FOLDER 'Order/Processing';

-- Move shared artifacts to a Shared folder or common module
SHOW IMPACT OF CRM.Header_Snippet;
MOVE SNIPPET CRM.Header_Snippet TO FOLDER 'Shared' IN Common;

-- Move entity to different module
SHOW IMPACT OF CRM.Customer;
MOVE ENTITY CRM.Customer TO CustomerModule;

-- Move enumeration to different module
MOVE ENUMERATION CRM.OrderStatus TO SharedModule;
```

## Moving Folders

Use `MOVE FOLDER` to reorganize folders. Syntax matches document moves: `Module.FolderName`.

```sql
-- Move a folder into another folder
MOVE FOLDER MyModule.Resources TO FOLDER 'Archive';

-- Move a nested folder (use double quotes for paths with /)
MOVE FOLDER MyModule."Orders/Archive" TO MyModule;

-- Move a folder to a different module
MOVE FOLDER MyModule.SharedWidgets TO CommonModule;

-- Move a folder into a folder in another module
MOVE FOLDER MyModule.Templates TO FOLDER 'Shared' IN CommonModule;
```

## Deleting Folders

Use `DROP FOLDER` to remove empty folders. The folder must not contain any documents or sub-folders.

```sql
-- Drop an empty folder
DROP FOLDER 'OldPages' IN MyModule;

-- Drop a nested folder (only the leaf is removed)
DROP FOLDER 'Orders/Archive' IN MyModule;

-- Move contents out first, then drop
MOVE MICROFLOW MyModule.ACT_Process TO MyModule;
DROP FOLDER 'Processing' IN MyModule;
```

## Validation Checklist

- [ ] Folder paths use `/` separator (not `\`)
- [ ] FOLDER keyword placement is correct (before BEGIN for microflows, inside properties for pages)
- [ ] Cross-module moves: checked impact with `SHOW IMPACT OF` first
- [ ] Folder naming is consistent across modules
- [ ] DROP FOLDER: verify folder is empty before dropping
