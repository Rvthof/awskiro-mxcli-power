---
name: "mendix-mxcli"
displayName: "Mendix MxCLI Development"
description: "AI-assisted Mendix development using MxCLI and MDL (Mendix Definition Language). Build domain models, microflows, pages, security, navigation, and more — all from the command line."
keywords: ["mendix", "mxcli", "mdl", "microflow", "nanoflow", "entity", "domain model", "page", "datagrid", "gallery", "snippet", "module", "enumeration", "association", "xpath", "oql", "atlas", "mendix studio", "mpr", "lint", "security", "navigation", "crud"]
author: "Mendix MxCLI Power"
---

# Mendix MxCLI Power

This power gives Kiro deep knowledge of Mendix development via MxCLI and MDL (Mendix Definition Language). It covers the full development lifecycle: domain modelling, microflows, pages, security, navigation, integrations, testing, and quality assessment.

> **Using Mendix via MCP instead?** If your project uses the Mendix Studio Pro MCP server, use the [Mendix MCP Power](https://github.com/Rvthof/kiropower-mendix-mcp) instead. This power is specifically for MxCLI/MDL-based development.

---

# Onboarding

## Step 1: Verify MxCLI is available

MxCLI is a local binary in the project root. Always use the local path — never rely on a system PATH entry.

```bash
./mxcli --version
```

If this fails, the binary may not be present or executable. Check the project root for `mxcli` (or `mxcli.exe` on Windows).

## Step 2: Connect to the project

All MxCLI commands require the `-p` flag pointing to the `.mpr` file:

```bash
./mxcli -p MyApp.mpr -c "SHOW MODULES"
```

To find the `.mpr` file:

```bash
ls *.mpr
```

## Step 3: Explore the project structure

```bash
./mxcli -p MyApp.mpr -c "SHOW STRUCTURE DEPTH 2"
./mxcli -p MyApp.mpr -c "SHOW MODULES"
```

## Step 4: Understand the MDL script workflow

MDL scripts live in `mdlsource/`. The workflow is always:

1. Write script to `mdlsource/my-script.mdl` using `fsWrite` + `fsAppend`
2. Validate: `./mxcli check mdlsource/my-script.mdl -p MyApp.mpr --references`
3. Fix any errors in the same file (never create a new file)
4. Execute: `./mxcli exec mdlsource/my-script.mdl -p MyApp.mpr`

## Step 5: Add the post-task test hook (optional)

For projects with Playwright tests, add this hook to `.kiro/hooks/run-tests-after-task.kiro.hook`:

```json
{
  "enabled": true,
  "name": "Run Tests After Task",
  "version": "1.0.0",
  "description": "Run Playwright tests after each spec task completes",
  "when": {
    "type": "postTaskExecution"
  },
  "then": {
    "type": "runCommand",
    "command": "npx playwright test --reporter=list"
  }
}
```

---

# Steering Instructions

## When to Load Steering Files

Load the appropriate steering file(s) based on what you're working on:

### Domain Model
| Working on... | Load this steering file |
|---|---|
| Domain model, entities, associations, enumerations | `generate-domain-model.md`, `mdl-entities.md` |

### Microflows & Logic
| Working on... | Load this steering file |
|---|---|
| Microflows, nanoflows, ACT_, SUB_, DS_, VAL_ | `write-microflows.md`, `cheatsheet-variables.md` |
| Validation microflows (VAL_) | `validation-microflows.md` |
| CRUD patterns | `patterns-crud.md` |
| Data processing (loops, aggregates, batch) | `patterns-data-processing.md` |
| XPath constraints | `xpath-constraints.md` |
| Debugging MDL errors, syntax issues | `cheatsheet-errors.md`, `check-syntax.md` |

### Pages & UI
| Working on... | Load this steering file |
|---|---|
| Pages, widgets, DATAGRID, GALLERY, DATAVIEW | `create-page.md` |
| Overview/list pages | `overview-pages.md` |
| Master-detail layouts | `master-detail-pages.md` |
| Modifying existing pages | `alter-page.md` |
| Bulk widget updates | `bulk-widget-updates.md` |
| Fragments (reusable widget groups) | `fragments.md` |
| Custom widgets (using existing) | `custom-widgets.md` |
| Building a new custom widget | `create-custom-widget.md` |

### Security & Navigation
| Working on... | Load this steering file |
|---|---|
| Security roles, GRANT, REVOKE, access rules | `manage-security.md` |
| Navigation profiles, menus, home pages | `manage-navigation.md` |

### Data & Configuration
| Working on... | Load this steering file |
|---|---|
| Demo data, seed data, IMPORT FROM | `demo-data.md` |
| Project settings, AfterStartup, constants | `project-settings.md` |
| Project organisation, folders, MOVE | `organize-project.md` |
| Writing MDL script files (fsWrite/fsAppend pattern) | `mdl-file-writing.md` |

### Integrations
| Working on... | Load this steering file |
|---|---|
| REST API consumption | `rest-client.md`, `rest-call-from-json.md` |
| JSON structures and mappings | `json-structures-and-mappings.md` |
| OData services | `odata-data-sharing.md` |
| External database connections | `database-connections.md` |
| Business events | `business-events.md` |
| Java actions | `java-actions.md` |
| Choosing integration approach | `browse-integrations.md` |
| RapidMiner / SPARQL graph | `connect-rapidminer-graph.md` |
| AI agents (GenAI, MCP) | `agents.md` |

### Quality, Testing & Infrastructure
| Working on... | Load this steering file |
|---|---|
| Lint, quality assessment, best practices report | `assess-quality.md`, `write-lint-rules.md` |
| OQL queries, VIEW entities | `write-oql-queries.md` |
| Verifying data with OQL | `verify-with-oql.md` |
| Playwright UI tests | `test-app.md` |
| Microflow unit tests | `test-microflows.md` |
| Docker build and run | `docker-workflow.md`, `run-app.md` |
| Runtime admin API (M2EE) | `runtime-admin-api.md` |
| System module entities | `system-module.md` |
| Theme and SCSS styling | `theme-styling.md` |
| BSON debugging | `debug-bson.md` |

### Migration
| Working on... | Load this steering file |
|---|---|
| Migration assessment (any technology) | `assess-migration.md` |
| K2 / Nintex migration | `migrate-k2-nintex.md` |
| Oracle Forms migration | `migrate-oracle-forms.md` |

## Core Principles (Always Apply)

### 0. Confirm the approach: MxCLI vs MCP

This power is for **MxCLI/MDL** workflows only. If the user mentions MCP, Studio Pro MCP server, or wants Kiro to talk directly to Studio Pro through an MCP client, stop and point them to the [Mendix MCP Power](https://github.com/Rvthof/kiropower-mendix-mcp) instead. Do not use MxCLI commands or MDL syntax for MCP-based workflows.

### 1. Always use the local mxcli binary
```bash
./mxcli -p MyApp.mpr    # ✅ correct
mxcli -p MyApp.mpr      # ❌ will fail
```

### 2. Always validate before executing
```bash
./mxcli check mdlsource/script.mdl -p MyApp.mpr --references
```

### 3. Always quote identifiers in MDL
Double-quote all entity names, attribute names, parameter names, and association names. Quotes are stripped automatically and prevent reserved keyword conflicts.

```sql
CREATE PERSISTENT ENTITY "MyModule"."Customer" (
  "Name": String(200),
  "Status": String(50)
);
```

### 4. Never show raw MDL in chat
Describe changes in plain language. Only show MDL if the user explicitly asks.

### 5. Use Administration.Account for user associations
Never use `System.User` as an association target. Always use `Administration.Account`.

### 6. Write MDL files in chunks
`fsWrite` truncates beyond ~50 lines. Always use `fsWrite` for the first ~40 lines, then `fsAppend` for subsequent chunks. See `mdl-file-writing.md`.
