# Agent Integration

## AI execution hierarchy

```
Deterministic -> Fast Router -> FlowRunner -> Agent
```

Agent is the most capable tier, used for open-ended reasoning when lower tiers cannot handle the request.

## Agent is NOT the universal dispatcher

- Marketplace Apps must NOT create Agent-specific metadata just to make the Agent work
- Agent consumes capabilities already exposed by the existing Marketplace App/runtime
- The Agent layer reads entity metadata and projects tools automatically

## Entity capability projection

**File:** `ai-customer-support/crates/api/src/agent/entity_capabilities.rs`

### Pipeline

1. **Parse entity definitions** from Marketplace manifest
2. **Generate LLM tool definitions** -- exactly 2 tools per entity:
   - `entity_{id}_list` -- with filter params from non-authority fields
   - `entity_{id}_get` -- by code/id
3. **Runtime argument transformation** -- transforms flat LLM args into nested format:
   ```
   {"city": "Chennai", "limit": 10}
   -> {"filter": {"city": "Chennai"}, "limit": 10}
   ```
4. **Execution** through `RuntimeAdapter::execute()`

### Authority field exclusion

`AUTHORITY_FIELDS` constant: `tenant_id`, `organization_id`, `workspace_id`, `installation_id`, `person_id`, `customer_id`, `created_by`, `updated_by`

These are excluded from LLM tool definitions -- the LLM never sees them as valid parameters.

### Field type mapping

| Entity type | JSON Schema type |
|-------------|-----------------|
| string | string |
| integer | integer |
| number/float | number |
| boolean | boolean |
| enum | string with enum values |
| date | string |
| person | string (person_id) |
| relation | string (reference) |
| image | string (URL) |

## Read-only vs mutation handling

**File:** `ai-customer-support/crates/api/src/agent/capabilities.rs`

`is_confidently_read_only(tool)` checks:
- Name prefixes: `get_`, `list_`, `search_`, `fetch_`, `find_`, `check_`, etc.
- Name suffixes: `_status`, `_detail`, `_history`, `_info`, `_summary`
- Write markers: `create_`, `update_`, `delete_`, `remove_`, `add_`, `edit_`, etc.
- HTTP method: REST/Runtime tools must be GET

**Only read-only tools and stageable mutation tools are advertised to the Agent.**

## Tool advertisement pipeline

**File:** `ai-customer-support/crates/api/src/agent/runtime.rs`

`advertise_tools()` computes effective tool set:

```
Effective = Role x Channel x RBAC x Capability x (ReadOnly | Stageable)
```

Checks (all must pass):
1. `tool_callable_from_chat` -- channel + identity policy
2. `is_confidently_read_only` OR `is_stageable_tool` (with staging enabled)
3. `core.allows_tool_def` -- capability flags + gates
4. `role_allows_tool` -- customer vs business role restrictions
5. `rbac_allows_tool` -- installation-scoped permissions

## Agent capability flags

**File:** `ai-customer-support/crates/api/src/agent/capabilities.rs`

```rust
pub struct AgentCapabilityFlags {
    pub analytics: bool,
    pub promotions: bool,
    pub pricing: bool,
    pub refunds: bool,
    pub orders: bool,
    pub reservations: bool,
    pub inventory: bool,
    pub customers: bool,
    pub knowledge: bool,
}
```

`allows_tool()` uses heuristic name matching:
- "refund"/"charge"/"payment" -> refunds flag
- "order" -> orders flag
- "customer"/"person"/"lead" -> customers flag
- "property"/"listing"/"viewing" -> customers+orders flag (domain-agnostic mapping)

## Marketplace metadata parsing

**File:** `ai-customer-support/crates/api/src/agent/metadata.rs`

```rust
pub struct AgentManifestMetadata {
    pub enabled: bool,
    pub roles: HashMap<AgentRole, AgentManifestRoleCaps>,
}
```

- `enabled=false` -> defaults
- `enabled=true` + empty caps -> all false (fail closed)
- `enabled=true` + caps -> `flags_from_capability_names()`
- Domain-agnostic: "properties"/"listings"/"leads"/"viewings" -> customers+orders

## Complexity routing

**File:** `ai-customer-support/crates/api/src/agent/router.rs`

```rust
pub enum ComplexityDecision {
    ExistingPath,     // Fast Router / LLM / Flow
    AgentCandidate,   // Agent (when flags allow)
}
```

- Mutations (cancel/refund/campaign/delete) -> Agent
- Simple direct lookups -> existing path
- Knowledge/FAQ -> existing path (RAG)
- Investigative ("why did", "recommend", "analyze") -> Agent

## Conversational approval

**File:** `ai-customer-support/crates/api/src/agent/conversational_approval.rs`

Affirmative: yes, y, yeah, yep, approve, confirm, proceed, go ahead, do it, ok, okay, sure, /approve
Negative: no, n, nope, reject, cancel, don't, stop, never mind, /reject

Routes through same `approve_pending_action()` / `reject_pending_action()` as REST/Mobile.

## Provenance tracking

**File:** `ai-customer-support/crates/api/src/agent/provenance.rs`

```rust
pub struct AgentTurnProvenance {
    pub turn_id: Uuid,
    pub role: String,
    pub tenant_id: Uuid,
    pub workspace_id: Option<Uuid>,
    pub installation_id: Option<Uuid>,
    pub conversation_id: Option<Uuid>,
    pub tool_calls: Vec<ToolCallProvenance>,
    pub rag: Option<RagProvenance>,
    pub observed_entities: Vec<String>,
}
```

Records every tool call with sanitized arguments, action class, success, entity refs, latency.

## Rules for Marketplace Apps and Agent

1. Do NOT create `agent:` metadata merely to support Agent behavior
2. Agent projects tools from entity metadata automatically
3. Only read-only entity tools are exposed by default (list + get)
4. Mutation tools require staging + human approval
5. Agent capability flags are domain-agnostic -- no vertical-specific branching
6. Marketplace metadata declares what Agent CAN do; RBAC enforces what it IS ALLOWED to do
