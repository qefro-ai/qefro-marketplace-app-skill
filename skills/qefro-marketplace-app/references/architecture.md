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

Meta WhatsApp Flow is **not** a fifth tier. It is a presentation/input surface over an existing Marketplace workflow (`whatsapp_flow:` on workflow YAML). FlowRunner still runs the procedure; RuntimeAdapter still mutates entities.

WhatsApp Catalog is **optional** and **not** a sixth tier and not a second catalog vs FlowRunner `flow_engine/catalog.rs`. It is a discovery/presentation overlay over an existing Marketplace **entity** (`catalog:` on entity YAML). Catalog Service maps records; the WhatsApp adapter talks to Meta. WhatsApp and Flows work when no Meta catalog is attached. The same entity may use conversational, Flow, and Catalog presentations independently.

## Metadata-first principle

Marketplace Apps are defined entirely in YAML metadata. The runtime interprets this metadata at execution time. There is no per-app compiled code.

```
manifest.yaml          -- app identity, permissions/capabilities, settings, triggers, slots, reactions
entities/<id>.yaml     -- fields, relations, integrity, availability, line items, presentation
workflows/<id>.yaml    -- procedures, choices, tools, optional WhatsApp Flow surface
connections/<id>.yaml  -- HTTP integration config (HTTP-backed apps only)
tools/<id>.yaml        -- HTTP tool definitions (HTTP-backed apps only)
ui/                    -- pages, navigation, sources, layouts, widgets, theme
```

Verify each declaration across the package parser, installed projection and actual consumer. Runtime executes entity integrity and workflows; the portal renders presentation metadata. `entity.ui.form.related_create` only initializes a generic staff child form and displays parent context; it does not execute FlowRunner or bypass CRUD validation. Availability and line-item integrity belong to the generic server primitives, not to app-specific controls.

Use Restaurant Pro for parent/child transactions and Real Estate Pro for search/booking/lead relationships. Adapt field mappings and labels, not the runtime. See [Reference apps](reference-apps.md) for a source-by-source map.

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
| `crates/domain/src/whatsapp_flow.rs` | Thin `whatsapp_flow` overlay parse + Meta Flow JSON compile |
| `crates/api/src/whatsapp_flows.rs` | Workspace Flow discovery/provision/send on existing WhatsApp client |
| `crates/api/src/whatsapp_templates.rs` | WABA template discovery from installed `send_whatsapp` actions |
| `crates/domain/src/catalog.rs` | Thin entity `catalog:` overlay parse + NormalizedCatalogItem mapping |
| `crates/api/src/catalog_service.rs` | Generic publish/sync/unpublish + bus consumer (no app branches) |
| `crates/api/src/whatsapp_catalog.rs` | WhatsApp/Meta Commerce adapter only |
| `crates/db/src/channel_catalog.rs` | Workspace-scoped external catalog item ids |
