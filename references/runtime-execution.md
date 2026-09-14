# Runtime Execution

## FlowRunner

**File:** `ai-customer-support/crates/api/src/flow_engine/executor.rs`

The FlowRunner is a step-runner state machine. It loads a flow definition from Postgres, resolves the starting step, and iterates through steps in a loop bounded by `MAX_STEPS_PER_TURN = 50`.

### Entry point

```rust
pub async fn run(&self, exec: FlowExecution, trigger: Trigger) -> Result<Option<FlowTurnReply>, CoreError>
```

### Trigger enum

```rust
pub enum Trigger {
    Start,                      // Begin at step 0
    Message(String),            // Satisfy ask/upload waits
    Wake,                       // Advance to next step
    Approval,                   // Advance after human approval
    Retry,                      // Re-enter same step
    ChallengeOutput(Value),     // Store tool output into variables
}
```

### StepOutcome enum

```rust
pub enum StepOutcome {
    Advance(Option<String>),    // Move to next (or named) step
    Goto(String),               // Jump to specific step
    WaitInput,                  // Pause for user message
    WaitUpload,                 // Pause for file upload
    WaitApproval,               // Pause for human approval
    WaitChallenge,              // Pause for identity challenge
    Pause(DateTime<Utc>),       // Delay until time
    Complete,                   // Flow finished
    Fail(String),               // Flow failed
}
```

### FlowTurnReply

```rust
pub struct FlowTurnReply {
    pub text: String,
    pub suggested_actions: Vec<ActionItem>,
}
```

### Orchestration entry

**File:** `ai-customer-support/crates/api/src/flow_engine/mod.rs`

- `try_advance_or_start()` -- the chat loop: if active execution exists, feeds message into it; otherwise selects new flow via catalog (cosine similarity >= 0.82 threshold)
- `start_flow_from_facade()` -- handles conversation uniqueness, creates FlowExecution rows, seeds variables from durable conversation memory
- One-active-flow enforced by partial unique index (see C5 contract below)
- Redis active-execution cache with 30-day TTL

## ExecutionAdapter

**File:** `ai-customer-support/crates/api/src/flow_engine/adapter.rs`

Routes tool invocations to the correct backend:

```rust
pub enum ExecutionBackend {
    Sdk,                        // External ERP/POS/CRM tools
    Runtime { solution: String }, // Marketplace metadata apps
}

pub enum AdapterOutcome {
    Success { output: Value, media_fields: Option<Vec<String>> },
    Challenge { prompt: String, content: Value },
    Failed { message: String },
}
```

Security gates:
- `deny_sdk_runtime_entity()` -- blocks SDK from calling marketplace entity CRUD
- `deny_sdk_http_runtime()` -- blocks SDK from calling HTTP tools

## RuntimeAdapter (entity.* execution)

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs` (~1900 lines)

### Entity capability parsing

Handles two formats:
- `entity.{name}.{op}` -- dotted form
- `entity_{name}_{op}` -- underscore form

Operations: `create|insert`, `list|find`, `get`, `update`, `delete|cancel`

### execute_storage() -- unified CRUD

**Create:**
1. Resolve workspace, installation, solution from trusted context (never parameters)
2. `strip_authority_overrides()` -- removes workspace_id, tenant_id, installation_id, organization_id, customer_id, person_id from caller parameters
3. RBAC authorization via `require_entity_permission()`
4. Check entity metadata for person-scoping (C1: `scope: customer` + `type: person` field)
5. `build_create_body()` -- injects person_id for customer-scoped entities; fails closed if identity missing; auto-fills customer name fields from Person record; auto-fills datetime fields from date+time; auto-resolves relation fields

**List:**
1. `build_list_body()` -- injects person_id filter for customer-scoped entities
2. Coerce limit/skip to positive integers (C2 contract)
3. Wrap filter values in case-insensitive regex

**C2 contract -- typed parameter coercion:**

The adapter performs **deliberately minimal** type coercion. Only `limit` and `skip` on list operations are coerced via `coerce_positive_int()`:

```rust
fn coerce_positive_int(v: &Value) -> Option<i64> {
    match v {
        Value::Number(n) => n.as_i64(),
        Value::String(s) => s.trim().parse::<i64>().ok(),
        _ => None,
    }
    .filter(|&n| n > 0)
}
```

- Accepts: integers, numeric strings (whitespace-trimmed)
- Rejects: floats (even `10.0`), zero, negatives, non-numeric, booleans
- On failure: silent fallback (limit defaults to 50, skip omitted)

**No general field-type-driven coercion exists.** Entity document field values (the business data the LLM provides for create/update) pass through as raw JSON to the storage service. The adapter performs structural transformations (person-id injection, datetime synthesis, relation resolution, authority stripping) but does NOT inspect declared field types (integer, float, boolean) to coerce values.

**Get:**
1. Resolve human-readable references to UUIDs via `resolve_reference_to_uuid()`
2. Support expand (with trust boundary)

**Update:**
1. Resolve reference to UUID
2. Build patch with auto-filled datetime and relation fields

**Delete:**
1. Resolve reference to UUID
2. Build delete

### resolve_reference_to_uuid()

- Valid UUID -> use directly
- Non-UUID -> search by code field
- Exactly one match -> return UUID
- Zero matches -> None
- Multiple matches -> ambiguity error

## choices_from implementation

**File:** `ai-customer-support/crates/api/src/flow_engine/steps.rs`

`choices_from_context_var()`:
1. Reads flow variable expected to contain `{ items: [...] }`
2. Applies `title_field` / `value_field` / `title_fields` to extract display text and tap values
3. Applies `choices_filter` to narrow items
4. Applies `chip_prefix` for prefix:value chip formatting
5. Limits to 10 items maximum
6. Deduplicates values
7. Truncates titles longer than 24 characters

During message matching (`apply_trigger`):
- `match_step_choices()` handles tap targets (exact value matches)
- `normalize_ask_reply()` handles relative dates, time parsing, prefix:value chip matching

## Expand trust boundary (C3 contract)

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs`

### Two distinct concepts

| Concept | What it is | Who controls it |
|---------|-----------|----------------|
| **Relation field reference** | Setting a foreign key value (e.g., `{"traveler": "some_uuid"}`) in create/update | LLM can supply (after authority stripping) |
| **Expand directive** | Execution control telling storage to dereference/join a relation and return related data inline | Flow constants ONLY |

### The trust rule

**Source authority matters, not content correctness.** Even a perfectly correct expand from the LLM is silently discarded.

`apply_expand_trust_boundary()`:
1. `strip_authority_overrides()` removes forbidden keys from parameters
2. `doc.remove("expand")` -- unconditionally strips ANY expand from untrusted parameters
3. If `trusted_expand` exists (from `step.config.constants.expand`):
   - Validates each key is a declared `type: relation` field
   - Validates each value matches the field's declared `ref_entity`
   - Fail-closed: rejects entire expand if entity metadata unavailable
4. Returns `(cleaned_document, validated_expand_map, optional_error)`

### Extraction path

Expand has its own dedicated extraction from `step.config.constants.expand` in `steps.rs`, bypassing the general `input_map` mechanism. This is a deliberate architectural choice -- expand is treated as a first-class security concern.

### Validation rules (fail-closed)

- Must be a JSON object
- Empty expand `{}` accepted even on entities with no relations
- Each key must be a field declared with `type: relation` in entity metadata
- Each value must be a non-empty string matching the field's `ref_entity`
- Any single invalid entry rejects the entire expand map
- If entity metadata is unavailable, operation fails entirely

### Where validated expand is applied

Only read operations receive expand:
- **LIST**: injected into `build_list_body()`
- **GET**: injected into request body
- **CREATE/UPDATE/DELETE**: no expand

## Flow execution lifecycle and routing (C5 contract)

### One-active-flow-per-conversation

The platform enforces a strict one-active-flow model. **Flows cannot be superseded or replaced.** A new flow starts only when no active execution exists.

**Database constraint** (partial unique index):
```sql
CREATE UNIQUE INDEX uq_flow_executions_active_conversation
    ON flow_executions (conversation_id)
    WHERE status NOT IN ('completed', 'failed', 'cancelled');
```

### Flow execution states

```rust
pub enum FlowExecutionStatus {
    Pending, Running,                        // actively executing
    WaitingForInput, WaitingForUpload,       // suspended pending user action
    WaitingForApproval, WaitingForChallenge, // suspended pending external event
    Paused,                                  // suspended (delay)
    Completed, Failed, Cancelled,            // terminal states
}
```

- **Terminal**: `Completed | Failed | Cancelled` -- no further transitions
- **Waiting**: `WaitingForInput | WaitingForUpload | WaitingForApproval | WaitingForChallenge | Paused`
- **Active**: anything non-terminal (blocks new flows)

### Routing decision: `try_advance_or_start()`

**File:** `ai-customer-support/crates/api/src/flow_engine/mod.rs`

1. **If active execution exists**: feed message into existing flow (no new flow starts)
   - Exception: `WaitingForChallenge` returns to normal LLM path
2. **If no active execution**: attempt flow selection via catalog
   - Cosine similarity scoring against flow intent embeddings
   - `FLOW_SELECT_MIN_SCORE = 0.82` (conservative threshold)
   - Below threshold: no flow selected, turn falls through to LLM/tool path
3. **Safety net**: `create_execution` catches unique-index violation from concurrent turns, defers to LLM path

### Key implication for Marketplace Apps

A flow that is `WaitingForInput` will continue to receive all messages on that conversation until it reaches a terminal state. New flows cannot interrupt or replace it. The 0.82 threshold governs whether a new flow starts at all (after the current one completes), not whether it replaces an existing one.

## input_map resolution

**File:** `ai-customer-support/crates/api/src/flow_engine/variables.rs`

`build_tool_input()` constructs JSON input for tool steps:

**With input_map:**
- `$constants.` prefix -- reads from flow-level constants (trusted, not user-influenced)
- `$literal:` prefix -- produces explicit literal value (everything after prefix is verbatim)
- Dotted keys (e.g. `filter.status`) nest into sub-objects
- `{{variable}}` references resolve from conversation context

**Without input_map:**
- Entire collected variables object passed through as-is

## Trigger matching

### Conversation trigger (intent selection)

**File:** `ai-customer-support/crates/api/src/flow_engine/catalog.rs`

- `select()` embeds all cataloged flows, scores via cosine similarity against user message embedding
- Picks winner above `FLOW_SELECT_MIN_SCORE` threshold

### Event trigger

**File:** `ai-customer-support/crates/api/src/flow_engine/dispatch.rs`

```rust
pub enum FlowTrigger {
    Conversation,
    Event { event: String, when: Option<String> },
    Schedule { cron: String },
    Webhook { name: Option<String>, when: Option<String> },
}
```

`trigger_matches()` uses bidirectional event aliases (e.g. `contact.created` <-> `person.created`).
`eval_trigger_when()` evaluates optional CEL-like predicates over event payloads.

## Full execution pipeline

```
User message
  -> try_advance_or_start()
    -> Check Redis for active execution
    -> If active: FlowRunner.run() with Trigger::Message
    -> If not: catalog.select() (cosine similarity)
      -> start_flow_from_facade() -> create FlowExecution
      -> FlowRunner.run() with Trigger::Start
        -> apply_trigger() -> resolve starting step
        -> Step loop (max 50):
          -> enter_step() dispatches by step_type
          -> For "tool" steps:
            -> build_tool_input() -> invoke_capability()
              -> Runtime: RuntimeAdapter::execute()
                -> execute_storage() for entity CRUD
                -> execute_http() for generic HTTP
              -> SDK: SdkAdapter::execute()
          -> StepOutcome determines next action
          -> Terminal: wait/complete/fail -> persist Postgres -> refresh Redis
```
