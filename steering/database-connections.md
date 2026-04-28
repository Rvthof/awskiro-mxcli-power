# Skill: Create External Database Connections

## Purpose
Create and manage external database connections in Mendix using the External Database Connector. This skill helps you set up JDBC connections to external databases (Oracle, PostgreSQL, MySQL, SQL Server, etc.) and define SQL queries that map results to non-persistent entities.

## When to Use This Skill
- User asks to connect to an external database **from a Mendix app** (via JDBC)
- User needs to query data from Oracle, PostgreSQL, MySQL, SQL Server, or other JDBC databases
- User wants to create database connection configurations
- User needs to define SQL queries with parameter binding
- User wants to map query results to Mendix entities

> **Tip:** Use `GENERATE CONNECTOR` to auto-create all constants, entities, and queries from a database schema:
> ```
> SQL CONNECT postgres 'postgres://user:pass@host/db' AS source;
> SQL source GENERATE CONNECTOR INTO MyModule;
> -- Or generate for specific tables and execute immediately:
> SQL source GENERATE CONNECTOR INTO MyModule TABLES (employees, departments) EXEC;
> ```
> For manual exploration, use `SQL source SHOW TABLES;` and `SQL source DESCRIBE tablename;`.

## Prerequisites

### 1. Required Mendix Version
- Mendix 9.22+ (Database Connector introduced)
- Mendix 10.10+ (stable version recommended)

### 2. Required Non-Persistent Entities
Database query results must be mapped to NON-PERSISTENT entities. Create these first:

```sql
-- Entity to hold query results
CREATE NON-PERSISTENT ENTITY MyModule.EmployeeRecord (
  EmployeeId: Integer,
  EmployeeName: String(100),
  Department: String(50),
  Salary: Decimal
);
```

### 3. Required Constants
Connection credentials should be stored in constants:

```sql
-- Connection string (JDBC URL)
CREATE CONSTANT MyModule.DbConnectionString TYPE String
  DEFAULT 'jdbc:oracle:thin:@//hostname:1521/SERVICENAME'
  COMMENT 'JDBC connection string for external database';

-- Username
CREATE CONSTANT MyModule.DbUsername TYPE String
  DEFAULT 'app_user'
  COMMENT 'Database username';

-- Password (use PRIVATE for local development)
CREATE CONSTANT MyModule.DbPassword TYPE String
  DEFAULT ''
  PRIVATE
  COMMENT 'Database password - inject via environment variable in production';
```

## Database Connection Syntax

### Basic Connection Structure

```sql
CREATE DATABASE CONNECTION Module.ConnectionName
TYPE '<database-type>'
CONNECTION STRING @Module.ConnectionStringConstant
USERNAME @Module.UsernameConstant
PASSWORD @Module.PasswordConstant
BEGIN
  -- Query definitions go here
END;
```

### Supported Database Types

| Database | TYPE Value |
|----------|------------|
| Oracle | `'Oracle'` |
| PostgreSQL | `'PostgreSQL'` |
| MySQL | `'MySQL'` |
| SQL Server | `'MSSQL'` or `'SQLServer'` |
| Snowflake | `'Snowflake'` |
| Amazon Redshift | `'Redshift'` |

## Query Definition Syntax

### Simple Query (No Parameters)

```sql
QUERY QueryName
  SQL 'SELECT column1, column2 FROM table_name'
  RETURNS Module.EntityName;
```

### Parameterized Query

```sql
QUERY QueryName
  SQL 'SELECT * FROM table_name WHERE column = {paramName}'
  PARAMETER paramName: String
  RETURNS Module.EntityName;
```

### Query with Column Mapping

When database column names don't match entity attribute names:

```sql
QUERY QueryName
  SQL 'SELECT emp_id, emp_name, dept_no FROM employees'
  RETURNS Module.EmployeeRecord
  MAP (
    emp_id AS EmployeeId,
    emp_name AS EmployeeName,
    dept_no AS DepartmentNumber
  );
```

### Supported Parameter Types

- `String` - Text values
- `Integer` - Whole numbers
- `Decimal` - Decimal numbers
- `Boolean` - true/false
- `DateTime` - Date and time values

### Parameter Test Values

Parameters can include a test value for Studio Pro testing, or indicate they should be tested with NULL:

```sql
-- Test value (used in Studio Pro's Execute Query dialog)
PARAMETER empName: String DEFAULT 'Smith'

-- Test with NULL value
PARAMETER optionalDate: DateTime NULL
```

## Complete Examples

### Example 1: Oracle HR Database Connection

```sql
-- Step 1: Create module
CREATE MODULE OracleDemo;

-- Step 2: Create constants for connection
CREATE CONSTANT OracleDemo.OracleConnectionString TYPE String
  DEFAULT 'jdbc:oracle:thin:@//10.211.55.2:1522/ORCLPDB1';

CREATE CONSTANT OracleDemo.OracleUser TYPE String DEFAULT 'scott';

CREATE CONSTANT OracleDemo.OraclePassword TYPE String DEFAULT 'tiger' PRIVATE;

-- Step 3: Create non-persistent entity for results
CREATE NON-PERSISTENT ENTITY OracleDemo.EmpRecord (
  EMPNO: Decimal,
  ENAME: String(10),
  JOB: String(9),
  SAL: Decimal,
  DEPTNO: Decimal
);

-- Step 4: Create database connection
CREATE DATABASE CONNECTION OracleDemo.HRDatabase
TYPE 'Oracle'
CONNECTION STRING @OracleDemo.OracleConnectionString
USERNAME @OracleDemo.OracleUser
PASSWORD @OracleDemo.OraclePassword
BEGIN
  QUERY GetAllEmployees
    SQL 'SELECT EMPNO, ENAME, JOB, SAL, DEPTNO FROM EMP ORDER BY EMPNO'
    RETURNS OracleDemo.EmpRecord;

  QUERY GetEmployeeByName
    SQL 'SELECT EMPNO, ENAME, JOB, SAL, DEPTNO FROM EMP WHERE ENAME = {empName}'
    PARAMETER empName: String
    RETURNS OracleDemo.EmpRecord;

  QUERY GetHighEarners
    SQL 'SELECT EMPNO, ENAME, JOB, SAL, DEPTNO FROM EMP WHERE SAL >= {minSalary}'
    PARAMETER minSalary: Decimal
    RETURNS OracleDemo.EmpRecord;
END;
```

### Example 2: PostgreSQL Connection

```sql
CREATE CONSTANT Inventory.PgConnectionString TYPE String
  DEFAULT 'jdbc:postgresql://localhost:5432/inventory_db';

CREATE CONSTANT Inventory.PgUser TYPE String DEFAULT 'inventory_app';
CREATE CONSTANT Inventory.PgPassword TYPE String DEFAULT '' PRIVATE;

CREATE NON-PERSISTENT ENTITY Inventory.ProductRecord (
  ProductId: Integer,
  ProductName: String(100),
  Quantity: Integer,
  Price: Decimal
);

CREATE DATABASE CONNECTION Inventory.ProductDatabase
TYPE 'PostgreSQL'
CONNECTION STRING @Inventory.PgConnectionString
USERNAME @Inventory.PgUser
PASSWORD @Inventory.PgPassword
BEGIN
  QUERY GetAllProducts
    SQL 'SELECT product_id, product_name, quantity, price FROM products'
    RETURNS Inventory.ProductRecord
    MAP (
      product_id AS ProductId,
      product_name AS ProductName,
      quantity AS Quantity,
      price AS Price
    );

  QUERY SearchProducts
    SQL 'SELECT product_id, product_name, quantity, price FROM products WHERE product_name ILIKE {searchPattern}'
    PARAMETER searchPattern: String
    RETURNS Inventory.ProductRecord
    MAP (
      product_id AS ProductId,
      product_name AS ProductName,
      quantity AS Quantity,
      price AS Price
    );
END;
```

## Viewing Connections

```sql
-- List all database connections
SHOW DATABASE CONNECTIONS;

-- List connections in a specific module
SHOW DATABASE CONNECTIONS IN MyModule;

-- View connection source code
DESCRIBE DATABASE CONNECTION MyModule.MyDatabase;
```

## Best Practices

### 1. Connection String Management
- Store JDBC URLs in constants for environment-specific overrides
- Use `MX_Module_ConstantName` environment variables in production

### 2. Credential Security
- Use `PRIVATE` flag for password constants during development
- Never commit real passwords to version control
- Inject credentials via CI/CD pipelines in production

### 3. Entity Design
- Use NON-PERSISTENT entities for query results
- Match attribute types to database column types
- Use MAP clause when column names differ from attribute names

### 4. Query Design
- Use parameterized queries to prevent SQL injection
- Keep queries simple and focused
- Create separate queries for different use cases

## Troubleshooting

### Connection Issues
1. Verify JDBC URL format for your database type
2. Check network connectivity to database host
3. Verify credentials are correct
4. Ensure JDBC driver is available

### Query Issues
1. Test queries directly in database client first
2. Check parameter types match expected database types
3. Verify entity attributes match query result columns
4. Use MAP clause for column name mismatches

## Related Commands

```sql
-- Constants for configuration
CREATE CONSTANT Module.Name TYPE String DEFAULT 'value';
SHOW CONSTANTS IN Module;

-- Non-persistent entities for results
CREATE NON-PERSISTENT ENTITY Module.Name (...);
SHOW ENTITIES IN Module;
```

## Executing Queries from Microflows

Once a database connection and queries are defined, execute them from microflows using `EXECUTE DATABASE QUERY`. The query is referenced by its **3-part qualified name**: `Module.Connection.Query`.

### Basic Syntax

```sql
-- Execute a query and store results
$ResultList = EXECUTE DATABASE QUERY Module.Connection.QueryName;

-- Fire-and-forget (no output variable)
EXECUTE DATABASE QUERY Module.Connection.QueryName;
```

### Dynamic SQL Override

Override the query's SQL at runtime using `DYNAMIC`:

```sql
$ResultList = EXECUTE DATABASE QUERY Module.Connection.QueryName
  DYNAMIC 'SELECT id, name FROM employees WHERE active = true LIMIT 10';
```

### Parameterized Queries

Pass values for query parameters defined with `PARAMETER` in the query definition:

```sql
-- Query definition (in DATABASE CONNECTION block):
--   QUERY GetDriversByNationality
--     SQL 'SELECT * FROM drivers WHERE nationality = {nation}'
--     PARAMETER nation: String
--     RETURNS Module.DriverRecord;

-- Microflow execution:
$Drivers = EXECUTE DATABASE QUERY Module.Connection.GetDriversByNationality
  (nation = $NationalityVar);
```

**CRITICAL**: Parameter names must exactly match those in the query definition (e.g., `nation`, not `nationality`). Mismatched names cause Studio Pro to regenerate mappings and clear values.

### Runtime Connection Override

Override connection parameters at runtime using `CONNECTION`. Use case: multiple databases with the same schema but different data (e.g., region-specific databases).

```sql
$Results = EXECUTE DATABASE QUERY Module.Connection.QueryName
  CONNECTION (DBSource = $Url, DBUsername = $User, DBPassword = $Pass);
```

**Caveat**: ConnectionParameterMappings require the database connection to have been tested/validated in Studio Pro first. Creating them programmatically may trigger "parameters have been updated" on first open.

### Error Handling

`EXECUTE DATABASE QUERY` only supports `ON ERROR ROLLBACK` (the default). `ON ERROR CONTINUE` is **not supported** for this action type.

### Complete Example

```sql
-- Set up non-persistent entity, constants, and connection
CREATE NON-PERSISTENT ENTITY HR.EmployeeRecord (
  EmpId: Integer,
  Name: String(100),
  Department: String(50)
);

CREATE CONSTANT HR.DbUrl TYPE String DEFAULT 'jdbc:postgresql://localhost:5432/hrdb';
CREATE CONSTANT HR.DbUser TYPE String DEFAULT 'app';
CREATE CONSTANT HR.DbPass TYPE String DEFAULT '' PRIVATE;

CREATE DATABASE CONNECTION HR.MainDB
TYPE 'PostgreSQL'
CONNECTION STRING @HR.DbUrl
USERNAME @HR.DbUser
PASSWORD @HR.DbPass
BEGIN
  QUERY GetAllEmployees
    SQL 'SELECT emp_id, name, department FROM employees'
    RETURNS HR.EmployeeRecord
    MAP (emp_id AS EmpId, name AS Name, department AS Department);

  QUERY GetByDepartment
    SQL 'SELECT emp_id, name, department FROM employees WHERE department = {dept}'
    PARAMETER dept: String
    RETURNS HR.EmployeeRecord
    MAP (emp_id AS EmpId, name AS Name, department AS Department);
END;

-- Microflow that executes the query
CREATE MICROFLOW HR.ACT_LoadEmployees($Department: String)
RETURNS List of HR.EmployeeRecord AS $Employees
BEGIN
  $Employees = EXECUTE DATABASE QUERY HR.MainDB.GetByDepartment
    (dept = $Department);
  RETURN $Employees;
END;
```

## Importing Data from External Databases

To bulk-import data from an external database directly into the Mendix app's PostgreSQL
database (bypassing the runtime), use `IMPORT FROM` instead of the Database Connector:

```sql
SQL CONNECT postgres 'postgres://user:pass@host:5432/legacydb' AS source;
IMPORT FROM source QUERY 'SELECT name, email FROM employees'
  INTO HRModule.Employee
  MAP (name AS Name, email AS Email);
```

See [demo-data.md](./demo-data.md) for details on the Mendix ID system and manual insertion.

## References

- [Mendix External Database Connector](https://docs.mendix.com/appstore/modules/external-database-connector/)
- [JDBC Connection Strings](https://docs.mendix.com/appstore/modules/external-database-connector/#connection-details)
