# Overview Pages - CRUD Page Pattern

## Overview

Standard pattern for creating CRUD (Create, Read, Update, Delete) pages in Mendix using MDL syntax. This pattern consists of:

1. **Navigation Snippet** - Reusable menu for consistent navigation
2. **Overview Page** - Lists all objects with a DataGrid and navigation snippet
3. **NewEdit Page** - Form for creating/editing a single object

## Pattern Summary

| Component | Type | Purpose | Key Widgets |
|-----------|------|---------|-------------|
| `Entity_Menu` | Snippet | Vertical sidebar navigation | NAVIGATIONLIST with ITEM actions |
| `Entity_Overview` | Page | List all records | SNIPPETCALL (sidebar), DATAGRID, Heading |
| `Entity_NewEdit` | Page | Create/Edit form | DataView, Input widgets, Save/Cancel |

## Navigation Menu Snippet

Create a reusable navigation snippet using NAVIGATIONLIST for vertical sidebar menus:

```sql
CREATE SNIPPET Module.Entity_Menu
{
  NAVIGATIONLIST navMenu {
    ITEM itemCustomers (Caption: 'Customers', Action: SHOW_PAGE Module.Customer_Overview)
    ITEM itemOrders (Caption: 'Orders', Action: SHOW_PAGE Module.Order_Overview)
    ITEM itemProducts (Caption: 'Products', Action: SHOW_PAGE Module.Product_Overview)
  }
}
```

### Snippet Syntax

```sql
CREATE [OR REPLACE] SNIPPET Module.SnippetName
[(
  Params: { $ParamName: Module.EntityType }
)]
[FOLDER 'path']
{
  -- Widget definitions (same as pages)
}
```

### NAVIGATIONLIST Syntax

The NAVIGATIONLIST widget creates a vertical menu with navigation items:

```sql
NAVIGATIONLIST widgetName {
  ITEM itemName (Caption: 'Caption', Action: SHOW_PAGE Module.PageName)
  ITEM itemName (Caption: 'Caption', Action: MICROFLOW Module.MicroflowName)
  ITEM itemName (Caption: 'Caption', Action: CLOSE_PAGE)
}
```

## Overview Page Template

Lists all objects of an entity type with a data grid and navigation menu in a sidebar layout.

**Layout Structure:**
```
┌─────────────────────────────────────────────┐
│ LAYOUTGRID                                  │
│ ┌────────┬──────────────────────────────────┤
│ │ COL 2  │ COL 10                           │
│ │ Menu   │ Heading + DataGrid               │
│ │Snippet │                                  │
│ └────────┴──────────────────────────────────┤
└─────────────────────────────────────────────┘
```

```sql
CREATE PAGE Module.Entity_Overview
(
  Title: 'Entity Overview',
  Layout: Atlas_Core.Atlas_Default,
  Folder: 'OverviewPages'
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN colNav (DesktopWidth: 2) {
        SNIPPETCALL navMenu (Snippet: Module.Entity_Menu)
      }
      COLUMN colContent (DesktopWidth: 10) {
        DYNAMICTEXT heading (Content: 'Entities', RenderMode: H2)
        DATAGRID EntityGrid (DataSource: DATABASE Module.Entity) {
          COLUMN colName (Attribute: Name, Caption: 'Name')
          COLUMN colDescription (Attribute: Description, Caption: 'Description')
        }
      }
    }
  }
}
```

### SNIPPETCALL Syntax

Include a snippet in a page using SNIPPETCALL:

```sql
-- Simple snippet call
SNIPPETCALL widgetName (Snippet: Module.SnippetName)

-- With parameters (for parameterized snippets):
SNIPPETCALL widgetName (Snippet: Module.SnippetName, Params: {Customer: $Customer})
```

### Overview Page Components

1. **Navigation Snippet**: `SNIPPETCALL` referencing `Module.NavigationMenu`
2. **Layout**: `Atlas_Core.Atlas_Default` - Full page with header/footer
3. **Heading**: `DYNAMICTEXT` with `RenderMode: H2`
4. **Data Grid**: `DATAGRID` with `DataSource: DATABASE` binding

### DATAGRID Syntax

```sql
DATAGRID GridName (
  DataSource: DATABASE FROM Module.Entity WHERE [IsActive = true] SORT BY Name ASC,
  Selection: Single|Multiple|None
) {
  COLUMN colName (Attribute: AttributeName, Caption: 'Label')
  COLUMN colCustom (Caption: 'Custom') {
    -- Nested widgets (ACTIONBUTTON, LINKBUTTON, DYNAMICTEXT)
  }
}
```

**Properties:**
- `DataSource: DATABASE FROM Module.Entity` - Entity data source (required)
- `WHERE [condition]` - Optional XPath filter (inline after entity in DataSource)
- `SORT BY attr ASC|DESC` - Optional sorting (inline after WHERE: `SORT BY Name ASC, Price DESC`)
- `Selection: Single|Multiple|None` - Optional selection mode

**Column Types:**
- `COLUMN colName (Attribute: Attribute, Caption: 'Label')` - Attribute column with binding
- `COLUMN colName (Caption: 'Label') { ... }` - Custom column with nested widgets

**Column Properties (non-default only in DESCRIBE output):**

| Property | Values | Default |
|----------|--------|---------|
| `Sortable` | `true`/`false` | `true` (with attribute) |
| `Resizable` | `true`/`false` | `true` |
| `Draggable` | `true`/`false` | `true` |
| `Hidable` | `yes`/`hidden`/`no` | `yes` |
| `ColumnWidth` | `autoFill`/`autoFit`/`manual` | `autoFill` |
| `Size` | integer (px) | `1` (when manual) |
| `Visible` | expression | `true` |
| `DynamicCellClass` | expression | (empty) |
| `Tooltip` | text | (empty) |

## NewEdit Page Template

Form for creating or editing a single entity. **Requires a page parameter** to receive the object.

```sql
CREATE PAGE Module.Entity_NewEdit
(
  Params: { $Entity: Module.Entity },
  Title: 'Edit Entity',
  Layout: Atlas_Core.PopupLayout,
  Folder: 'OverviewPages'
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: AutoFill) {
        DATAVIEW dataView1 (DataSource: $Entity) {
          -- Input fields for each attribute
          TEXTBOX txtName (Label: 'Name', Attribute: Name)
          TEXTBOX txtDescription (Label: 'Description', Attribute: Description)
          DATEPICKER dpDueDate (Label: 'Due Date', Attribute: DueDate)
          COMBOBOX cbStatus (Label: 'Status', Attribute: Status)

          FOOTER footer1 {
            ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Success)
            ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
          }
        }
      }
    }
  }
}
```

### Page Parameter Syntax

```sql
CREATE PAGE Module.PageName
(
  Params: { $ParamName: Module.EntityName },
  Title: '...',
  Layout: ...
)
```

- Parameter name conventionally matches the entity name (e.g., `$Store`, `$Customer`)
- The DataView's binding references this parameter (`DataSource: $ParamName`)
- When calling the page via SHOW_PAGE, pass an object of this entity type

### NewEdit Page Components

1. **Page Parameter**: `Params: { $Entity: Module.Entity }` - Receives the object to edit
2. **Layout**: `Atlas_Core.PopupLayout` - Popup/modal style
3. **DataView**: Container bound to page parameter (`DataSource: $Entity`)
4. **Input Widgets**: Match entity attributes with `Attribute:` property
5. **Footer**: Save and Cancel buttons

## Complete Example: Store Entity

### Step 1: Create the Navigation Snippet

First, create a navigation menu snippet that will be shared across all overview pages:

```sql
CREATE SNIPPET MdlTemplates.NavigationMenu
{
  LAYOUTGRID navGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 12) {
        ACTIONBUTTON btnStores (Caption: 'Stores', Action: SHOW_PAGE MdlTemplates.Store_Overview)
        ACTIONBUTTON btnCars (Caption: 'Cars', Action: SHOW_PAGE MdlTemplates.Car_Overview)
      }
    }
  }
}
```

### Step 2: Create the Entity

```sql
CREATE PERSISTENT ENTITY MdlTemplates.Store (
  Name: String(200) NOT NULL,
  Location: String(200)
);
```

### Step 3: Create the Overview Page

```sql
CREATE PAGE MdlTemplates.Store_Overview
(
  Title: 'Store Overview',
  Layout: Atlas_Core.Atlas_Default,
  Folder: 'OverviewPages'
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 12) {
        SNIPPETCALL navMenu (Snippet: MdlTemplates.NavigationMenu)
      }
    }
    ROW row2 {
      COLUMN col2 (DesktopWidth: 12) {
        DYNAMICTEXT heading (Content: 'Stores', RenderMode: H2)
      }
    }
    ROW row3 {
      COLUMN col3 (DesktopWidth: 12) {
        DATAGRID StoreGrid (DataSource: DATABASE MdlTemplates.Store) {
          COLUMN colName (Attribute: Name, Caption: 'Name')
          COLUMN colLocation (Attribute: Location, Caption: 'Location')
        }
      }
    }
  }
}
```

### Store NewEdit Page

```sql
CREATE PAGE MdlTemplates.Store_NewEdit
(
  Params: { $Store: MdlTemplates.Store },
  Title: 'Edit Store',
  Layout: Atlas_Core.PopupLayout,
  Folder: 'OverviewPages'
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: AutoFill) {
        DATAVIEW dataView1 (DataSource: $Store) {
          TEXTBOX txtName (Label: 'Name', Attribute: Name)
          TEXTBOX txtLocation (Label: 'Location', Attribute: Location)

          FOOTER footer1 {
            ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Success)
            ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
          }
        }
      }
    }
  }
}
```

## Complete Example: Car Entity

### Entity Definition

```sql
CREATE PERSISTENT ENTITY MdlTemplates.Car (
  Brand: String(200) NOT NULL,
  Model: String(200),
  Price: Decimal,
  PurchaseYear: Integer,
  PurchaseDate: DateTime,
  CarType: Enumeration(MdlTemplates.CarType)
);

CREATE ENUMERATION MdlTemplates.CarType (
  Sedan 'Sedan',
  SUV 'SUV',
  Truck 'Truck',
  Sports 'Sports Car'
);
```

### Car NewEdit Page

Shows various input widget types:

```sql
CREATE PAGE MdlTemplates.Car_NewEdit
(
  Params: { $Car: MdlTemplates.Car },
  Title: 'Edit Car',
  Layout: Atlas_Core.PopupLayout,
  Folder: 'OverviewPages'
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: AutoFill) {
        DATAVIEW dataView1 (DataSource: $Car) {
          TEXTBOX txtBrand (Label: 'Brand', Attribute: Brand)
          TEXTBOX txtModel (Label: 'Model', Attribute: Model)
          TEXTBOX txtPrice (Label: 'Price', Attribute: Price)
          TEXTBOX txtYear (Label: 'Purchase year', Attribute: PurchaseYear)
          DATEPICKER dpDate (Label: 'Purchase date', Attribute: PurchaseDate)
          RADIOBUTTONS rbType (Label: 'Car type', Attribute: CarType)

          FOOTER footer1 {
            ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Success)
            ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
          }
        }
      }
    }
  }
}
```

## Widget Selection Guide

Choose input widgets based on attribute type:

| Attribute Type | Widget | Example |
|----------------|--------|---------|
| String | `TEXTBOX` | Name, Description |
| String (long) | `TEXTAREA` | Comments, Notes |
| Integer, Long, Decimal | `TEXTBOX` | Price, Quantity |
| Boolean | `CHECKBOX` or `RADIOBUTTONS` | IsActive, IsPublished |
| DateTime | `DATEPICKER` | DueDate, OrderDate |
| Enumeration | `COMBOBOX` or `RADIOBUTTONS` | Status, Type |
| Association (reference) | `COMBOBOX` with DataSource | Category, Owner |

**Note:** `DROPDOWN` is deprecated. Use `COMBOBOX` for enumeration attributes.

**ComboBox modes:**
- Enum mode: `COMBOBOX cb (Label: 'Status', Attribute: Status)`
- Association mode: `COMBOBOX cb (Label: 'Customer', Attribute: Order_Customer, DataSource: DATABASE MyModule.Customer, CaptionAttribute: Name)`

**Reserved Attribute Names:** Do not use `CreatedDate`, `ChangedDate`, `Owner`, `ChangedBy` as attribute names - these are system attributes automatically added to all entities.

## Naming Conventions

| Item | Convention | Example |
|------|------------|---------|
| Navigation Snippet | `NavigationMenu` | `MdlTemplates.NavigationMenu` |
| Overview Page | `Entity_Overview` | `Customer_Overview` |
| NewEdit Page | `Entity_NewEdit` | `Customer_NewEdit` |
| Folder | `OverviewPages` | — |
| DataView | `dataView1` or `dv{Entity}` | `dvCustomer` |
| DataGrid | `dataGrid1` or `dg{Entity}` | `dgCustomer` |
| SnippetCall | `navMenu` or descriptive name | `navMenu`, `headerSnippet` |

## Button Styles

| Style | Use Case | Color |
|-------|----------|-------|
| `Success` | Save, Confirm | Green |
| `Default` | Cancel, Back | Gray |
| `Primary` | Primary action | Blue |
| `Danger` | Delete | Red |
| `Warning` | Caution actions | Yellow |

## Folder Organization

```
Module/
├── Snippets/
│   └── NavigationMenu
├── OverviewPages/
│   ├── Customer_Overview
│   ├── Customer_NewEdit
│   ├── Order_Overview
│   ├── Order_NewEdit
│   └── ...
├── Microflows/
└── Entities/
```

## Parameterized Snippets

Snippets can accept parameters to display context-specific data:

```sql
-- Create a snippet with a parameter
CREATE SNIPPET Module.CustomerDetails
(
  Params: { $Customer: Module.Customer }
)
{
  LAYOUTGRID detailsGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 12) {
        DYNAMICTEXT heading (Content: 'Customer Details', RenderMode: H3)
      }
    }
  }
}

-- Use the snippet with parameter passing
SNIPPETCALL customerDetails (Snippet: Module.CustomerDetails, Params: {Customer: $Customer})
```

## Entity Menu Snippets with NavigationList

For entity-specific action menus (Edit, Delete, etc.), use the `NAVIGATIONLIST` widget:

```sql
CREATE SNIPPET Module.Entity_Menu
(
  Params: { $EntityParameter: Module.Entity }
)
{
  NAVIGATIONLIST EntityMenuNav {
    ITEM itemEdit (Caption: 'Edit', Action: SHOW_PAGE Module.Entity_NewEdit(Entity: $EntityParameter))
    ITEM itemDelete (Caption: 'Delete', Action: DELETE)
    ITEM itemBack (Caption: 'Back', Action: CLOSE_PAGE)
  }
}
```

### NavigationList Syntax

```sql
NAVIGATIONLIST widgetName {
  ITEM itemName (Caption: 'Caption', Action: ACTION_TYPE)
}
```

**Supported Actions:**
- `Action: SAVE_CHANGES` - Save changes
- `Action: CANCEL_CHANGES` - Cancel changes
- `Action: CLOSE_PAGE` - Close current page
- `Action: DELETE` - Delete object
- `Action: MICROFLOW Module.MicroflowName` - Call microflow
- `Action: MICROFLOW Module.MicroflowName(Param: $value)` - Call microflow with parameters
- `Action: SHOW_PAGE Module.PageName` - Navigate to page
- `Action: SHOW_PAGE Module.PageName(Param: $value)` - Navigate with parameters

## Handling Circular Dependencies

When a navigation snippet references pages (via `SHOW_PAGE`) and those pages reference the snippet (via `SNIPPETCALL`), you have a circular dependency. Use the **placeholder pattern**:

### Creation Order

1. **Create placeholder snippet first** (before pages)
2. **Create all pages** (which reference the snippet via SNIPPETCALL)
3. **Replace snippet with full content** (which can now reference existing pages)

### Example Pattern

```sql
-- Step 1: Create placeholder snippet (pages can reference this)
CREATE SNIPPET Module.NavigationMenu
{
  LAYOUTGRID navGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 12) {
        DYNAMICTEXT loading (Content: 'Loading...')
      }
    }
  }
}
/

-- Step 2: Create all pages (they reference the snippet via SNIPPETCALL)
CREATE PAGE Module.Customer_NewEdit
(
  Params: { $Customer: Module.Customer },
  Title: 'Edit Customer',
  Layout: Atlas_Core.PopupLayout
)
{
  -- ... page content with SNIPPETCALL navMenu (Snippet: Module.NavigationMenu)
}
/

CREATE PAGE Module.Customer_Overview
(
  Title: 'Customer Overview',
  Layout: Atlas_Core.Atlas_Default
)
{
  -- ... page content with SNIPPETCALL navMenu (Snippet: Module.NavigationMenu)
}
/

-- Step 3: Replace snippet with full navigation (pages now exist)
CREATE OR REPLACE SNIPPET Module.NavigationMenu
{
  LAYOUTGRID navGrid {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 12) {
        ACTIONBUTTON btnCustomers (Caption: 'Customers', Action: SHOW_PAGE Module.Customer_Overview)
      }
    }
  }
}
/
```

### Key Points

- The placeholder snippet must exist before pages are created (for SNIPPETCALL to resolve)
- Use `CREATE OR REPLACE SNIPPET` to update the placeholder after pages exist
- Page references in the final snippet will resolve correctly because pages already exist

## Related Skills

- [Create Page](./create-page.md) - Basic page creation syntax
- [ALTER PAGE/SNIPPET](./alter-page.md) - Modify existing pages/snippets in-place (SET, INSERT, DROP, REPLACE)
- [Master-Detail Pages](./master-detail-pages.md) - Selection binding pattern

## Snippet Commands Reference

| Command | Description |
|---------|-------------|
| `SHOW SNIPPETS [IN module]` | List all snippets |
| `SHOW SNIPPET Module.Name` | Show snippet summary |
| `DESCRIBE SNIPPET Module.Name` | Show snippet MDL source |
| `CREATE SNIPPET Module.Name { ... }` | Create a new snippet |
| `CREATE OR REPLACE SNIPPET Module.Name { ... }` | Create or update snippet |
| `ALTER SNIPPET Module.Name { ... }` | Modify snippet widgets in-place |
| `DROP SNIPPET Module.Name` | Delete a snippet |
