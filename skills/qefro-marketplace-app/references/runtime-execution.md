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

## WhatsApp Flow (Meta surface, same FlowRunner)

**Files:** `crates/domain/src/whatsapp_flow.rs`, `crates/api/src/whatsapp_flows.rs`

`compile_whatsapp_flow_json(workflow_id, definition)` reads `definition.whatsapp_flow` plus existing `ask` / `tool` / `complete` steps and emits Meta Flow JSON (data_api_version 3.0). Dynamic screens that use `choices_from` call the existing Data Endpoint, then `complete_nfm_reply` starts the **same** FlowRunner with collected fields.

This is not a second executor. Discovery is generic over installed `workflow_definitions`. Do not add `if solution ==` or a second metadata namespace.

## WhatsApp Catalog (Meta commerce, same Catalog Service)

**Files:** `crates/domain/src/catalog.rs`, `crates/api/src/catalog_service.rs`, `crates/api/src/whatsapp_catalog.rs`, `crates/db/src/channel_catalog.rs`

`map_record_to_catalog_item(entity, overlay, record)` reads `definition.catalog` plus the stored record and emits a `NormalizedCatalogItem`. The WhatsApp adapter translates that into Meta Commerce Product Item fields. FlowRunner never calls Meta catalog APIs. Marketplace YAML never contains WABA/catalog ids or tokens.

This is not a second catalog vs `flow_engine/catalog.rs` (workflow selector). Bus dispatch is `catalog_service::handle_event` after CRM/goals. Do not add `if entity == "property"` or a second metadata namespace.

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

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs`

### Entity capability parsing

`parse_entity_capability()` handles two formats and splits on the **last** separator (`rsplit_once`), so an entity id may contain `_` (`entity.order_item.create` → entity `order_item`, op `create`):

- `entity.{name}.{op}` -- dotted form
- `entity_{name}_{op}` -- underscore form

| Op alias | Normalised | Notes |
|----------|-----------|-------|
| `create`, `insert` | `create` | runs defaults → `prepare_document(require_all: true)` → version stamp → code allocation |
| `list`, `find` | `list` | |
| `get` | `get` | |
| `update` | `update` | `prepare_document(require_all: false)` |
| `delete`, `cancel` | `delete` | honours `delete.mode` (`hard`/`soft`/`restrict`) |
| `restore` | `restore` | `delete.mode: soft` only |
| `availability`, `dates`, `slots` | `availability` | the runtime dispatches further on the `kind` param |
| `aggregate` | `aggregate` | read-only `op: count|sum` |

Anything else returns `None`, which the caller reports as `unsupported entity op '<op>'` — a tool step naming `entity.order.mark_paid` fails at execution, not at publish.

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

**No coercion at the adapter boundary — but validation happens one step later.** `build_create_body()` / `build_patch_body()` perform structural transformations only (person-id injection, datetime synthesis, relation resolution, authority stripping) and pass field values through as raw JSON. The declared field types are enforced afterwards, in the entity-op dispatcher, by `entity_validate::prepare_document(entity_def, doc, require_all)`:

- `create` runs it with `require_all = true`, **after** `apply_create_defaults_at()` and before `stamp_create_version()` / `attach_create_allocate_code()`.
- `update` runs it with `require_all = false`, so a partial patch is never re-checked for completeness.
- `enum` is matched case-insensitively against `enum_values` and stored as the declared spelling; `integer` rejects a fractional value; `float`/`number`, `boolean` (accepting `true/yes/1/y` and `false/no/0/n`) and `date` (via `normalize_date_string`, which understands `today`/`tomorrow`) are coerced.
- `person`, `image`, `media` and `json` values pass through **uncoerced**, and authority-named keys are skipped.

The practical consequence: an LLM that supplies `"quantity": "3"` or `"confirmed": "yes"` reaches storage as a number and a boolean, and a bad enum or a non-integer `integer` fails the tool step with a message instead of writing. Do not pre-coerce in metadata or in a flow, and do not assume a value was stored as sent — assert the persisted document in tests.

**Booking routing:** when the entity declares `availability:`, `create` dispatches to `storage.insert_booking` under `capacity.mode: exclusive` (atomic conflict detection in storage), plain `storage.insert` under `unlimited`, and fails loudly under `limited`. This is the only place the capacity mode changes the storage call.

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
1. Reads flow variable — `{ items: [...] }` or a bare array
2. Applies `title_field` / `value_field` / `title_fields` to extract display text and tap values
3. Applies `choices_filter` to narrow items
4. Applies `chip_prefix` for prefix:value chip formatting
5. Limits to 10 items maximum
6. Deduplicates values
7. Truncates titles longer than 24 characters

Two details that change flow behaviour:

- The `empty_message` / fail gate is a **separate** function, `choice_list_items()`, and it only recognises an object with an `items` array. A variable holding a bare empty array therefore skips the gate and yields a chip-less `ask` instead of the graceful `empty_message` completion. Bind `choices_from` to a tool `output:` (which is `{ items: […] }`) when the empty case must be handled.
- Items whose resolved title is not a **non-empty string** are dropped, not rendered as `0`/`false`. Numeric-only lists produce no chips.

During message matching (`apply_trigger`):
- `match_step_choices()` handles tap targets (exact value matches)
- `normalize_ask_reply()` handles relative dates, time parsing, prefix:value chip matching
- A `choices_from` answer is then resolved back to its item in `executor.rs`: `<field>` is overwritten with the item's canonical `value_field`/`id`/`_id`, `<title_field>` with the title, and `<field>_code` with the item's `code` when present. A reply matching nothing on a non-empty list sets `_validation_error`, un-marks the field as collected and **re-asks** the same step. This is why a flow can ask by title and still create by UUID.

## Availability and booking validation

**Files:** `ai-customer-support/crates/api/src/flow_engine/availability.rs`, `runtime_adapter.rs`

Entity `availability:` is flattened by `extract_availability_config()` (structural: `booking.datetime_field`, `capacity.mode`, `capacity.resource.field`) and then overlaid with workspace install settings by `overlay_workspace_settings()` (operating: timezone, duration, working hours, closed days, date window, blocking statuses — accepted flat or nested under `availability:`, each with several aliases, comma-separated strings allowed). An invalid overlay value is skipped rather than rejected, so a mistyped setting silently keeps the entity-side default.

`entity.<id>.availability` dispatches on `kind` (`dates` | `slots` | `validate`, with `times`/`time`, `check`, `date` aliases and inference from supplied params), and `require_valid_config()` fails closed on `limited` capacity, a bad timezone, bad `HH:MM` hours, non-positive duration/window, or an empty datetime field. Conflicting bookings are loaded only for `slots` and `validate`, and only under `exclusive`.

After an ask step on the availability datetime, `apply_availability_reply()` writes `date`, `time` and `booking.datetime_field` (UTC) into flow variables; an invalid reply sets `_validation_error` and re-asks. See **Availability declaration (structural capability)** in `metadata-contract.md` for the metadata half and the reference flows.

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

**With input_map** — every value is a **bare dotted path or a `$`-prefixed source**; there is no template engine here:

- `$constants.<path>` — reads from flow-level `constants:` (trusted, not user-influenced)
- `$literal:<raw>` — `parse_flow_literal` tries `serde_json` first, so `$literal:20` is the number `20` and `$literal:scheduled` is the string `"scheduled"` (unparseable input stays a verbatim string)
- `<dotted.path>` — `ctx.get_path(path)` over flow variables (collected answers, prior `output:` documents)
- Dotted **keys** (`filter.status`, `filter.capacity.$gte`) nest into sub-objects via `set_dotted`

A `{{variable}}` value does **not** resolve — it is looked up as a path segment literally named `{{variable}}`, misses, and is then **dropped silently** (`if let Some(v) = …`). Unresolved sources never become `null`, so a typo removes a parameter rather than erroring. `{{…}}` substitution belongs to `message`/`prompt`/`complete` text, which goes through `FlowVariableContext::resolve_template`.

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
`eval_trigger_when()` evaluates a **single** optional predicate over the event payload: `<path> <|>|<=|==|!=|>|< <literal>` with an optional `payload.` / `event.payload.` prefix, or a bare path for truthiness. It is not CEL — there are no boolean operators, parentheses, or functions — and an empty expression matches while a missing left-hand path fails closed, which silently skips the flow.

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
