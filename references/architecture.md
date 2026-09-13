# Architecture

## Execution hierarchy

Qefro routes user messages through four tiers, from cheapest to most capable:

```
1. Deterministic patterns  (direct match, bypasses LLM)
2. Fast Router             (pattern + optional semantic embedding match)
3. FlowRunner              (deterministic multi-step flows)
4. Agent                   (LLM-driven multi-tool reasoning loop)
```

Each tier uses the same tools, auth, and audit. Higher tiers are only reached when lower tiers cannot handle the request.

## Metadata-first principle

Marketplace Apps are defined entirely in YAML metadata. The runtime interprets this metadata at execution time. There is no per-app compiled code.

```
manifest.yaml          -- app identity, capabilities, triggers, slots
entities/<id>.yaml     -- data model (fields, relations, scope)
workflows/<id>.yaml    -- conversational flows (steps, choices, tools)
connections/<id>.yaml  -- HTTP integration config (HTTP-backed apps only)
tools/<id>.yaml        -- HTTP tool definitions (HTTP-backed apps only)
ui/                    -- pages, navigation, widgets, theme (vertical apps)
```

## Two app categories

### HTTP-backed apps (integrations)

Apps like shopify, stripe, slack, freshdesk. These have:
- `connections/` -- OAuth/bearer/basic auth config for external APIs
- `tools/` -- HTTP tool definitions with path, method, headers, body templates
- Webhook metadata for inbound events from the provider

### Entity-native apps (vertical solutions)

Apps like clinic-pro, restaurant-pro, real-estate-pro, appointment, education, logistics. These have:
- `entities/` -- data models with `entity.*` managed storage
- `workflows/` -- conversational flows using entity CRUD tools
- `ui/` -- pages, navigation, widgets, theme

Entity-native apps use `entity.{name}.{create|list|get|update|delete}` tools executed by the RuntimeAdapter directly against the platform storage service. No external server is involved.

## Hosting constraint

All apps MUST declare `hosting: runtime`. The validation suite rejects `hosting: custom`. Apps are not allowed to run custom server code.

## App directory structure

```
apps/<app-id>/
  manifest.yaml
  connections/          # HTTP-backed only
    <conn-id>.yaml
  tools/                # HTTP-backed only
    <tool-id>.yaml
  entities/             # Entity-native only
    <entity-id>.yaml
  workflows/            # All apps
    <flow-id>.yaml
  ui/                   # Vertical apps
    pages.yaml
    navigation.yaml
    widgets.yaml
    sources.yaml
    layouts.yaml
    theme.yaml
```

## Key Rust crates

| Crate | Purpose |
|-------|---------|
| `crates/api/src/flow_engine/executor.rs` | FlowRunner step state machine |
| `crates/api/src/flow_engine/runtime_adapter.rs` | Marketplace entity CRUD execution |
| `crates/api/src/flow_engine/adapter.rs` | ExecutionBackend dispatch (Runtime vs SDK) |
| `crates/api/src/flow_engine/steps.rs` | Step handlers (ask, tool, message, complete, condition) |
| `crates/api/src/flow_engine/variables.rs` | input_map resolution, $literal, $constants |
| `crates/api/src/flow_engine/catalog.rs` | Flow selection via cosine similarity |
| `crates/api/src/flow_engine/mod.rs` | try_advance_or_start, conversation orchestration |
| `crates/api/src/agent/` | Agent layer, capability projection, staging |
| `crates/api/src/services/person.rs` | Customer Hub Person service |
| `crates/api/src/conversation_protocol.rs` | Conversation slot harvesting |
| `crates/domain/src/` | Domain types (FlowExecution, Person, Tool, etc.) |
