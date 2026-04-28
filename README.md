# Mendix MxCLI Power for Kiro

A [Kiro Power](https://kiro.dev/docs/powers/) that gives the Kiro agent deep, accurate knowledge of Mendix development using [MxCLI](https://github.com/mendixlabs/mxcli) and MDL (Mendix Definition Language).

---

## What it does

When you're working on a Mendix project with MxCLI, this power loads the right reference material into Kiro's context automatically — based on what you're working on. Ask about microflows and it loads the microflow syntax guide. Ask about security and it loads the access rule reference. You get precise, project-aware help without manually pointing Kiro at documentation.

**Covers the full Mendix development lifecycle:**

- Domain modelling — entities, associations, enumerations, OQL view entities
- Microflows and nanoflows — ACT_, SUB_, DS_, VAL_, NF_ patterns, error handling, REST calls
- Pages and widgets — DATAGRID, GALLERY, DATAVIEW, COMBOBOX, master-detail, fragments
- Security — module roles, user roles, entity access rules, XPath constraints, demo users
- Navigation — profiles, menus, role-based home pages
- Integrations — REST, OData, external databases, business events, Java actions, AI agents
- Quality — lint rules, best practices report, OQL verification
- Testing — Playwright UI tests, microflow unit tests
- Infrastructure — Docker build/run, hot reload, M2EE admin API
- Migration — assessment framework, K2/Nintex, Oracle Forms

**49 steering files** — every one a direct copy of a battle-tested skill file, not a summary. The lessons learned from real projects (wrong association targets, missing `ShowContentAs`, FOOTER placement rules, and more) are all in there.

---

## How it works

Kiro Powers use keyword-based activation. When you mention words like `mendix`, `mxcli`, `mdl`, `microflow`, `entity`, `datagrid`, or any of the other keywords in the power's frontmatter, Kiro activates this power and gains access to its steering files.

Rather than loading all 49 files at once (which would flood the context), the power's `POWER.md` contains a steering map — a table that tells Kiro which file to load for which task. When you ask about pages, it loads `create-page.md`. When you ask about OQL, it loads `write-oql-queries.md`. Context stays focused.

The power has no MCP server — it's documentation-only. MxCLI itself is a local binary in your project root, so no server configuration is needed.

---

## Requirements

You need a Mendix project set up with MxCLI. There are two ways to get there:

### Option A — New project

Use `mxcli new` to create a blank Mendix project with everything configured in one command:

```bash
mxcli new MyApp --version 11.8.0
```

This downloads MxBuild, creates a blank project, sets up AI tooling and a Dev Container, and installs the correct `mxcli` binary. Open the resulting folder in VS Code and reopen in the Dev Container.

### Option B — Existing project

For an existing Mendix project, run `mxcli init` to add AI tooling and a Dev Container:

```bash
mxcli init /path/to/my-mendix-project
```

Either way, you'll end up with `mxcli` (or `mxcli.exe`) in the project root and an `.mpr` file to work with.

> **Get MxCLI:** Download the latest release from [github.com/mendixlabs/mxcli/releases](https://github.com/mendixlabs/mxcli/releases) or see the [MxCLI README](https://github.com/mendixlabs/mxcli) for full installation instructions.

---

## Installation

1. Open Kiro
2. Click the **Powers** panel (the ⚡ icon in the sidebar)
3. Click **Add power from GitHub**
4. Enter this repository's URL
5. Click **Install**

After installing, open your Mendix project folder in Kiro. The power's onboarding steps will guide you through verifying `./mxcli` is available, connecting to your `.mpr` file, and understanding the MDL script workflow.

### Verify it's working

Try asking Kiro something like:

> "Create a Customer entity with Name, Email, and IsActive attributes"

Kiro should activate the power and produce correct MDL syntax using `generate-domain-model.md` as its reference.

---

## Steering files

All 49 steering files live in `steering/`. Each one is a direct copy of a skill file — full syntax references, working examples, common mistakes, and known gotchas.

| Category | Files |
|---|---|
| Domain model | `generate-domain-model.md`, `mdl-entities.md` |
| Microflows | `write-microflows.md`, `cheatsheet-variables.md`, `validation-microflows.md`, `patterns-crud.md`, `patterns-data-processing.md`, `xpath-constraints.md` |
| Debugging | `cheatsheet-errors.md`, `check-syntax.md` |
| Pages | `create-page.md`, `overview-pages.md`, `master-detail-pages.md`, `alter-page.md`, `bulk-widget-updates.md`, `fragments.md`, `custom-widgets.md`, `create-custom-widget.md` |
| Security & Navigation | `manage-security.md`, `manage-navigation.md` |
| Data & Config | `demo-data.md`, `project-settings.md`, `organize-project.md`, `mdl-file-writing.md` |
| Integrations | `rest-client.md`, `rest-call-from-json.md`, `json-structures-and-mappings.md`, `odata-data-sharing.md`, `database-connections.md`, `business-events.md`, `java-actions.md`, `browse-integrations.md`, `connect-rapidminer-graph.md`, `agents.md` |
| Quality & Testing | `assess-quality.md`, `write-lint-rules.md`, `write-oql-queries.md`, `verify-with-oql.md`, `test-app.md`, `test-microflows.md` |
| Infrastructure | `docker-workflow.md`, `run-app.md`, `runtime-admin-api.md`, `system-module.md`, `theme-styling.md`, `debug-bson.md` |
| Migration | `assess-migration.md`, `migrate-k2-nintex.md`, `migrate-oracle-forms.md` |

---

## Structure

```
power-mendix-mxcli/
├── POWER.md          # Power metadata, onboarding steps, steering map
├── README.md         # This file
└── steering/         # 49 reference files, one per topic
    ├── generate-domain-model.md
    ├── write-microflows.md
    ├── create-page.md
    └── ...
```
