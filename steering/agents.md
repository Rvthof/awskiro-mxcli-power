# Agents — AI Agent Documents in Mendix

## Overview

Four AI agent document types stored as JSON inside Mendix MPR files:

| Type | CREATE keyword | Notes |
|------|---------------|-------|
| Model | `CREATE MODEL` | GenAI model configuration; required by Agent |
| Knowledge Base | `CREATE KNOWLEDGE BASE` | KB source; referenced by Agent body |
| Consumed MCP Service | `CREATE CONSUMED MCP SERVICE` | MCP tool server; referenced by Agent body |
| Agent | `CREATE AGENT` | Orchestrates model + tools + prompts |

**Requires:** `AgentEditorCommons` marketplace module, Mendix 11.9+.

## Syntax

### Model

```sql
CREATE MODEL Module.MyModel (
  Provider: MxCloudGenAI,   -- default, can omit
  Key: Module.ApiKeyConst   -- must be a String constant
);
```

### Knowledge Base

```sql
CREATE KNOWLEDGE BASE Module.ProductDocs (
  Provider: MxCloudGenAI,
  Key: Module.KBKeyConst
);
```

### Consumed MCP Service

```sql
CREATE CONSUMED MCP SERVICE Module.WebSearch (
  ProtocolVersion: v2025_03_26,
  Version: '1.0',
  ConnectionTimeoutSeconds: 30,
  Documentation: 'Web search MCP server'
);
```

### Agent (full syntax)

```sql
CREATE AGENT Module.MyAgent (
  UsageType: Task,              -- Task | Conversational
  Model: Module.MyModel,        -- must exist
  Description: 'Agent description',
  MaxTokens: 4096,
  Temperature: 0.7,
  TopP: 0.9,
  ToolChoice: Auto,
  Variables: ("Language": EntityAttribute, "Name": String),
  SystemPrompt: $Multi-line
prompt here.$,
  UserPrompt: 'Single line prompt.'
)
{
  MCP SERVICE Module.WebSearch {
    Enabled: true
  }

  KNOWLEDGE BASE KBAlias {
    Source: Module.ProductDocs,
    Collection: 'product-docs',
    MaxResults: 5,
    Description: 'Product docs',
    Enabled: true
  }

  TOOL MyMicroflowTool {
    Description: 'Fetch customer data',
    Enabled: true
  }
};
```

## Gotchas

### Dollar-quoting for multi-line prompts
Single-quoted strings cannot span lines. Use `$...$` for any SystemPrompt or UserPrompt that contains newlines.

### Portal-populated metadata fields
`DisplayName`, `KeyName`, `KeyID`, `Environment`, `ResourceName`, `DeepLinkURL` are populated by the Mendix portal at sync time. Do not set them manually.

### Drop order
Always drop Agents before dropping their dependencies:
```sql
DROP AGENT Module.MyAgent;
DROP CONSUMED MCP SERVICE Module.WebSearch;
DROP KNOWLEDGE BASE Module.ProductDocs;
DROP MODEL Module.MyModel;
```

### Variables: syntax
- `"Key": EntityAttribute` — binds an attribute from the entity in the agent's context
- `"Key": String` — binds a plain string value

## Common Patterns

### Minimal agent (no tools)
```sql
CREATE MODEL Module.M (Provider: MxCloudGenAI, Key: Module.K);
CREATE AGENT Module.A (
  UsageType: Task,
  Model: Module.M,
  SystemPrompt: 'You are a helpful assistant.',
  UserPrompt: 'Ask me anything.'
);
```

### Check all agent documents in a module
```sql
LIST MODELS IN Module;
LIST KNOWLEDGE BASES IN Module;
LIST CONSUMED MCP SERVICES IN Module;
LIST AGENTS IN Module;
```

## Calling Agents from Microflows

Use `CALL JAVA ACTION` with the AgentCommons Java actions:

```sql
-- Single-call (Task) agent — no chat history
$Response = CALL JAVA ACTION AgentCommons.Agent_Call_WithoutHistory(
  Agent = $Agent,
  UserMessage = $UserInput
);

-- Conversational agent — with chat history
$Response = CALL JAVA ACTION AgentCommons.Agent_Call_WithHistory(
  Agent = $Agent,
  ChatContext = $ChatContext,
  UserMessage = $UserInput
);
```

Retrieve the `AgentCommons.Agent` entity by qualified name before calling:

```sql
RETRIEVE $Agent FROM DATABASE AgentCommons.Agent
  WHERE AgentCommons.Agent/QualifiedName = 'Module.MyAgent'
  LIMIT 1;
```
