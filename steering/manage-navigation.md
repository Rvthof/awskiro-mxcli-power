# Navigation Management Skill

This skill covers inspecting and modifying Mendix navigation profiles via MDL: home pages, menu items, login pages, role-based routing, and navigation catalog queries.

## When to Use This Skill

Use when the user asks to:
- View or change navigation home pages
- View or modify the navigation menu structure
- Set login or not-found pages
- Configure role-based home page routing
- Discover which pages are navigation entry points
- Set up navigation for a new project

## Navigation Concepts

- **Navigation Profiles** — Every Mendix project has navigation profiles: Responsive, Phone, Tablet, and optionally Native. Each profile has its own home page, menu, and login page.
- **Home Page** — The default page shown after login. Can be a PAGE or MICROFLOW.
- **Role-Based Home Pages** — Override the default home page per user role (e.g., admins see a dashboard, users see a task list).
- **Menu Items** — Hierarchical menu tree. Each item has a caption and optionally targets a PAGE or MICROFLOW. Sub-menus nest with `MENU 'Caption' (...)`.
- **Login Page** — Custom login page (optional; Mendix provides a default).
- **Not-Found Page** — Custom 404 page (optional).

## Show Commands (Read-Only)

```sql
-- Summary of all navigation profiles (home pages, menu counts)
SHOW NAVIGATION;

-- Full MDL description of a profile (round-trippable output)
DESCRIBE NAVIGATION Responsive;
DESCRIBE NAVIGATION;              -- all profiles

-- Menu tree for a specific profile
SHOW NAVIGATION MENU Responsive;
SHOW NAVIGATION MENU;             -- all profiles

-- Home page assignments across all profiles and roles
SHOW NAVIGATION HOMES;
```

## CREATE OR REPLACE NAVIGATION (Full Replacement)

This command fully replaces a navigation profile's configuration. All clauses are optional — omitted clauses clear that section. The output from `DESCRIBE NAVIGATION` can be pasted back directly.

### Basic: Set Home and Login Page

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  LOGIN PAGE Administration.Login;
```

### Role-Based Home Pages

Add `FOR Module.Role` to override the home page for specific user roles:

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  HOME PAGE MyModule.AdminDashboard FOR Administration.Administrator
  HOME PAGE MyModule.CustomerPortal FOR MyModule.Customer
  LOGIN PAGE Administration.Login;
```

### Full Menu Tree

The `MENU (...)` block replaces the entire menu. Use `MENU ITEM` for leaf items and `MENU 'Caption' (...)` for sub-menus:

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  LOGIN PAGE Administration.Login
  MENU (
    MENU ITEM 'Home' PAGE MyModule.Home_Web;
    MENU 'Orders' (
      MENU ITEM 'All Orders' PAGE Orders.Order_Overview;
      MENU ITEM 'New Order' PAGE Orders.Order_New;
    );
    MENU 'Admin' (
      MENU ITEM 'Users' PAGE Administration.Account_Overview;
      MENU ITEM 'Run Report' MICROFLOW Reports.ACT_GenerateReport;
    );
  );
```

### Clear the Menu

An empty `MENU ()` block removes all menu items:

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  MENU ();
```

### Not-Found Page

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  NOT FOUND PAGE MyModule.Custom404;
```

### Microflow as Home Page

Use `HOME MICROFLOW` instead of `HOME PAGE` to run a microflow on login:

```sql
CREATE OR REPLACE NAVIGATION Responsive
  HOME MICROFLOW MyModule.ACT_ShowHome;
```

## Round-Trip Workflow

The DESCRIBE output is directly executable. Use this pattern to inspect, modify, and re-apply:

```sql
-- Step 1: Inspect current state
DESCRIBE NAVIGATION Responsive;

-- Step 2: Copy the output, modify as needed, paste back
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  LOGIN PAGE Administration.Login
  MENU (
    MENU ITEM 'Home' PAGE MyModule.Home_Web;
    MENU ITEM 'New Feature' PAGE MyModule.NewFeature;
  );

-- Step 3: Verify
DESCRIBE NAVIGATION Responsive;
```

## Catalog Queries

After `REFRESH CATALOG FULL`, navigation references appear in the `REFS` table:

```sql
REFRESH CATALOG FULL;

-- Find all pages that are navigation entry points
SELECT SourceName, TargetName, RefKind
FROM CATALOG.REFS
WHERE RefKind IN ('home_page', 'menu_item', 'login_page');

-- What references point to a specific page?
SHOW REFERENCES TO MyModule.Home_Web;

-- Impact analysis: what breaks if I change this page?
SHOW IMPACT OF MyModule.Home_Web;

-- Full context for a page (includes navigation references)
SHOW CONTEXT OF MyModule.Home_Web;
```

## Common Patterns

### New Project Setup

Set up navigation for a freshly created project:

```sql
-- Create home page
CREATE PAGE MyModule.Home_Web
(
  Title: 'Home',
  Layout: Atlas_Core.Atlas_Default
)
{
  CONTAINER ctnMain {
    DYNAMICTEXT txtWelcome (Content: 'Welcome!')
  }
}

-- Configure navigation
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  MENU (
    MENU ITEM 'Home' PAGE MyModule.Home_Web;
  );
```

### Adding a New Page to Navigation

After creating a new page, add it to the menu:

```sql
-- First inspect current menu
DESCRIBE NAVIGATION Responsive;

-- Then re-apply with the new item added (copy existing + add new)
CREATE OR REPLACE NAVIGATION Responsive
  HOME PAGE MyModule.Home_Web
  LOGIN PAGE Administration.Login
  MENU (
    MENU ITEM 'Home' PAGE MyModule.Home_Web;
    MENU ITEM 'Customers' PAGE MyModule.Customer_Overview;  -- new
    MENU 'Admin' (
      MENU ITEM 'Users' PAGE Administration.Account_Overview;
    );
  );
```

## Checklist

- [ ] Profile name matches an existing profile (Responsive, Phone, Tablet, or a native profile)
- [ ] All PAGE/MICROFLOW targets are fully qualified (`Module.Name`)
- [ ] Role references in `FOR` clauses are fully qualified (`Module.Role`)
- [ ] Every `MENU ITEM` and `MENU 'caption' (...)` ends with `;`
- [ ] Sub-menu items are wrapped in `MENU 'Caption' ( ... );`
- [ ] Use `DESCRIBE NAVIGATION` to verify changes after applying
