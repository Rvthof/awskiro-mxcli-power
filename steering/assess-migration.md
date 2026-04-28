# Migration Assessment: Investigating Non-Mendix Projects

This skill guides the investigation of existing non-Mendix applications to produce a structured migration assessment for Mendix.

## When to Use This Skill

Use this skill when:
- A user asks to analyze an existing project for migration to Mendix
- Investigating a codebase in any technology (Java, .NET, Python, Node.js, PHP, etc.)
- Producing a migration inventory or assessment report
- Planning the scope and phases of a migration project

## Investigation Process

### Step 1: Identify the Technology Stack

Determine the application's technology stack by examining:
- Build files (`pom.xml`, `*.csproj`, `package.json`, `requirements.txt`, `Gemfile`, `composer.json`)
- Configuration files (`application.properties`, `appsettings.json`, `web.config`, `.env`)
- Framework indicators (Spring Boot, ASP.NET, Django, Express, Laravel, Rails)
- Database configuration (connection strings, ORM config, migration files)
- Frontend framework (`angular.json`, `next.config.js`, Vue/React/Svelte indicators)

### Step 2: Map the Data Model

Investigate and document all entities, their attributes, and relationships.

**Where to look:**

| Technology | Data Model Location |
|------------|-------------------|
| Java/Spring | `@Entity` classes, JPA annotations, Hibernate mappings |
| .NET/EF | `DbContext`, entity classes, EF migrations |
| Django | `models.py` files |
| Rails | `app/models/`, `db/schema.rb` |
| Node.js | Sequelize/TypeORM/Prisma models, Mongoose schemas |
| PHP/Laravel | Eloquent models, migrations |
| Database-first | SQL schema, stored procedures, views |

**Output format:**

```markdown
### Data Model

#### Entities

| Entity | Attributes | Type | Constraints | Mendix Mapping |
|--------|-----------|------|-------------|----------------|
| Customer | id | Long (auto) | PK | (auto-generated) |
| | name | String(200) | NOT NULL | String(200) |
| | email | String(200) | UNIQUE | String(200) |
| | creditLimit | Decimal(10,2) | | Decimal |
| | isActive | Boolean | DEFAULT true | Boolean |
| | createdAt | DateTime | | DateTime |
| Order | id | Long (auto) | PK | (auto-generated) |
| | orderNumber | String(50) | UNIQUE, NOT NULL | String(50) |
| | status | Enum | | Enumeration |
| | totalAmount | Decimal | | Decimal |

#### Associations

| From | To | Type | Mendix Mapping |
|------|----|------|----------------|
| Order | Customer | Many-to-One | Reference (Order → Customer) |
| Order | OrderLine | One-to-Many | Reference (OrderLine → Order) |
| User | Role | Many-to-Many | ReferenceSet |

#### Enumerations

| Name | Values | Used By |
|------|--------|---------|
| OrderStatus | PENDING, PROCESSING, COMPLETED, CANCELLED | Order.status |
| UserRole | ADMIN, MANAGER, USER | User.role |
```

### Step 3: Catalog Business Logic and Rules

This is the most critical part. Identify and explicitly document all business logic, validation rules, calculations, and workflows.

**Where to look:**

| Technology | Logic Location |
|------------|---------------|
| Java/Spring | `@Service` classes, `@Component`, `@Transactional` methods |
| .NET | Service classes, domain logic, middleware |
| Django | Views, forms, signals, managers |
| Rails | Models (callbacks, validations), services, concerns |
| Node.js | Route handlers, middleware, service modules |
| Database | Stored procedures, triggers, functions, constraints |

**Categorize each piece of logic:**

```markdown
### Business Logic

#### Validation Rules

| Rule | Location | Description | Mendix Mapping |
|------|----------|-------------|----------------|
| VR-001 | CustomerService.validate() | Customer name is required, max 200 chars | Validation microflow |
| VR-002 | OrderService.validate() | Order date cannot be in the past | Validation microflow |
| VR-003 | DB trigger: trg_check_credit | Credit limit cannot exceed $1M for non-premium | Validation microflow + before commit |
| VR-004 | OrderLine model | Quantity must be > 0 | Attribute validation |

#### Business Rules

| Rule | Location | Description | Mendix Mapping |
|------|----------|-------------|----------------|
| BR-001 | OrderService.calculateTotal() | Sum of line items * quantity, apply tax | Microflow |
| BR-002 | DiscountService.applyDiscount() | 10% for orders > $1000, 15% for premium | Microflow |
| BR-003 | CustomerService.updateStatus() | Deactivate customer after 12 months inactive | Scheduled microflow |
| BR-004 | DB proc: sp_close_month | Monthly closing with balance calculations | Microflow + scheduled event |

#### Workflows / Multi-Step Processes

| Workflow | Steps | Description | Mendix Mapping |
|----------|-------|-------------|----------------|
| Order Approval | Submit → Review → Approve/Reject → Notify | Orders > $5000 need manager approval | Workflow or microflow chain |
| User Onboarding | Register → Verify Email → Complete Profile → Activate | New user registration flow | Microflow chain + pages |

#### Calculated Fields / Derived Data

| Field | Calculation | Location | Mendix Mapping |
|-------|-------------|----------|----------------|
| Order.totalAmount | SUM(lines.price * lines.qty) | OrderService | Calculated attribute or microflow |
| Customer.orderCount | COUNT(orders) | SQL view | Microflow or calculated attribute |
| Invoice.dueDate | invoiceDate + paymentTerms | InvoiceService | Microflow |
```

### Step 4: Inventory Pages and UI

Document all screens, their purpose, and the data they display or edit.

**Where to look:**

| Technology | UI Location |
|------------|-------------|
| Java/Spring MVC | JSP/Thymeleaf templates, controllers |
| .NET MVC/Razor | Views, Razor pages, controllers |
| React/Angular/Vue | Component files, route definitions |
| Django | Templates, URL conf |
| Rails | Views, routes |
| Mobile | Activities/Fragments (Android), ViewControllers (iOS) |

**Output format:**

```markdown
### Pages / Screens

| Page | Type | Data | Key Actions | Mendix Mapping |
|------|------|------|-------------|----------------|
| Customer List | Overview | Customer (filtered, paged) | Search, New, Edit, Delete | Overview page + DataGrid |
| Customer Edit | Form | Customer + Addresses | Save, Cancel, Validate | Edit page + DataView |
| Order Dashboard | Dashboard | Orders (grouped by status) | Filter, Drill-down | Page + multiple DataGrids |
| Order Entry | Multi-step form | Order + OrderLines | Add line, Calculate, Submit | Page + ListView + microflows |
| Reports | Read-only | Aggregated data | Export, Print | Page + charts or custom widgets |
| Login | Authentication | Credentials | Login, Forgot Password | Login page (built-in) |
```

### Step 5: Map Integrations

Document all external system connections, APIs consumed, and APIs exposed.

**Where to look:**
- REST/SOAP client configurations
- HTTP client usage, API base URLs
- Message queue consumers/producers (Kafka, RabbitMQ, SQS)
- File import/export (CSV, Excel, XML, JSON)
- Email sending configuration
- External authentication (OAuth, SAML, LDAP)
- Third-party SDK usage (payment, notification, storage)

**Output format:**

```markdown
### Integrations

#### APIs Consumed (Outbound)

| Integration | Protocol | Endpoint | Auth | Mendix Mapping |
|-------------|----------|----------|------|----------------|
| Payment Gateway | REST | api.stripe.com | API Key | REST client (consumed) |
| Email Service | REST | api.sendgrid.com | API Key | Email module or REST |
| ERP Sync | SOAP | erp.company.com/ws | Certificate | Web service call |
| File Storage | SDK | AWS S3 | IAM | FileDocument + custom Java |

#### APIs Exposed (Inbound)

| Endpoint | Method | Purpose | Mendix Mapping |
|----------|--------|---------|----------------|
| /api/customers | GET, POST | Customer CRUD | Published REST service |
| /api/orders/{id} | GET, PUT | Order management | Published REST service |
| /webhooks/payment | POST | Payment notifications | Published REST service |

#### Data Feeds

| Feed | Format | Direction | Frequency | Mendix Mapping |
|------|--------|-----------|-----------|----------------|
| Customer export | CSV | Outbound | Daily | Scheduled microflow + export mapping |
| Product catalog | XML | Inbound | Hourly | Scheduled microflow + import mapping |
| Financial data | OData | Both | Real-time | External entities (OData) |

#### Message Queues / Events

| Queue/Topic | Direction | Purpose | Mendix Mapping |
|-------------|-----------|---------|----------------|
| order-events | Publish | Order status changes | Business events |
| inventory-updates | Subscribe | Stock level changes | Business events |
```

### Step 6: Document Security Model

Investigate how authentication, authorization, user roles, and data access control are implemented.

**Where to look:**

| Technology | Security Location |
|------------|-------------------|
| Java/Spring | Spring Security config, `@PreAuthorize`, `@Secured`, `@RolesAllowed` |
| .NET | `[Authorize]`, Identity config, policies, claims |
| Django | `@login_required`, permissions, groups |
| Rails | Devise, CanCanCan/Pundit policies |
| Node.js | Passport.js, JWT middleware, RBAC libraries |
| Database | GRANT statements, row-level security |

**Output format:**

```markdown
### Security

#### Authentication

| Method | Details | Mendix Mapping |
|--------|---------|----------------|
| Username/Password | Local DB with bcrypt | Built-in authentication |
| OAuth 2.0 | Google, Microsoft SSO | OIDC SSO module |
| SAML | Corporate IdP | SAML module |
| API Keys | For service accounts | Custom implementation |

#### User Roles

| Role | Description | Rough Privileges | Mendix Mapping |
|------|-------------|------------------|----------------|
| Admin | Full system access | All CRUD, user management, settings | Administrator role |
| Manager | Department-level access | Approve orders, view reports, manage team | Custom module role |
| User | Standard operations | Create/edit own records, view assigned | Custom module role |
| Viewer | Read-only access | View only, no modifications | Custom module role |
| API Service | Machine-to-machine | Specific API endpoints only | Custom module role |

#### Data Access Rules

| Entity | Role | Create | Read | Write | Delete | Constraint |
|--------|------|--------|------|-------|--------|------------|
| Customer | Admin | Yes | All | All | Yes | None |
| Customer | Manager | Yes | Department | Department | No | Department = User.Department |
| Customer | User | No | Own | Own | No | CreatedBy = CurrentUser |
| Order | Manager | Yes | Department | Department | No | Status != 'Closed' |

#### Row-Level Security / Data Filtering

| Rule | Description | Mendix Mapping |
|------|-------------|----------------|
| Department isolation | Users only see their department's data | XPath constraint on entity access |
| Record ownership | Users only edit records they created | XPath: `[CreatedBy = '[%CurrentUser%]']` |
| Status-based locking | Closed records are read-only | XPath on write: `[Status != 'Closed']` |
```

## Assessment Report Template

Combine all findings into a structured report:

```markdown
# Migration Assessment: [Application Name]

## Executive Summary
- **Application**: [Name and brief description]
- **Technology stack**: [Languages, frameworks, databases]
- **Size**: [Entities, services/controllers, pages, integrations]
- **Complexity**: [Low / Medium / High]
- **Recommended approach**: [Big bang / Phased / Strangler fig]

## Inventory Summary

| Category | Count | Complexity | Notes |
|----------|-------|------------|-------|
| Entities | X | | |
| Associations | X | | |
| Enumerations | X | | |
| Business rules | X | | List the critical ones |
| Validation rules | X | | |
| Pages/Screens | X | | |
| Integrations | X | | List external systems |
| User roles | X | | |
| Scheduled jobs | X | | |

## Data Model
[From Step 2]

## Business Logic and Rules
[From Step 3 — this is the most important section]

## Pages and UI
[From Step 4]

## Integrations
[From Step 5]

## Security
[From Step 6]

## Migration Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Complex stored procedures | Logic may not map 1:1 to microflows | Review and simplify, consider Java actions |
| Custom UI components | No direct Mendix equivalent | Evaluate pluggable widgets or custom widgets |
| Real-time integrations | Mendix has different async patterns | Consider business events or polling |
| Row-level security complexity | Mendix XPath constraints have limits | Simplify access rules where possible |

## Recommended Migration Phases

1. **Domain model** — Entities, associations, enumerations
2. **Core business logic** — Validation rules, business rules, calculations
3. **Pages** — Overview pages, edit forms, dashboards
4. **Integrations** — REST clients, file handling, external systems
5. **Security** — Roles, access rules, authentication
6. **Testing & cutover** — Data migration, parallel running, go-live
```

## Tips

- **Be thorough with business logic**: This is where migrations fail. A missing validation rule or calculation creates bugs that are hard to trace back to the source.
- **Check the database**: Stored procedures, triggers, views, and constraints often contain business logic that isn't visible in application code.
- **Look for implicit rules**: Framework conventions (e.g., Rails validations, Spring annotations) encode rules that are easy to miss.
- **Document what you DON'T migrate**: Some features may not need to be migrated (legacy reports, dead code, deprecated features). Call these out explicitly.
- **Ask about undocumented behavior**: Users often know about special cases and workarounds that aren't in the code.

## Related Skills

- [/migrate-oracle-forms](./migrate-oracle-forms.md) - Oracle Forms-specific migration
- [/generate-domain-model](./generate-domain-model.md) - Creating entities and associations in MDL
- [/write-microflows](./write-microflows.md) - Implementing business logic in MDL
- [/organize-project](./organize-project.md) - Folder structure for the migrated project
