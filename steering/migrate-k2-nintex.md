# K2/Nintex to Mendix Migration Skill

This skill provides comprehensive guidance for assessing and migrating K2 (now Nintex K2) applications to Mendix using MDL (Mendix Definition Language).

## When to Use This Skill

Use this skill when:
- Analyzing K2/Nintex applications for migration to Mendix
- Converting SmartObjects to Mendix domain models
- Mapping SmartForms Views to Mendix pages
- Translating K2 Workflows to Mendix microflows or workflows
- Planning a migration strategy for legacy K2 systems

## Understanding K2 Application Architecture

K2 applications are fundamentally different from Mendix in how they're stored and structured.

### K2 Storage Model

**Key Difference**: K2 applications are **server-side/database-stored**, not file-based like Mendix. All K2 project elements and data are saved into the K2 database, not as files. This is one of the key challenges for migration.

| Aspect | K2/Nintex | Mendix |
|--------|-----------|--------|
| Storage | Server database | `.mpr` file (SQLite) |
| Versioning | Server-managed | Git-based (MPR v2) |
| Export format | `.kspx` package | `.mpk` package |
| Project file | No single file | `.mpr` project file |

### K2 Artifact Types

A K2 application is composed of several artifact types:

#### SmartObjects

Middle layer between data providers (SQL, SAP, SharePoint) and data consumers (forms, workflows, reports). They abstract data from LOB systems.

| SmartObject Type | Description | Mendix Mapping |
|------------------|-------------|----------------|
| SmartBox | Stores data in K2's own database | Persistable Entity |
| SQL Connector | Connects to SQL Server tables | External Database Connector |
| SAP Connector | Connects to SAP systems | OData/REST Integration |
| SharePoint Connector | Connects to SharePoint lists | REST Client |
| Service Object | Exposes services/methods | Microflow/Java Action |

#### SmartForms

Browser-based forms composed of Views and Forms:

| SmartForms Element | Description | Mendix Mapping |
|--------------------|-------------|----------------|
| View | Reusable collection of controls + rules bound to SmartObjects | Snippet or Page Section |
| Form | Container for views, accessible via URL | Page |
| Control | UI element (text, date, dropdown, etc.) | Widget |
| Rule | Event-driven logic ("when button clicked, execute method") | Nanoflow or Microflow |

#### Workflows

Process definitions with steps, tasks, branching logic:

| Workflow Element | Description | Mendix Mapping |
|------------------|-------------|----------------|
| Workflow | Full process definition | Workflow or Microflow chain |
| Activity | Individual step in workflow | Microflow activity or User Task |
| Task | Human task requiring user action | User Task |
| Destination Rule | Routing logic for tasks | Decision (microflow) |
| Datafield | Workflow data variable | Parameter or Variable |

## Export and Package Options

There's no single "project file" like Mendix's `.mpr`. Your options for extracting K2 app definitions:

### 1. K2 Package (.kspx)

The K2 Package and Deployment tool packages K2 artifacts (SmartObjects, forms, views, workflows) into a single file with a `.kspx` extension. This is the primary mechanism for moving apps between environments.

**How to export:**
```
K2 Management Site → Solutions → Package → Export
```

### 2. Legacy Project Files

Older K2 Studio / K2 for Visual Studio projects used:

| File Type | Extension | Contents |
|-----------|-----------|----------|
| Project file | `.k2proj` | Project structure and references |
| Workflow definition | `.kprx` | Workflow definition |
| SmartObject definition | `.sodx` | SmartObject schema |

### 3. K2 APIs

SmartObject Runtime API and management APIs can be used programmatically to extract definitions:

```csharp
// Example: Programmatic SmartObject extraction
using SourceCode.SmartObjects.Client;

SmartObjectClientServer server = new SmartObjectClientServer();
server.CreateConnection();
SmartObject so = server.GetSmartObject("CustomerSO");
// Extract properties, methods, etc.
```

## Migration Strategy

For a K2 → Mendix migration, approach it in layers:

### Layer 1: Data Model (SmartObjects → Entities)

| SmartObject Type | Migration Approach |
|------------------|-------------------|
| SmartBox SmartObjects | Direct translation to Mendix persistable entities |
| SQL Connector SmartObjects | Options: (a) Import data to Mendix, (b) External Database Connector |
| Service SmartObjects | Microflows that call external services |

**Example MDL:**
```sql
-- SmartBox SmartObject "Customer" → Mendix Entity
CREATE PERSISTENT ENTITY CRM.Customer (
  CustomerCode: String(50),
  CustomerName: String(200),
  Email: String(200),
  Phone: String(50),
  IsActive: Boolean DEFAULT true,
  CreatedDate: DateTime
);

-- SmartBox SmartObject "Order" → Mendix Entity
CREATE PERSISTENT ENTITY CRM.Order (
  OrderNumber: String(50),
  OrderDate: DateTime,
  Status: CRM.OrderStatus,  -- Enumeration
  TotalAmount: Decimal
);

-- SmartObject relationship → Association
CREATE ASSOCIATION CRM.Order_Customer (
  CRM.Order [*] -> CRM.Customer [1]
);
```

### Layer 2: UI (SmartForms → Pages)

SmartForms Views map to Mendix pages/snippets. The rules system (event-driven, "when X happens, do Y") maps well to Mendix's nanoflow/microflow-on-events pattern.

| SmartForms Control | Mendix Widget |
|--------------------|---------------|
| Text Box | TEXTBOX |
| Text Area | TEXTAREA |
| Drop-down List | COMBOBOX |
| Date Picker | DATEPICKER |
| Check Box | CHECKBOX |
| Radio Button | RADIOBUTTONS |
| Data Label | DYNAMICTEXT |
| Button | ACTIONBUTTON |
| List View | LISTVIEW or DATAGRID |
| Subview | SNIPPETCALL |
| Tab Control | Tab container pattern |

#### SmartForms Rules to Mendix Events

| SmartForms Rule | Mendix Implementation |
|-----------------|----------------------|
| When Control is Clicked | Button action microflow/nanoflow |
| When View is Initialized | Page data source microflow |
| When Control Value Changes | OnChange nanoflow |
| When Data Loads | Data source microflow |
| Execute SmartObject Method | Microflow calling entity operations |
| Transfer Data | Variable assignment in microflow |
| Show/Hide Control | Conditional visibility |
| Enable/Disable Control | Editable expression |

**Example MDL (SmartForm View → Mendix Page):**
```sql
CREATE PAGE CRM.Customer_Edit
(
  Params: { $Customer: CRM.Customer },
  Title: 'Edit Customer',
  Layout: Atlas_Core.PopupLayout
)
{
  DATAVIEW dvCustomer (DataSource: $Customer) {
    -- Text Box controls
    TEXTBOX txtCode (Label: 'Customer Code', Attribute: CustomerCode)
    TEXTBOX txtName (Label: 'Customer Name', Attribute: CustomerName)
    TEXTBOX txtEmail (Label: 'Email', Attribute: Email)
    TEXTBOX txtPhone (Label: 'Phone', Attribute: Phone)

    -- Check Box control
    CHECKBOX chkActive (Label: 'Active', Attribute: IsActive)

    -- Button bar (SmartForms action buttons)
    FOOTER footer1 {
      ACTIONBUTTON btnSave (Caption: 'Save', Action: SAVE_CHANGES, ButtonStyle: Primary)
      ACTIONBUTTON btnCancel (Caption: 'Cancel', Action: CANCEL_CHANGES)
    }
  }
}
```

### Layer 3: Process (Workflows → Microflows/Workflows)

K2 Workflows translate to Mendix microflows or the Workflow module:

| K2 Workflow Element | Mendix Mapping |
|--------------------|----------------|
| Start | Microflow start / Workflow start |
| Task (human) | User Task activity |
| Reference (call SmartObject) | Microflow activities (Create, Change, Retrieve) |
| Decision | Decision (split/merge) |
| Send Email | Email activity |
| Generate Document | Generate document microflow |
| Web Service Call | REST/Web service call |
| Script | Java action or expressions |
| End | End event / Microflow return |
| Escalation | Scheduled event or timer |
| Destination Rule | Microflow logic for task assignment |

#### Task Allocation Mapping

| K2 Destination Rule | Mendix User Task |
|--------------------|------------------|
| Specific User | User by association |
| Role | XPath targeting user role |
| Manager | Microflow calculating manager |
| Queue | First user from filtered list |

**Example MDL (K2 Task → Mendix Microflow):**
```sql
-- K2 Task "Review Order" → Mendix microflow for task handling
CREATE MICROFLOW CRM.ACT_Order_SubmitForReview ($Order: CRM.Order)
BEGIN
  -- Update status (like K2 "Set Status")
  CHANGE $Order (Status = CRM.OrderStatus.PendingReview);
  COMMIT $Order WITH EVENTS;

  -- Show page for review (like K2 "Task" with form)
  SHOW PAGE CRM.Order_Review ($Order = $Order);
END;

-- K2 Decision "Order > $5000?" → Mendix microflow with decision
CREATE MICROFLOW CRM.ACT_Order_ProcessApproval ($Order: CRM.Order)
RETURNS Boolean AS $Approved
BEGIN
  DECLARE $Approved Boolean = false;

  IF $Order/TotalAmount > 5000 THEN
    -- Route to manager (K2 Destination Rule equivalent)
    CALL MICROFLOW CRM.ACT_Order_SubmitForManagerReview ($Order = $Order);
  ELSE
    -- Auto-approve (K2 "Go To" equivalent)
    CHANGE $Order (Status = CRM.OrderStatus.Approved);
    COMMIT $Order WITH EVENTS;
    SET $Approved = true;
  END IF;

  RETURN $Approved;
END;
```

## Assessment Workflow

When assessing a K2 application for migration:

### Step 1: Inventory SmartObjects

Export SmartObject definitions and categorize:

```markdown
| SmartObject Name | Type | Data Source | Entity Count | Mendix Mapping |
|------------------|------|-------------|--------------|----------------|
| CustomerSO | SmartBox | K2 DB | Single | CRM.Customer entity |
| OrderSO | SmartBox | K2 DB | Single | CRM.Order entity |
| EmployeeSO | SQL | HR Database | Single | Integration or import |
| SAPOrderSO | SAP | SAP ERP | Multiple | OData service |
```

### Step 2: Inventory SmartForms

Document all forms and views:

```markdown
| Form Name | Views Used | SmartObjects | Mendix Mapping |
|-----------|------------|--------------|----------------|
| Customer Entry | CustomerView, AddressView | CustomerSO, AddressSO | Customer_Edit page |
| Order Dashboard | OrderListView, FilterView | OrderSO | Order_Overview page |
| Order Entry | OrderHeaderView, LineItemsView | OrderSO, OrderLineSO | Order_Edit page |
```

### Step 3: Inventory Workflows

Document all workflows and their complexity:

```markdown
| Workflow Name | Tasks | Activities | Complexity | Mendix Mapping |
|---------------|-------|------------|------------|----------------|
| Order Approval | 3 | 12 | Medium | Microflow chain |
| New Employee Onboarding | 8 | 25 | High | Workflow module |
| Leave Request | 2 | 6 | Low | Microflows only |
```

### Step 4: Map Rules and Logic

Extract business rules from SmartForms rules and workflow logic:

```markdown
| Rule ID | Location | Description | Mendix Implementation |
|---------|----------|-------------|----------------------|
| R-001 | CustomerView | Email format validation | Validation microflow |
| R-002 | OrderWorkflow | Orders > $5000 need manager approval | Decision in microflow |
| R-003 | LineItemsView | Auto-calculate line total | On-change nanoflow |
```

## Migration Execution Order

Execute migration in this order to manage dependencies:

### Phase 1: Domain Model
```sql
-- 1. Enumerations first (no dependencies)
CREATE ENUMERATION CRM.OrderStatus AS (
  Pending: 'Pending',
  Approved: 'Approved',
  Rejected: 'Rejected',
  Completed: 'Completed'
);

-- 2. Entities (may reference enumerations)
CREATE PERSISTENT ENTITY CRM.Customer (...);
CREATE PERSISTENT ENTITY CRM.Order (...);

-- 3. Associations (reference entities)
CREATE ASSOCIATION CRM.Order_Customer (...);
```

### Phase 2: Business Logic (Microflows)
```sql
-- Core CRUD microflows
CREATE MICROFLOW CRM.ACT_Customer_Save ($Customer: CRM.Customer)
BEGIN
  -- Validation (from SmartForms rules)
  IF $Customer/Email = empty THEN
    VALIDATION FEEDBACK $Customer/Email MESSAGE 'Email is required';
    RETURN false;
  END IF;

  COMMIT $Customer WITH EVENTS;
  RETURN true;
END;

-- Workflow logic
CREATE MICROFLOW CRM.ACT_Order_Submit ($Order: CRM.Order)
BEGIN
  -- Workflow start logic
  ...
END;
```

### Phase 3: Pages
```sql
-- Overview pages
CREATE PAGE CRM.Customer_Overview (...);

-- Edit pages (can reference microflows from Phase 2)
CREATE PAGE CRM.Customer_Edit (...);
```

### Phase 4: Security
```sql
-- Module roles matching K2 roles
CREATE MODULE ROLE CRM.Manager DESCRIPTION 'Can approve orders and manage customers';
CREATE MODULE ROLE CRM.User DESCRIPTION 'Can create and edit own records';

-- Access rules
GRANT CRM.Manager ON CRM.Order (CREATE, DELETE, READ *, WRITE *);
GRANT CRM.User ON CRM.Order (CREATE, READ *, WRITE *) WHERE [Owner = '[%CurrentUser%]'];
```

## Common Challenges and Solutions

### Challenge 1: Server-Side Storage

**Problem**: K2 stores everything in a database, no single project file.

**Solution**: Use K2 Package (.kspx) export or K2 APIs to extract definitions. Work with K2 administrators to get comprehensive exports.

### Challenge 2: SmartObject Connectors

**Problem**: SmartObjects may connect to external systems (SQL, SAP, SharePoint).

**Solutions**:
| Connector Type | Mendix Options |
|----------------|----------------|
| SQL Direct | External Database Connector or data migration |
| SAP | SAP BAPI Connector, OData, or REST |
| SharePoint | REST integration via Microsoft Graph API |
| Web Service | REST/SOAP consumption |

### Challenge 3: Complex Rules

**Problem**: SmartForms rules are event-driven and can be deeply nested.

**Solution**: Map each rule to appropriate Mendix mechanism:
```
Event-based UI logic → Nanoflows
Validation → Validation microflows + VALIDATION FEEDBACK
Data manipulation → Microflows
Complex calculations → Microflow expressions
```

### Challenge 4: Workflow Participants

**Problem**: K2 uses destination rules for task assignment that may be complex.

**Solution**: Implement participant logic in microflows:
```sql
-- K2 Destination Rule "Route to Manager" → Mendix microflow
CREATE MICROFLOW CRM.SUB_GetManager ($Employee: HR.Employee)
RETURNS HR.Employee AS $Manager
BEGIN
  DECLARE $Manager HR.Employee;
  RETRIEVE $Manager FROM HR.Employee
    WHERE [HR.Employee_Reports = $Employee];
  RETURN $Manager;
END;
```

## Pre-Migration Checklist

Before starting migration:
- [ ] Obtain K2 Package (.kspx) export for all artifacts
- [ ] Get SmartObject documentation or extract via APIs
- [ ] Document all SmartForms views and their rules
- [ ] Map workflow steps and decision logic
- [ ] Identify external system integrations (SQL, SAP, SharePoint)
- [ ] Understand K2 security/role model
- [ ] Plan for data migration (SmartBox data → Mendix entities)

During migration:
- [ ] Create entities in dependency order
- [ ] Create enumerations before entities that use them
- [ ] Create microflows before pages that reference them
- [ ] Test validation rules thoroughly
- [ ] Verify workflow logic paths

After migration:
- [ ] Run `mxcli check script.mdl -p app.mpr --references`
- [ ] Open in Mendix Studio Pro to verify
- [ ] Test all workflow scenarios
- [ ] Validate data migration completeness

## Investigation Tips

If you have access to K2:

1. **K2 Management Site**: Export packages and view SmartObject schemas
2. **K2 Designer**: View SmartForms rules in detail
3. **K2 Workspace**: Examine workflow definitions
4. **SQL Server**: Query K2 database for SmartBox data and metadata
5. **K2 API**: Programmatically extract definitions for automation

If you have a `.kspx` file, it can potentially be parsed (it's a ZIP-based format) to extract XML definitions of the artifacts inside.

## Related Skills

- [/assess-migration](./assess-migration.md) - General migration assessment framework
- [/generate-domain-model](./generate-domain-model.md) - Creating entities and associations in MDL
- [/write-microflows](./write-microflows.md) - Implementing business logic in MDL
- [/create-page](./create-page.md) - Building pages in MDL
- [/manage-security](./manage-security.md) - Setting up roles and access rules
- [/organize-project](./organize-project.md) - Folder structure for the migrated project
