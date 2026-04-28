# Master-Detail Pages

## Overview

Master-Detail is a common UI pattern showing:
- **Master list** (left): Selectable list of items (Gallery widget)
- **Detail form** (right): Form showing selected item details (DataView with SELECTION source)

## MDL Syntax

### Basic Structure

```sql
CREATE PAGE Module.Entity_MasterDetail
(
  Title: 'Entity Master-Detail',
  Layout: Atlas_Core.Atlas_Default
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      -- Master list (4 columns)
      COLUMN colMaster (DesktopWidth: 4) {
        GALLERY entityList (DataSource: DATABASE Module.Entity, Selection: Single) {
          TEMPLATE template1 {
            DYNAMICTEXT name (Content: '{1}', ContentParams: [{1} = Name], RenderMode: H4)
          }
        }
      }

      -- Detail form (8 columns)
      COLUMN colDetail (DesktopWidth: 8) {
        DATAVIEW entityDetail (DataSource: SELECTION entityList) {
          TEXTBOX txtName (Label: 'Name', Attribute: Name)

          FOOTER footer1 {
            ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Success)
          }
        }
      }
    }
  }
}
```

### Key Components

#### 1. GALLERY Widget (Master List)

```sql
GALLERY widgetName (
  DataSource: DATABASE FROM Module.Entity SORT BY Name ASC,
  Selection: Single|Multiple|None
) {
  TEMPLATE template1 {
    -- Widgets for each item
    DYNAMICTEXT name (Content: '{1}', ContentParams: [{1} = AttrName], RenderMode: H4)
  }
}
```

**Properties:**
- `DataSource: DATABASE FROM Entity SORT BY attr ASC|DESC` - Entity data source with optional sorting
- `Selection: Single` - Selection mode (Single for master-detail)
- Template content inside TEMPLATE widget (requires name)

#### 2. DataView with SELECTION Source

```sql
DATAVIEW widgetName (DataSource: SELECTION sourceWidgetName) {
  -- Form widgets
}
```

The `SELECTION` source creates a binding to another widget's selection. When the user selects an item in the Gallery, the DataView displays that item.

#### 3. LISTVIEW Widget (Nested Data)

```sql
LISTVIEW widgetName (DataSource: DATABASE Module.Entity, PageSize: 10) {
  TEMPLATE template1 {
    -- Widgets for each associated item
  }
}
```

Used inside the detail form to show related/associated data.

**Nested list by association:** Use `DataSource: $currentObject/Module.Assoc` (or the explicit `DataSource: ASSOCIATION Path` form) inside a parent DATAVIEW. Both forms produce the same BSON (ByAssociation data source). Example: `DATAGRID lines (DataSource: $currentObject/Order_OrderLine)` inside a `DATAVIEW dv (DataSource: DATABASE Order)`.

## Complete Example

```sql
CREATE PAGE CRM.Customer_MasterDetail
(
  Title: 'Customer Management',
  Layout: Atlas_Core.Atlas_Default
)
{
  LAYOUTGRID mainGrid {
    ROW row1 {
      COLUMN colMaster (DesktopWidth: 4) {
        DYNAMICTEXT heading (Content: 'Customers', RenderMode: H3)
        GALLERY customerList (DataSource: DATABASE FROM CRM.Customer SORT BY Name ASC, Selection: Single) {
          TEMPLATE template1 {
            DYNAMICTEXT name (Content: '{1}', ContentParams: [{1} = Name], RenderMode: H4)
            DYNAMICTEXT email (Content: '{1}', ContentParams: [{1} = Email])
          }
        }
      }

      COLUMN colDetail (DesktopWidth: 8) {
        DATAVIEW customerDetail (DataSource: SELECTION customerList) {
          DYNAMICTEXT detailHeading (Content: 'Customer Details', RenderMode: H3)
          TEXTBOX txtName (Label: 'Name', Attribute: Name)
          TEXTBOX txtEmail (Label: 'Email', Attribute: Email)
          TEXTBOX txtPhone (Label: 'Phone', Attribute: Phone)

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

## Key Patterns

### Selection Binding

The core of master-detail is the selection binding:
1. Gallery has `Selection: Single` - enables single item selection
2. DataView uses `DataSource: SELECTION galleryName` - listens to Gallery selection
3. When user clicks an item in Gallery, DataView automatically updates

### Widget Names

The selection binding uses widget names to connect:
- Gallery widget name: `customerList`
- DataView references: `DataSource: SELECTION customerList`

### Template Content with ContentParams

Inside Gallery templates, use `ContentParams` to reference current item attributes:
```sql
TEMPLATE template1 {
  DYNAMICTEXT name (Content: '{1}', ContentParams: [{1} = Name], RenderMode: H4)
  DYNAMICTEXT email (Content: '{1}', ContentParams: [{1} = Email])
}
```

## Syntax Summary

| Element | Syntax |
|---------|-----------|
| Page properties | `(Title: 'Title', Layout: Module.Layout)` |
| Widget name | Required after type: `GALLERY myGallery (...)` |
| Database source | `DataSource: DATABASE FROM Module.Entity` |
| Selection binding | `DataSource: SELECTION widgetName` |
| Sort by | `DataSource: DATABASE FROM Entity SORT BY Name ASC` |
| Where filter | `DataSource: DATABASE FROM Entity WHERE [IsActive = true]` |
| Selection mode | `Selection: Single` |
| Attribute binding | `Attribute: AttributeName` |
| Action binding | `Action: SAVE_CHANGES` |
| Button style | `ButtonStyle: Success` |
| Text content | `Content: 'text'` with `ContentParams: [{1} = Attr]` |
| Render mode | `RenderMode: H4` |
| Template content | `TEMPLATE template1 { ... }` |

## Related Skills

- [Overview Pages](./overview-pages.md) - CRUD page patterns
- [Create Page](./create-page.md) - Basic page syntax
- [ALTER PAGE/SNIPPET](./alter-page.md) - Modify existing pages in-place (SET, INSERT, DROP, REPLACE)

## Implementation Notes

- Gallery is a pluggable widget (similar to DataGrid2)
- Selection binding uses `ListenTargetSource` in the Model SDK
- ListView is a built-in Mendix widget
- All widget properties use explicit `(Key: value)` syntax
