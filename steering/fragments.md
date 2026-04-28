# Mendix Fragments Skill

## When to Use This Skill

Use this skill when:
- Defining reusable widget groups with `DEFINE FRAGMENT`
- Inserting fragments into pages or snippets with `USE FRAGMENT`
- Listing or inspecting fragments with `SHOW FRAGMENTS` / `DESCRIBE FRAGMENT`
- Building multiple pages that share common widget patterns (footers, form fields, buttons)
- Avoiding copy-paste of repeated widget structures across pages

## What Are Fragments?

Fragments are **script-scoped, transient** widget groups:
- Defined once, reused in multiple pages/snippets within the same script
- **Not persisted** in the MPR file — they exist only during script execution
- Widgets are deep-cloned on expansion (each USE gets independent copies)
- Optional prefix support to avoid name conflicts when using the same fragment multiple times

## Syntax Reference

### DEFINE FRAGMENT

```mdl
DEFINE FRAGMENT SaveCancelFooter AS {
  FOOTER footer1 {
    ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Primary)
    ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
  }
};
```

Multiple top-level widgets:

```mdl
DEFINE FRAGMENT CustomerFields AS {
  TEXTBOX txtName (Label: 'Name', Attribute: Name)
  TEXTBOX txtEmail (Label: 'Email', Attribute: Email)
  TEXTBOX txtPhone (Label: 'Phone', Attribute: Phone)
};
```

### USE FRAGMENT

Inside a page or snippet body:

```mdl
CREATE PAGE Module.CustomerEdit
(
  Params: { $Customer: Module.Customer },
  Title: 'Edit Customer',
  Layout: Atlas_Core.PopupLayout
)
{
  DATAVIEW dvCustomer (DataSource: $Customer) {
    USE FRAGMENT CustomerFields
    USE FRAGMENT SaveCancelFooter
  }
};
```

With prefix (avoids name conflicts):

```mdl
USE FRAGMENT SaveCancelFooter AS order_
-- Creates: order_footer1, order_btnSave, order_btnCancel
```

### SHOW FRAGMENTS

```mdl
SHOW FRAGMENTS;
-- Lists all defined fragments with widget counts
```

### DESCRIBE FRAGMENT

```mdl
DESCRIBE FRAGMENT SaveCancelFooter;
-- Outputs the full MDL definition
```

## Common Patterns

### Pattern 1: Standard CRUD Footer

```mdl
DEFINE FRAGMENT CrudFooter AS {
  FOOTER footer1 {
    ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Primary)
    ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
  }
};

-- Use in every edit page
CREATE PAGE Module.Customer_Edit (...) {
  DATAVIEW dv (DataSource: $Customer) {
    TEXTBOX txtName (Label: 'Name', Attribute: Name)
    USE FRAGMENT CrudFooter
  }
};

CREATE PAGE Module.Order_Edit (...) {
  DATAVIEW dv (DataSource: $Order) {
    TEXTBOX txtNumber (Label: 'Order #', Attribute: Number)
    USE FRAGMENT CrudFooter
  }
};
```

### Pattern 2: Form Field Groups

```mdl
DEFINE FRAGMENT AddressFields AS {
  TEXTBOX txtStreet (Label: 'Street', Attribute: Street)
  TEXTBOX txtCity (Label: 'City', Attribute: City)
  TEXTBOX txtZip (Label: 'Zip Code', Attribute: ZipCode)
  TEXTBOX txtCountry (Label: 'Country', Attribute: Country)
};

-- Reuse in customer and supplier pages
CREATE PAGE Module.Customer_Edit (...) {
  DATAVIEW dv (DataSource: $Customer) {
    TEXTBOX txtName (Label: 'Name', Attribute: Name)
    USE FRAGMENT AddressFields
    USE FRAGMENT CrudFooter
  }
};
```

### Pattern 3: Same Fragment with Prefix

```mdl
DEFINE FRAGMENT ActionButtons AS {
  ACTIONBUTTON btnApprove (Caption: 'Approve', Action: SAVE_CHANGES, ButtonStyle: Success)
  ACTIONBUTTON btnReject (Caption: 'Reject', Action: CANCEL_CHANGES, ButtonStyle: Danger)
};

CREATE PAGE Module.DualPanel (...) {
  LAYOUTGRID lg {
    ROW row1 {
      COLUMN col1 (DesktopWidth: 6) {
        USE FRAGMENT ActionButtons AS left_
      }
      COLUMN col2 (DesktopWidth: 6) {
        USE FRAGMENT ActionButtons AS right_
      }
    }
  }
};
```

## Common Mistakes

### Duplicate Fragment Names

```mdl
-- WRONG: Defining the same fragment name twice causes an error
DEFINE FRAGMENT Footer AS { ... };
DEFINE FRAGMENT Footer AS { ... };  -- Error: fragment "Footer" already defined
```

### Missing Fragment

```mdl
-- WRONG: Using a fragment that hasn't been defined
CREATE PAGE Module.MyPage (...) {
  USE FRAGMENT NonExistent   -- Error: fragment "NonExistent" not found
};
```

### Name Conflicts Without Prefix

```mdl
-- WRONG: Using same fragment twice without prefix creates duplicate widget names
USE FRAGMENT Footer
USE FRAGMENT Footer   -- Widget name "footer1" already exists!

-- CORRECT: Use prefix for uniqueness
USE FRAGMENT Footer AS first_
USE FRAGMENT Footer AS second_
```

### Fragment Order

```mdl
-- WRONG: Using a fragment before defining it
CREATE PAGE Module.MyPage (...) {
  USE FRAGMENT Footer   -- Error: fragment "Footer" not found
};
DEFINE FRAGMENT Footer AS { ... };

-- CORRECT: Define before use
DEFINE FRAGMENT Footer AS { ... };
CREATE PAGE Module.MyPage (...) {
  USE FRAGMENT Footer   -- OK
};
```

## Validation Checklist

- [ ] All `DEFINE FRAGMENT` statements appear before their `USE FRAGMENT` references
- [ ] No duplicate fragment names in the script
- [ ] Prefix used when the same fragment appears multiple times on one page
- [ ] Fragment widget names don't conflict with other widgets on the page
- [ ] All widgets inside fragments use valid syntax (same as page bodies)

## Related Documentation

- `mxcli syntax fragment` — CLI help topic
- `create-page.md` — Page/widget syntax reference
- `overview-pages.md` — CRUD page patterns
- Proposal: `docs/11-proposals/proposal_page_composition.md`
