# Metadata Contract

**Qefro Event-Driven Runtime v1 frozen.**

Architecture: Marketplace Metadata → Generic Runtime → Entity mutation → Business Event → CRM Automation | Goal Engine | Event → FlowRunner. No Agent v2 / Goal v2 / FlowRunner v2 / EventBus v2 / Scheduler v2 / Availability v2 unless a real product/customer gap appears.

## Manifest schema

```yaml
id: <app-id>                    # kebab-case, e.g. "clinic-pro"
name: <display name>
version: <semver>               # e.g. "1.0.0"
hosting: runtime                # ALWAYS "runtime" -- no exceptions
description: <string>
category: <category>            # commerce|healthcare|hospitality|real-estate|scheduling|collaboration|support|productivity|education|logistics
tags: [<strings>]
channels: [widget, whatsapp]    # supported channels

# Entity-native apps:
entities: [<entity-ids>]
flows: [<flow-ids>]
events: [<dot-notation names>]  # e.g. ["appointment.created", "appointment.cancelled"]
permissions: [workflow.execute, storage.read, storage.write, storage.update, storage.delete]
capabilities: [theme.get, user.get, tenant.get, runtime.query, workflow.trigger, storage.read, storage.write]

# HTTP-backed apps:
connections: [<connection-ids>]
http_tools: [<tool-ids>]

# NLU intent matching:
triggers:
  - id: <trigger_id>
    workflow: <flow-id>
    input_variable: <variable_name>
    match:
      intents: [<natural language phrases>]
      confirmation:
        reply_signals: [<response signals>]
        required_slots: [<slot-ids>]
    confirmation_message: "<template with {{vars}}>"

# Domain-agnostic slot extraction:
conversation_slots:
  - id: <slot_id>
    labels: [<synonyms>]
    kind: <slot_kind>
    identity: person_name       # optional -- marks identity-relevant slot
    chip_prefix: <prefix>       # optional -- UI chip label

ui:
  name: <display name>
```

## Entity schema

```yaml
id: <entity_id>
scope: customer                 # optional -- present when customer-facing
name: <display name>
description: <string>

# Optional auto-incrementing code:
allocate_code:
  prefix: <2-4 chars>           # e.g. "ORD", "PAT", "APT"
  start: 1001

fields:
  - name: <field_name>
    type: <field_type>
    required: true|false
    description: <optional>
    enum_values: [<values>]     # required when type: enum
    ref_entity: <entity_id>     # required when type: person or relation
    unique: true|false          # optional — CRUD create/update + CSV importer enforce uniqueness
                                # per tenant/workspace. Absence means always-create.
    import_synonyms: [<headers>] # optional — extra CSV header names for this field
    import_default: <value>      # optional — filled when CSV omits the column
    default: <value>             # optional — filled on create when empty
    default_from_setting: <key>  # optional — install setting used as create default

# Optional status-to-event mapping. On create, `{entity}.created` is always
# emitted; if the new record has a status with a mapping, that event is also
# emitted (deduped when it equals `{entity}.created`). On update, status
# events fire only when the status field actually changes.
status_events:
  <status_value>: <event.name>

# Optional deterministic computed fields (no LLM, no tenant code):
computed:
  - field: <field_name>
    op: relation_sum|relation_count|add|subtract|multiply|status_from_number|status_from_date
    # relation_sum / relation_count:
    relation: <child_entity_id>
    foreign_key: <field on child pointing at this record>
    sum_field: <numeric child field>          # relation_sum only
    # add / subtract / multiply:
    left: <field>
    right: <field>
    minus: <field>                            # optional, multiply only
    plus: <field>                             # optional, multiply only
    # status_from_number:
    source: <numeric field>
    preserve: [<status values that must not be overwritten>]
    zero: <status when source == 0>
    positive: <status when source > 0>
    partial:
      field: <numeric field>
      status: <status when partial.field > 0 and source > 0>
    # When `source` is a subtract result (typically outstanding = total − paid),
    # the zero/paid status is NOT applied if the minuend (total) is also 0.
    # Empty documents stay on their current/default status. paid requires total > 0.
    # outstanding > 0 → positive/partial; outstanding == 0 && total > 0 → zero/paid.
    # status_from_date:
    date_field: <date|datetime field>
    past: <status when date is before today>
    when_number_gt: <numeric field>           # optional gate
    preserve: [<statuses>]

# Relation rollup policy:
# - On the parent’s own create/update, an empty relation_sum does not overwrite
#   an explicit total (children may not exist yet).
# - add / subtract / multiply skip when the target already has an explicit
#   numeric value AND an operand was a skipped empty relation_sum. Empty
#   targets still compute (e.g. outstanding = total − 0 when no allocations).
# - When a child mutates, parent relation_sum/count rewrite including 0 so
#   deleting the last child zeros inventory/order/invoice totals.

# Optional child-vs-parent numeric cap (domain-agnostic). On child create/update,
# sum of sibling `field` values (active rows only, same relation parent) plus
# this row must be ≤ parent.`parent_field`. Workspace-scoped storage; the
# relation must be a trusted `type: relation` field (no arbitrary SQL).
# Consumers: payment allocations vs receipt amount, order payments, stock splits.
relation_caps:
  - field: <numeric field on this child>
    relation: <FK field on this child>
    parent_field: <numeric field on the related parent>

# Optional event → related-record update (bounded, equality filters only).
# Use `event:` not `on:` — YAML 1.1 treats `on` as a boolean.
# After each related write, date watches on the target are resynced (cancel
# when status no longer matches).
on_events:
  - event: <event.name>
    target: <entity_id>
    match_field: <field on target>
    match_from: id|<field on the triggering record>
    where:
      <field>: <equality value>
    set:
      <field>: <literal value>                # never authority keys
    limit: 50                                 # max 50

# Optional date-field automation trigger (workspace timezone, idempotent).
# Honors entity `status_field` (default `status`). At fire time the live
# record is re-fetched; deleted or status-mismatched watches do not emit.
date_events:
  - field: <date|datetime field>
    event: <event.name>
    when_status: [<status values>]            # optional; omit to always watch

# Create/update (and CSV) also enforce:
#   required: true     — rejected on create after defaults (person/image/authority skipped)
#   enum_values        — rejected if not in the list (case-insensitive, stored as declared)
#   unique: true       — advisory lock on one Postgres session is held through the storage write
#                      (UniqueWriteLock unlocks on success, error, storage failure, and Drop)
#   integer/float/boolean/date — coerced on CRUD (same idea as CSV)

# List accepts `sort` (object or field name). Authority keys stripped; max 3 fields.
# Filter objects such as `{ due_date: { $gte: "2026-01-01" } }` are passed through;
# string equality filters are still case-insensitive exact regex.
# `entity.{name}.aggregate` is person-scoped on customer channels (same as list).

# CSV commit stamps version=1, skips soft-deleted rows on unique lookup, then
# replays generic post-create hooks (computed, events, on_events, date watches)
# for created ids (bounded). It is not a second import engine.

# Optional CSV import helpers on a field:
#   unique: true           — importer AND CRUD create/update enforce uniqueness
#   import_synonyms: []    — extra header names for the generic mapper
#   import_default: <val>  — filled when the CSV omits the column
#   default: <val>         — filled on create when the field is empty
#   default_from_setting: <settings key>  — same, from install settings

# Optional delete policy (default `hard` — current storage.delete behavior).
# No cascade. Related records are never auto-deleted.
delete:
  mode: hard|soft|restrict
# `delete_policy: soft` is accepted as an alias for `delete.mode`.
# hard     — storage.delete (legacy default)
# soft     — patch platform `deleted_at` (RFC3339); hidden from list/get/aggregate;
#            restore via `entity.{name}.restore` (requires update permission).
#            Soft does NOT refuse related records (that is restrict only).
# restrict — refuse delete when metadata relations or parent computed FKs still
#            point at this record (active rows only). Error names the child entity id.

# restore (`entity.{name}.restore`, delete.mode=soft only):
#   1. Load the soft-deleted document (get-by-id).
#   2. UniqueWriteLock on the same unique fields as create/update (plus record id).
#   3. Uniqueness vs ACTIVE records excluding this id; conflict fails the restore.
#   4. Clear `deleted_at`, increment `version`, write audit, then computed/events/watches.
# Get-by-id of a soft-deleted row is not-found. List/aggregate send `{ deleted_at: null }`
# to storage.find before limit/skip (Mongo null-or-absent). `total` is the storage count.

# Internal computed / on_events patches are trusted platform writes after the user
# mutation has already passed uniqueness and concurrency. They do not re-enter
# RuntimeAdapter and do not require a caller version. Parent relation rollups
# take UniqueWriteLock on the parent record id, re-read children under that lock,
# and increment platform `version` on the parent write. This is not a second
# mutation path and is not a Billing lock. on_events related writes remain
# trusted patches without a caller version.

# Optional optimistic concurrency (opt-in). Platform `version` integer is always
# stamped on create (1) and incremented on update. Callers cannot set version.
concurrency: optimistic          # alias: optimistic: true
# When enabled, update MUST send the current `version` (or `expected_version`).
# Omit or mismatch → VERSION_CONFLICT (HTTP 409). Legacy callers that omit
# version keep working unless this key is set — the runtime does not silently
# last-write-wins on opted-in entities.
# Serialization: a per-record Postgres advisory lock is held through the HTTP
# storage write. RuntimeAdapter → storage-service is NOT a distributed
# transaction; version increment and the document write share the lock, not a
# single DB commit. Audit insert is Postgres after a successful storage write
# (best-effort; a crash can leave a mutation without an audit row).

# Mutation pipeline (Portal / Mobile / Agent / CSV hydrate / FlowRunner / automation):
# authorization → metadata validation → uniqueness/concurrency → storage write
# → audit (append-only `entity_mutation_audit`) → computed/rollup → events
# → on_events → date watches.
# Actor identity is taken from the authenticated runtime context only.
# Audit source: portal | mobile | agent | flow | csv | automation | api
# (channel mapping: Portal→portal, WhatsApp/Widget→flow, Api→api; CSV hydrate
# writes csv; on_events related writes write automation).

# Platform authority fields (never accepted from caller/model input):
# workspace_id, tenant_id, installation_id, organization_id, customer_id,
# deleted_at, version, expected_version.
# HTTP/Portal/Agent/FlowRunner may send `version` / `expected_version` for
# optimistic checks; they are stripped from the stored document and must not
# overwrite the stamped version on the response.
#
# person_id is a scoped Hub relation, not a spoofable authority key.
# LLM/agent payloads still strip it. Staff/API may select an existing Person
# only after Hub lookup proves the id exists in THIS tenant+workspace.
# Customer channels inject session identity (self only). Event-triggered
# create may copy it via inherit_from. CSV resolves via PersonService.
# Cross-workspace, nonexistent, and nil ids are rejected. Tenant/workspace
# spoofing remains impossible.
```

### Canonical field types

| Type | Purpose | Notes |
|------|---------|-------|
| `string` | General text | |
| `integer` | Whole numbers | |
| `float` | Decimal numbers | |
| `boolean` | True/false | |
| `date` | Date only | Auto-filled from date+time conversation slots |
| `datetime` | Date and time | |
| `email` | Email addresses | |
| `phone` | Phone numbers | |
| `url` | URLs | |
| `uuid` | UUIDs | |
| `enum` | Enumeration | Requires `enum_values` |
| `json` | Arbitrary JSON | |
| `person` | Hub identity link | Always has `ref_entity: person` |
| `relation` | Cross-entity reference | Has `ref_entity: <target>` |
| `image` | Image URL | |

### Person field contract

Entities with `scope: customer` MUST include a `person_id` field:
```yaml
- name: person_id
  type: person
  ref_entity: person
  required: true
```

`person_id` is a **scoped business relation** to Customer Hub, not caller-controlled
authority (`tenant_id` / `workspace_id` / `installation_id` / `user_id`).

How it is bound:

- **Customer channels** (WhatsApp, Widget): runtime injects `person_id` from the
  authenticated Hub session (self only). Caller/LLM values are ignored.
- **Portal / staff API**: caller MAY select an existing Hub Person. Runtime
  accepts the id only after storage/Hub lookup proves the Person exists in
  THIS tenant+workspace. Cross-workspace, nonexistent, and arbitrary ids are
  rejected. Tenant spoofing remains impossible.
- **CSV**: PersonService resolves identity from phone/email (never a CSV
  `person_id` column).
- **Agent / LLM**: tool schemas omit `person_id`; `strip_authority_overrides`
  still drops it. Unvalidated ids are not stored.
- **Event-triggered flows**: `inherit_from` copies Person from a related
  workspace record after authority strip. Payload cannot supply it.

Optional `inherit_from` on a `type: person` field (event-triggered flows only):

```yaml
- name: person_id
  type: person
  ref_entity: person
  inherit_from: appointment_id   # relation field on this entity
```

When `ToolAuthContext.causation_event_id` is set, RuntimeAdapter copies `person_id` from the related record loaded via workspace-scoped storage.get. Payload, LLM, and caller parameters still cannot supply `person_id` on this path (authority strip runs first; staff bind is skipped when `causation_event_id` is set). Use this when a follow-up, visit, or job is created from an invoice, appointment, or work order that already has a trusted Hub Person.

### Relation field contract

```yaml
- name: property_id
  type: relation
  ref_entity: property
  required: false
```

The runtime resolves human-readable references (codes like "APT-1001") to UUIDs via `resolve_reference_to_uuid()`. Non-UUID values are searched against the entity's code field.

## Flow schema

```yaml
id: <flow-id>
name: <display name>
description: <string>
surfaces: [customer|staff]      # optional -- access control
enabled: true                   # optional; false skips conversation + event dispatch

trigger:
  type: conversation            # or: event, schedule, webhook
  # event: <opaque event name>  # required when type: event (e.g. invoice.overdue)
  # when: <payload predicate>   # optional; fail-closed (payload.total > 1000)
  # cron: "0 9 * * *"           # required when type: schedule

steps:
  - id: <step_id>
    type: <step_type>
    # type-specific fields (see below)
```

### Event-triggered flows

`trigger.type: event` binds an existing Business Event name to an existing Marketplace Flow. The event name is opaque metadata — the runtime does not hardcode domain events.

```yaml
trigger:
  type: event
  event: invoice.overdue
```

Dispatch (same Postgres event bus + FlowRunner as conversation/schedule — not a second bus, scheduler, workflow engine, or FlowRunner, and not Marketplace-specific Rust branches):

1. Entity mutation (RuntimeAdapter) emits a canonical envelope onto the existing bus.
2. CRM Automation and Goal Engine observe the same event (not merged into this path).
3. Resolution is workspace-scoped: only installed, enabled flows whose `trigger.event` equals the envelope name. No tenant-wide Marketplace scan. SDK event flows remain a separate subscriber path (priority unchanged).
4. FlowRunner starts with trusted refs: `event.id`, `event.name`, `entity`, `record_id` / `id`. The flow loads the record with `entity.*.get` — the full document is not auto-injected. Tenant comes from the record/envelope; workspace from trusted top-level `workspace_id`. Event facts are trusted.
5. Payload cannot override tenant, workspace, actor, or installation. `person_id` is a scoped Hub relation: customer channels inject session identity; staff/API may select an existing workspace Person after Hub lookup; event-triggered `entity.*.create` may copy `person_id` from a related record via field `inherit_from` after authority strip — never from payload/LLM.
6. Idempotency is `event_id + flow_id` (system conversation session). Same pair at most once while Completed; different events are independent. Bus idempotency remains authoritative for delivery.
7. FlowRunner failure NACKs the bus event (existing retry/dead-letter). Status lookup error, a missing execution row, or explicit `Failed` are not success. Catalog unavailable NACKs (retry). Empty installed set (no matching flows) ACKs. Success, no-op, and already-idempotent complete ACK.
8. Recursion: child emits from a flow mutation — including computed rollup, `on_events` related updates, and CSV hydrate when those run under an event-triggered flow — inherit `EventLineage` (`causation_event_id` + `depth`+1). Date-watch ticks remain depth-0 roots. Default max depth is 8 on the bus. This is loop protection, not a workflow graph.
9. Fan-out: at most 8 flows per event. Additional subscribers are truncated; the runtime logs/metrics the drop so truncation is observable. Subscriber priority (SDK vs installed) is unchanged.

Conversation catalog ignores `type: event` / `type: schedule` so chat selection is unchanged. Schedule ticks still emit `schedule.*` and resolve via the existing scheduler path.

Event vs CRM Automation vs Goal vs Agent:

- Event = fact (already happened)
- Flow = procedure (this trigger)
- CRM Automation = reaction (same bus, separate engine)
- Goal = outcome (same bus, separate engine)
- Agent = reasoning (unchanged)

Natural Event → Flow → Entity mutation → Event → CRM is allowed and bounded by depth + idempotency.

### Step types

| Step type | Purpose | Key fields |
|-----------|---------|------------|
| `ask` | Prompt user for input | `field`, `message`, `choices` or `choices_from`, `title_field`, `value_field`, `chip_prefix` |
| `tool` | Execute a capability | `tool`, `execution` (runtime\|http), `input_map`, `output` |
| `message` | Display text | `message` (supports `{{var}}` templates) |
| `complete` | End the flow | `message` |
| `condition` | Branch on logic | `conditions`, `on_match`, `on_no_match` |
| `challenge` | Verify identity (OTP) | `via`, `identity` |
| `upload` | Accept file | `field`, `message` |
| `delay` | Wait/pause | `duration` |
| `approval` | Request approval | `message`, `approver` |
| `tag` | Add tag | `tag` |
| `assign` | Assign record | `field`, `value` |
| `activity` | Log activity | `message` |
| `branch` | Multi-way branch | `branches` |
| `notify` | Send notification | `message`, `channel` |
| `handoff` | Transfer to human | `message`, `target` |

Every flow MUST end with a `complete` step.

### input_map syntax

```yaml
input_map:
  status: "$literal:scheduled"        # constant value (trusted)
  limit: "$literal:20"                # constant numeric
  customer_name: "{{customer_name}}"  # variable reference (from conversation)
  property_id: "{{property_id}}"      # variable reference
  filter.city: "{{city}}"             # nested key (becomes filter.city in JSON)
```

**$literal:** prefix produces a trusted constant. These are the only values the runtime trusts for expand directives and authority-adjacent fields.

**{{variable}}** references resolve from conversation variables (slots, tool outputs, person context).

**$constants.** prefix reads from flow-level constants (trusted, set by app developer at design time).

## choices_from pattern

```yaml
# Step 1: Fetch data
- id: fetch_properties
  type: tool
  tool: entity.property.list
  execution: runtime
  input_map:
    status: "$literal:available"
  output: properties              # captures result into variable

# Step 2: Present choices
- id: select_property
  type: ask
  field: property_id
  message: "Which property?"
  choices_from: properties        # references the output variable
  title_field: title              # display label from each item
  value_field: id                 # value from each item
  chip_prefix: "Property"         # UI chip label
```

Static choices alternative:
```yaml
choices:
  - id: consultation
    label: "Initial Consultation"
  - id: followup
    label: "Follow-up Visit"
```

## Event naming

Events use dot notation: `<domain>.<action>`.

```yaml
events:
  - order.created
  - order.updated
  - appointment.confirmed
  - appointment.cancelled
  - patient.updated
```

Events are produced by:
- Entity `status_events` mapping
- HTTP tool `emits` declarations
- Connection webhook topic mappings
- Date watches (`date_events`)

Marketplace Flows subscribe with `trigger.type: event` and `trigger.event: <same opaque name>`. The runtime matches by string equality against the workspace install catalog and starts the existing FlowRunner. See **Event-triggered flows** above.

## Permission set

```yaml
permissions:
  - workflow.execute
  - storage.read
  - storage.write
  - storage.update
  - storage.delete
```

## Capability vocabulary

### Format contract

Capabilities follow `resource.action` dot-notation (e.g., `storage.read`, `workflow.trigger`). The format itself is the contract -- any well-formed `resource.action` token is a valid capability.

```yaml
capabilities:
  - theme.get
  - user.get
  - tenant.get
  - runtime.query
  - workflow.trigger
  - storage.read
  - storage.write
```

The seven capabilities above are what all current entity-native Marketplace Apps declare. This is the common pattern, not a closed vocabulary.

### Negotiation

At install time, `negotiate_capabilities()` intersects what the UI requests with what the installation grants. The negotiation layer handles arbitrary `resource.action` tokens as long as they match a granted permission.

### Note on agent capabilities

Marketplace Apps do NOT declare `agent:` metadata. The Agent layer consumes capabilities already exposed by the Marketplace App through the runtime's entity capability projection and flow-based capabilities. See `agent-integration.md` for how Agent discovers and uses these capabilities automatically.

Runtime entity operations: `entity.{name}.{create|list|get|update|delete|restore|availability|aggregate}`.
`aggregate` is read-only (`op: count|sum`) and is projected to Business Agent automatically.
`restore` requires update permission and `storage.update`. Soft delete uses `storage.update`;
hard/restrict delete uses `storage.delete`.

## Conversation slot kinds

| Kind | Parser |
|------|--------|
| `string` | General text extraction |
| `integer` | Integer parsing |
| `float` | Decimal parsing |
| `date` | Date parsing (relative + absolute) |
| `time` | Time parsing |
| `email` | Email format validation |
| `phone` | Phone normalization (E.164) |
| `person_name` | Name extraction |

Slots with `identity: person_name` link to Customer Hub identity resolution.

## Connection auth types (HTTP-backed apps)

| Auth type | Config |
|-----------|--------|
| `bearer` | OAuth token management |
| `named_header` | Custom header injection |
| `basic` | Username/password |
| `none` | No auth |

## Webhook verification metadata

```yaml
webhooks:
  hmac:
    algorithm: hmac-sha256
    encoding: hex|base64
    header: X-Provider-Hmac
```

Or extended signature format:
```yaml
webhooks:
  signature:
    algorithm: hmac-sha256
    source: header
    header: X-Signature
    format: timestamped|prefixed|raw
    encoding: hex|base64
    timestamp_param: t
    signature_param: v1
    signed_payload: "{timestamp}.{body}"
    max_skew_seconds: 300
```
