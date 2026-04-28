# MDL Entity Syntax Reference

Complete syntax reference for creating entities, attributes, and associations.

## Entity Types

| Type | Keyword | Stored in DB | Use Case |
|------|---------|--------------|----------|
| Persistent | `CREATE PERSISTENT ENTITY` | Yes | Business data |
| Non-Persistent | `CREATE NON-PERSISTENT ENTITY` | No | Temporary/view data |
| View | `CREATE VIEW ENTITY` | No (OQL query) | Aggregated/computed data |

## Persistent Entity

```mdl
/**
 * Customer entity for storing customer data
 */
CREATE PERSISTENT ENTITY Module.Customer (
  -- String attributes
  Name: String(100) NOT NULL,
  Email: String(200),
  Code: String(20) UNIQUE,

  -- Numeric attributes
  Age: Integer,
  CreditLimit: Decimal,

  -- Boolean
  IsActive: Boolean DEFAULT true,

  -- Date/Time
  CreatedDate: DateTime,
  BirthDate: Date,

  -- Enumeration
  Status: Module.CustomerStatus DEFAULT Active,

  -- Auto-number
  CustomerNumber: AutoNumber
);
/
```

## Non-Persistent Entity

Used for temporary data, form parameters, or calculated values.

```mdl
/**
 * Search parameters for customer search form
 */
CREATE NON-PERSISTENT ENTITY Module.CustomerSearchParams (
  SearchName: String(100),
  SearchEmail: String(200),
  MinCreditLimit: Decimal,
  IncludeInactive: Boolean DEFAULT false
);
/
```

## Attribute Types

| Type | Syntax | Example |
|------|--------|---------|
| String | `Name: String(length)` | `Name: String(100)` |
| Integer | `Name: Integer` | `Count: Integer` |
| Long | `Name: Long` | `BigNumber: Long` |
| Decimal | `Name: Decimal` | `Amount: Decimal` |
| Boolean | `Name: Boolean` | `IsActive: Boolean` |
| DateTime | `Name: DateTime` | `CreatedAt: DateTime` |
| Date | `Name: Date` | `BirthDate: Date` |
| Enumeration | `Name: Module.EnumName` | `Status: Module.Status` |
| AutoNumber | `Name: AutoNumber` | `Code: AutoNumber` |
| Binary | `Name: Binary` | `FileData: Binary` |
| Hashed String | `Name: HashedString` | `Password: HashedString` |

## Attribute Modifiers

| Modifier | Meaning | Example |
|----------|---------|---------|
| `NOT NULL` | Required field | `Name: String(100) NOT NULL` |
| `UNIQUE` | Unique constraint | `Code: String(20) UNIQUE` |
| `DEFAULT value` | Default value | `IsActive: Boolean DEFAULT true` |

**Note:** Boolean attributes auto-default to `false` when no `DEFAULT` is specified.

## Generalization (Inheritance)

**CRITICAL: EXTENDS goes BEFORE the opening parenthesis, not after!**

```mdl
/**
 * Base entity
 */
CREATE PERSISTENT ENTITY Module.Person (
  PersonName: String(100) NOT NULL,
  Email: String(200)
);
/

/**
 * Customer extends Person - EXTENDS before (
 */
CREATE PERSISTENT ENTITY Module.Customer EXTENDS Module.Person (
  CustomerCode: String(20),
  CreditLimit: Decimal
);
/
```

Common parent entities for file/image storage:
```mdl
-- Image entity (inherits Name, Size, Contents, thumbnail)
CREATE PERSISTENT ENTITY Module.ProductPhoto EXTENDS System.Image (
  PhotoCaption: String(200),
  SortOrder: Integer DEFAULT 0
);

-- File document (inherits Name, Size, Contents)
CREATE PERSISTENT ENTITY Module.Attachment EXTENDS System.FileDocument (
  AttachmentDescription: String(500)
);
```

**Wrong** (parse error):
```mdl
-- EXTENDS after ) = parse error!
CREATE PERSISTENT ENTITY Module.Photo (
  PhotoCaption: String(200)
) EXTENDS System.Image;
```

## Associations

### Reference (Many-to-One)

```mdl
/**
 * Order belongs to one Customer
 */
CREATE ASSOCIATION Module.Order_Customer (
  PARENT Module.Customer,
  CHILD Module.Order
);
/
```

### Reference Set (Many-to-Many)

```mdl
/**
 * Product can be in many Categories
 * Category can have many Products
 */
CREATE ASSOCIATION Module.Product_Category (
  PARENT Module.Category AS REFERENCE SET,
  CHILD Module.Product
);
/
```

### Association with Delete Behavior

```mdl
/**
 * Delete orders when customer is deleted
 */
CREATE ASSOCIATION Module.Order_Customer (
  PARENT Module.Customer,
  CHILD Module.Order,
  DELETE PARENT CASCADE  -- Delete orders when customer deleted
);
/
```

Delete behaviors:
- `DELETE PARENT CASCADE` - Delete children when parent deleted
- `DELETE PARENT PREVENT` - Prevent deletion if children exist
- `DELETE CHILD CASCADE` - Delete parent when last child deleted

## Enumerations

```mdl
/**
 * Order status values
 */
CREATE ENUMERATION Module.OrderStatus (
  Draft = 'Draft',
  Pending = 'Pending',
  Approved = 'Approved',
  Shipped = 'Shipped',
  Delivered = 'Delivered',
  Cancelled = 'Cancelled'
);
/
```

## View Entity (OQL)

```mdl
/**
 * Monthly sales summary by customer
 */
CREATE VIEW ENTITY Module.CustomerSalesSummary (
  CustomerName: String(100),
  TotalOrders: Integer,
  TotalAmount: Decimal,
  LastOrderDate: DateTime
)
AS
  SELECT
    c.Name as CustomerName,
    count(o.OrderID) as TotalOrders,
    sum(o.Amount) as TotalAmount,
    max(o.OrderDate) as LastOrderDate
  FROM Module.Customer c
  LEFT JOIN c/Module.Order_Customer/Module.Order o
  GROUP BY c.Name;
/
```

## Entity with Index

```mdl
/**
 * Product with search index
 */
CREATE PERSISTENT ENTITY Module.Product (
  Code: String(20) NOT NULL,
  Name: String(100) NOT NULL,
  Category: String(50),
  Price: Decimal
)
INDEX idx_product_code ON (Code)
INDEX idx_product_category ON (Category);
/
```

## Complete Domain Model Example

```mdl
-- Enumeration
CREATE ENUMERATION Shop.OrderStatus (
  Draft = 'Draft',
  Confirmed = 'Confirmed',
  Shipped = 'Shipped',
  Delivered = 'Delivered'
);
/

-- Customer entity
CREATE PERSISTENT ENTITY Shop.Customer (
  Name: String(100) NOT NULL,
  Email: String(200) NOT NULL UNIQUE,
  Phone: String(20),
  IsActive: Boolean DEFAULT true,
  CreatedDate: DateTime
);
/

-- Product entity
CREATE PERSISTENT ENTITY Shop.Product (
  Code: String(20) NOT NULL UNIQUE,
  Name: String(100) NOT NULL,
  Description: String(500),
  Price: Decimal NOT NULL,
  Stock: Integer DEFAULT 0,
  IsAvailable: Boolean DEFAULT true
);
/

-- Order entity
CREATE PERSISTENT ENTITY Shop.Order (
  OrderNumber: AutoNumber,
  OrderDate: DateTime NOT NULL,
  Status: Shop.OrderStatus DEFAULT Draft,
  TotalAmount: Decimal,
  Notes: String(500)
);
/

-- Order line entity
CREATE PERSISTENT ENTITY Shop.OrderLine (
  Quantity: Integer NOT NULL,
  UnitPrice: Decimal NOT NULL,
  LineTotal: Decimal
);
/

-- Associations
CREATE ASSOCIATION Shop.Order_Customer (
  PARENT Shop.Customer,
  CHILD Shop.Order
);
/

CREATE ASSOCIATION Shop.OrderLine_Order (
  PARENT Shop.Order,
  CHILD Shop.OrderLine,
  DELETE PARENT CASCADE
);
/

CREATE ASSOCIATION Shop.OrderLine_Product (
  PARENT Shop.Product,
  CHILD Shop.OrderLine
);
/
```

## Quick Reference

### Entity Creation
```mdl
CREATE PERSISTENT ENTITY Module.Name (attributes);
CREATE NON-PERSISTENT ENTITY Module.Name (attributes);
CREATE VIEW ENTITY Module.Name (attributes) AS SELECT ...;
```

### Attribute Syntax
```mdl
AttributeName: Type [(length)] [NOT NULL] [UNIQUE] [DEFAULT value]
```

### Association Syntax
```mdl
CREATE ASSOCIATION Module.Name (
  PARENT Module.ParentEntity [AS REFERENCE SET],
  CHILD Module.ChildEntity
  [, DELETE PARENT CASCADE|PREVENT]
);
```

### Enumeration Syntax
```mdl
CREATE ENUMERATION Module.Name (
  Value1 = 'Caption1',
  Value2 = 'Caption2'
);
```
