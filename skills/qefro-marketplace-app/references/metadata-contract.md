# Metadata Contract

**Qefro Event-Driven Runtime v1 frozen.**

Architecture: Marketplace Metadata → Generic Runtime → Entity mutation → Business Event → CRM Automation | Goal Engine | Event → FlowRunner. No Agent v2 / Goal v2 / FlowRunner v2 / EventBus v2 / Scheduler v2 / Availability v2 unless a real product/customer gap appears.

## Three contracts, and which one bites

A key in a package passes through three independent stages. Each has its own vocabulary, and a key can survive one and die at the next:

1. **Package parser** (`solution-service/src/solution/*.rs`) — decides what publishes. Different structs behave differently and this is the most common source of wrong assumptions:
   - `FlowDefinition` / `FlowStep` end in `#[serde(flatten)] extra`, so unknown keys are **preserved verbatim** into stored JSON.
   - `UiSection` and its nested structs are closed without `extra`, so an unknown key fails the whole package with `invalid ui section: …`.
   - `SolutionPackageManifest` is closed **without** `deny_unknown_fields`, so an unknown manifest key is **silently accepted and then dropped** — it never reaches the registry or any consumer. `ManifestAutomationTemplate` and `ManifestAutomationTrigger` *do* set `deny_unknown_fields`.
   - Entity definitions are stored as permissive JSON, so entity-level keys generally survive.
2. **Installed bundle** — what a consumer can read back. Chat context (`api/chat_context.rs`) forwards `manifest.components.entities` as **raw stored JSON**, so every entity key survives to the runtime. The UI bundle is the **typed `UiSection` re-serialization**, so a UI key the DTO lacks is gone even though it parsed.
3. **Executed / rendered behaviour** — the ACS runtime or the portal's Generic UI. A key can be preserved at stages 1 and 2 and still do nothing here, because nothing reads it.

Practical rule: to prove a metadata key works, find the code that reads it in stage 3 — not a reference app that merely declares it. Reference-app YAML is evidence of intent, not of enforcement. When a doc or a plan says "the app declares X, therefore Y happens", check which of the three stages actually consumes X.

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
entities: [<entity-ids>]        # each must exist under entities/
flows: [<flow-ids>]             # each must exist under workflows/
prompts: [<prompt-ids>]         # each must exist under prompts/
onboarding: [<step-ids>]        # each must exist under onboarding/
events: [<dot-notation names>]  # e.g. ["appointment.created", "appointment.cancelled"]

# Optional CRM Automation presets (not executable). Instantiating one
# creates a normal existing CrmAutomation in that workspace. Never put
# URLs, secrets, headers, or connection_id here. send_webhook templates
# may declare payload_mapping only; the workspace picks the destination.
automation_templates:
  - id: <kebab_or_snake_id>
    name: <display name>
    description: <string>
    trigger:
      event: <declared event>
    actions:
      - type: create_followup|notify_team|send_webhook|...  # CRM AutomationAction vocabulary
        # existing action fields only
permissions: [workflow.execute, storage.read, storage.write, storage.update, storage.delete]
capabilities: [theme.get, user.get, tenant.get, runtime.query, workflow.trigger, storage.read, storage.write]

# HTTP-backed apps:
connections: [<connection-ids>]
http_tools: [<tool-ids>]

# SDK-app / hosting extras (all optional):
connectors:                      # external connector ids resolved from the shared pool
  - name: <connector-name>       # `core.customer_hub` is reserved and always stripped
    version: <semver>            # default "*"; the pool resolves it, never the tenant
tools: [<app-tool-ids>]          # advertised by this app's SDK connector
runtime_adapters: [<adapter-ids>]  # explicit opt-in; the runtime never infers one from tool names
collections:                     # informational code-allocation hints
  - name: <collection>
    allocate_code: { prefix: "<prefix>", start: <int> }
agent:                           # capability declaration only — grants no RBAC
  enabled: true
  roles:
    customer:
      capabilities: [<KNOWN_AGENT_CAPABILITIES tokens>]
      tools: [<names present in tools / http_tools>]
      flows: [<ids present in flows>]
    business:                    # the staff-side role name is `business`, not `staff`
      capabilities: []
      tools: []
      flows: []
implementation_kind: workflow    # legacy ADR-003 spelling, mapped into `hosting`; do not use in new apps
endpoint: <url>                  # optional /qefro endpoint for externally hosted apps

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

# Optional workspace install settings (Real Estate Pro, manifest.yaml).
# ALLOWLIST form: bare keys and/or typed specs. Types are narrower than the
# settings schema's — json|object|array are rejected here. Every key listed
# must also exist in `settings/business.yaml` (subset check at publish).
settings:
  - <setting_key>                                  # shorthand: type string
  - key: <setting_key>
    type: string|number|boolean|color|url
    required: <bool>
    default: <value>
    description: <string>
    secret: <bool>       # portal password field; value never enters the package

# Optional multi-step WhatsApp cadence (Real Estate Pro `lead_followup`).
# Not an install-time campaign: a Goal must reference followup_sequence_id.
# The manifest parser validates NOTHING here — see "Follow-up sequences".
followup_sequences:
  - id: <sequence_id>
    name: <display name>
    description: <string>
    channel: whatsapp
    steps:
      - id: <step_id>
        template: <meta_template_name>
        delay: { unit: minutes|hours|days, value: <positive int> }
        parameters: ["{{person.name}}", "{{entity.<field on the goal's entity_type>}}"]

ui:
  name: <display name>
  navigation_layout: sidebar|topbar   # host chrome; theme.navigation_layout also accepted
```

## Entity schema

```yaml
id: <entity_id>
scope: customer                 # optional -- present when customer-facing
name: <display name>
description: <string>
collection: <storage collection>  # optional; defaults to `id`

title_field: <field>            # record title; hosts must not infer it from field names
display_field: <field>          # optional secondary display field
status_field: <field>           # field compared for status events (default `status`)

views:                          # declared list surfaces
- type: table
  columns: [<field>, ...]
- type: tile
  title_field: <field>
  status_field: <field>

# Optional auto-incrementing code:
allocate_code:
  prefix: <string>              # literal, e.g. "O-" produces "O-1001" (no separator added)
  start: 1001
  field: <unique string field>  # optional; otherwise the first unique string field, else "code"

fields:
  - name: <field_name>
    type: <field_type>
    required: true|false
    description: <optional>
    enum_values: [<values>]     # required when type: enum
    ref_entity: <entity_id>     # required when type: person or relation
    multiple: true|false        # media only
    max_items: <int>            # media only
    unique: true|false          # optional — CRUD create/update + CSV importer enforce uniqueness
                                # per tenant/workspace. Absence means always-create.
                                # Rejected on person|relation|image|media|json fields.
    import_synonyms: [<headers>] # optional — extra CSV header names for this field
    import_default: <value>      # optional — filled when CSV omits or blanks the cell
    default: <value>             # optional — filled on create when empty; reserved token
                                 # `current_date` = workspace-local today (not a formula)
    default_from_setting: <key>  # optional — install setting used as create default.
                                 # Checked BEFORE `default`; `0`/`false` count as filled.
    inherit_from: <relation>     # person fields only; event-triggered creates
    computed: true               # server-computed; hosts render readonly, never calculate
    readonly: true               # display hint
    label: <string>              # display label
    format: currency             # display hint; never inferred from name or geography

# Optional status-to-event mapping. On create, `{entity}.created` is always
# emitted; if the new record has a status with a mapping, that event is also
# emitted (deduped when it equals `{entity}.created`). On update, status
# events fire only when the status field actually changes.
status_events:
  <status_value>: <event.name>

# Optional WhatsApp Catalog overlay (presentation only).
# Not a second entity type. Catalog Service maps this record via `mapping`.
catalog:
  enabled: true
  type: product|service
  mapping:
    title: <field>              # required
    description: <field>
    price: <field>
    currency: <field>
    image: <field>              # public HTTPS URL field
    url: <field>
    availability: <field>
    brand: <field>
  publish_when:
    field: <status field>
    in: [<values that may publish>]
  availability:
    in_stock: [<values>]
    out_of_stock: [<values>]

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
#   required: true     — enforced for EVERY declared field, including person/image/media,
#                      after defaults are applied. Updates do not re-check completeness.
#   enum_values        — rejected if not in the list (case-insensitive, stored as declared)
#   unique: true       — advisory lock on one Postgres session is held through the storage write
#                      (UniqueWriteLock unlocks on success, error, storage failure, and Drop)
#   integer/float/boolean/date — coerced AND validated on CRUD (entity_validate::prepare_document);
#                      person/image/media/json values pass through uncoerced
#   allocate_code      — caller-supplied target value is cleared on create so the
#                      generated code cannot be overridden

# List accepts `sort` (object or field name). Authority keys stripped; max 3 fields.
# Filter objects such as `{ due_date: { $gte: "2026-01-01" } }` are passed through;
# string equality filters are still case-insensitive exact regex.
# `entity.{name}.aggregate` is person-scoped on customer channels (same as list).

# CSV commit stamps version=1, skips soft-deleted rows on unique lookup, then
# replays generic post-create hooks (computed, events, on_events, date watches)
# for at most 200 created ids. It is not a second import engine and does not
# apply availability or relation-cap checks.

# Optional CSV import helpers on a field:
#   unique: true           — importer AND CRUD create/update enforce uniqueness
#   import_synonyms: []    — extra header names for the generic mapper
#   import_default: <val>  — filled when the CSV omits the column OR the cell is blank
#   default / default_from_setting — NOT applied by the CSV importer (CRUD only)
# `allocate_code` is skipped for a batch when any row already supplies the target field.

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
# person_id is a tenant-global Hub relation, not a spoofable authority key.
# LLM/agent payloads still strip it. Staff/API may select an existing Person
# only after Hub lookup proves the id exists in THIS tenant (any workspace).
# Customer channels inject session identity (self only). Event-triggered
# create may copy it via inherit_from. CSV resolves via PersonService.
# Cross-tenant, nonexistent, and nil ids are rejected. Tenant/workspace
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
| `media` | Image list | Same host as `image`; `multiple` / `max_items` |
| `textarea` / `text_long` | Long text | UI alias for multi-line `string` |

Hosts render by **canonical `type`** (`enum_values`, `ref_entity`, `multiple`). They must not infer widgets from field names (`property_title`, `image_url`, `patient_name`) or ship domain enum lists. Computed specs stay server-side: if a field is listed under entity `computed:` or `computed: true`, the UI shows the **server-provided value** readonly.

Optional UI-only hints (passthrough, ignored by the runtime if unknown): `readonly`, `placeholder`, `help`, `section`, `label`, `title_field`, `display_field`, `format` (`currency` for money display). Hosts must not infer currency from field names (`price`, `rent`) or geography (`Chennai` → INR).

### Availability declaration (structural capability)

`availability:` on an entity is the **only** booking primitive. It declares what the entity *can* do; the workspace settings declare how it operates for this customer. Both halves are required and neither is optional metadata you can skip.

```yaml
# entities/viewing.yaml (real-estate-pro) / entities/reservation.yaml (restaurant-pro)
availability:
  booking:
    datetime_field: scheduled_at      # required; the canonical UTC datetime column
  capacity:
    mode: exclusive                   # exclusive | unlimited  (limited is NOT enforced)
    resource:
      field: property_id              # one booking at a time per value of THIS field
```

Split of responsibilities — `availability::extract_availability_config` reads the entity, `overlay_workspace_settings` then overwrites the operating values from install settings:

| Owned by | Keys | Why |
|----------|------|-----|
| **Entity metadata** | `booking.datetime_field`, `capacity.mode`, `capacity.resource.field` | Structural. Two workspaces of the same app must not silently disagree about whether a viewing blocks a *property* or an *agent*. Deliberately **not overridable** by settings. |
| **Workspace settings** | `timezone`, `duration_minutes`, `start_time`/`end_time` (or `working_hours.start_time`/`.end_time`, `viewing_hours.*`, `opening`/`closing`), `closed_days`, `date_window_days`, `blocking_statuses` | Operating. Changing office hours must not require editing app YAML. |

`overlay_workspace_settings` accepts flat keys **or** an `availability:` nested object, each alias list in order, and ignores unknown keys. `closed_days` and `blocking_statuses` accept a JSON array **or** a comma/semicolon-separated string (`"sunday"`, `"scheduled,confirmed"`). Numeric settings may arrive as strings and are parsed. Anything that fails validation (`timezone` not IANA, `start_time` not `HH:MM`, `duration_minutes <= 0`) is **skipped silently**, leaving the entity-side value — so a mistyped setting looks like "my change did nothing".

`require_valid_config` runs at call time and fails closed on: `limited` capacity mode (explicit V1 error, never treated as unlimited), unparseable timezone, bad working hours, non-positive duration or window, empty `datetime_field`. Because it is a runtime error rather than a publish error, an entity with `availability:` and no workspace settings at all publishes fine and fails on first use.

Tool: `entity.<id>.availability` (`execution: runtime`), dispatched by `kind`:

| `kind` | Required params | Output |
|--------|-----------------|--------|
| `dates` | — | `{ items: [{ id, label, value, start_at }], availability, timezone, duration_minutes, datetime_field }` over the configured window; error if the window yields nothing |
| `slots` | `date` | items additionally carry `time`, `date`, `timezone`, `duration_minutes`, `resource`, `local_start`, `utc_start`, `utc_end`; `id`/`value`/`start_at` are the canonical UTC datetime |
| `validate` | `date`, `time` | `{ valid, utc_datetime, utc_end, items:[…] }`, or a tool error carrying the customer-facing reason |

`kind` accepts `slots`/`times`/`time`, `validate`/`check`, `dates`/`date`; with no `kind` it infers from whether `date` (and `time`) were supplied. The resource comes from `params.resource`, else from `params[capacity.resource.field]` — which is why the reference flow passes `property_id:` on every availability call. Existing bookings are loaded **only** for `slots` and `validate`, and only under `exclusive`.

After an `ask` step whose field is the availability datetime (or `date`/`time`/`scheduled_at`/`input`), the executor runs `apply_availability_reply` and writes `date`, `time` and `booking.datetime_field` (UTC) back into flow variables before advancing. An invalid reply sets `_validation_error` and **re-asks the same step**; it does not fall through to the create. That is what makes the create step safe to write `scheduled_at: scheduled_at` from a bare path.

### Presentation metadata V1

Presentation is **not** authorization. It never creates capabilities, bypasses permissions, or changes FlowRunner/storage semantics.

Declared list views stay on the entity (`views: [{ type: table|tile|... }]`). Do **not** duplicate that as `ui.list.views` or `display_mode`.

Optional `entity.ui` (package YAML + workspace `_ui_customization` overlay, merged; unknown IDs ignored):

```yaml
ui:
  style: modern          # visual-only: modern | classic | compact
  list:
    columns: [<field>]
    sort: { <field>: asc|desc }
    filter: { <field>: <scalar> }
    default_view: tile   # must already be in entity.views
    empty_state:
      title: <string>
      description: <string>
      action_label: <string>
  detail:
    layout: split        # stack | split
    hero:
      title_field: <field>
      subtitle_field: <field>
      status_field: <field>
      image_field: <image|media field>
    metrics: [<field>]   # max 6; CanonicalField display
    hidden: [<field>]
    order: [<field>]
    sections:
      - title: <string>
        fields: [<field>]
    related:
      order: [<entity_id>]
      hidden: [<entity_id>]
    activity: true
  form:
    order: [<field>]
    hidden: [<field>]
    sections:
      - title: <string>
        fields: [<field>]
    labels: { <field>: <string> }
    help: { <field>: <string> }
    related_create:             # package-authored only; not a workspace customization
      <child_foreign_key>:
        prefill: { <child_field>: <parent_field> }
        context_fields: [<parent_field>]
  actions:
    primary: <workflow-id>     # existing workflow trigger order
    order: [<workflow-id>]
    hidden: [<workflow-id>]
    items:                     # typed Generic UI actions (not workflows)
      - type: print
        target: current_record   # current_record | current_view
        label: Print             # optional
        print:                   # optional per-action override
          format: document       # document | receipt
          paper_size: A4         # A4 | letter | receipt
          orientation: portrait  # portrait | landscape
  print:                       # optional entity defaults; presentation only
    enabled: true
    format: document
    paper_size: A4
    orientation: portrait
```

`type: print` is a Generic UI action. The portal Print Renderer builds a print-friendly document from the current record or the current list/table view and opens the browser/system print dialog. It is not USB/Bluetooth, not ESC/POS, and not an app-specific printer service.

YAML may also write `ui.actions` as a list of typed actions when there are no workflow `primary`/`order`/`hidden` hints. Unknown action types are ignored. `enabled: false` hides print. Views and table widgets may declare the same `actions` / `print` keys.

### Work queues and calendar views

Declare queues under the entity's `ui.list.queues`. A queue has a stable `id`, a `label`, and optional `filter`, `sort`, and `date_window`. Filters use declared scalar fields. Enum filters may list up to 20 allowed values. Sort uses up to three declared fields. A `date_window` names a `date` or `datetime` field and a range of `today` or `next_7_days` (including today). The runtime resolves queue IDs against the installed package and computes date boundaries in the workspace timezone. Selecting a queue replaces the list defaults and preserves the relationship scope and search.

Declare entity calendar views with `views: [{ type: calendar, start_field: <field> }]`. Web requests the displayed month and mobile requests the displayed week. The entity list filters before paging and returns workspace-local day keys for grouping. Date-only values are calendar dates. Offset-bearing datetimes are converted to the workspace timezone.

### Conditional entity fields

Add `visible_when` or `required_when` to an entity field using `{ field, operator, value }`. The source is a declared enum or boolean field. Use `eq` (the default) or `in`. If both rules are present, use the same condition. Visibility controls the generated forms. Conditional requirements apply to create, CSV import, and updates that touch either side of the rule.

### Contextual record actions

In `manifest.yaml`, extend a first-class `actions` item with `when` using the same condition syntax. Set `target: { entity, record_bound: true }` and `required_permission: entity.<entity>.update`. Optional `primary` and `confirm` flags control placement and confirmation. Supported execution types are `state_transition`, a literal `mutation` update using `payload_template`, or a declared staff `workflow` without interactive asks. The existing action endpoint checks the live record. Synchronous updates carry the loaded record version. Workflows receive the record ID and `expected_version`, which their update step should pass to the runtime.

### Related creation defaults and parent context

Declare this on the **child entity**, keyed by its actual relation field. The generic related-record Add form reads the selected parent record; it does not run a create workflow or copy same-named fields automatically.

Restaurant Pro's `entities/order.yaml` supplies the concrete pattern:

```yaml
ui:
  form:
    related_create:
      reservation_id:
        prefill:
          guest_name: guest_name
          person_id: person_id
          table_id: table_id
          table_name: table_name
          notes: notes
        context_fields: [guest_count, date, time]
    hidden: [table_name, reservation_code, code, total_amount]
```

- Read `prefill` as **child destination → parent source**, each a literal declared field name. No expressions, templates, nested paths, or implicit copying.
- Require the child FK to resolve to the current parent entity. Require both mapped fields to exist, have equal types and, for references, equal relation targets.
- Skip protected destinations: parent FK, record identifiers, authority fields, allocated/unique fields, readonly and computed fields. Skip missing/empty, structured, nonfinite and invalid enum values; retain valid `0` and `false`.
- Apply valid parent values before ordinary create defaults. Keep mapped editable fields editable, even when they have an ordinary default; retain explicit form-hidden rules and existing required-field protection.
- Always supply the real parent FK separately and keep it fixed in the form. Update/clear a selected relation's display companion when its picker changes; do not leave stale parent-derived labels.
- Render `context_fields` read-only in declared order using the parent's field metadata. Do not submit them as child values. Omit unknown, empty, opaque-ID, Person, media, object and raw-UUID context values.
- Cancel/reopen resets unsaved edits to parent defaults. Standalone New/Edit forms and relationships without mappings keep their existing behavior.
- Preserve package mappings when workspace `_ui_customization` replaces form presentation. That overlay cannot add or override `related_create`.
- Treat this as staff UI convenience, **not** trusted server inheritance or authorization. Normal workspace-scoped CRUD, Person validation, permissions and entity integrity still apply. `person.inherit_from` is a separate server event-create feature.

Reservation date/time and guest count are context; do not map them to takeaway `pickup_date` / `pickup_time`. Do not copy parent status or generated codes. For another app, choose semantically equivalent fields explicitly.

For example, a property → viewing relationship can opt into `property_id: { prefill: { property_title: title }, context_fields: [city, property_type] }` under `ui.form.related_create` if those fields exist. This is an **adaptation example**, not a claim that Real Estate Pro currently declares the mapping. Cross-app portal tests also rename the solution, entities, FK and source/destination fields to prove this is generic.

Consumers: portal `fieldModel.ts::relatedCreateContext`, `EntityCrud.tsx::RelatedRecordsPanel`, and `uiCustomization.ts` under `src/pages/solutions/ui/`; shape in `src/api/client.ts`. Entity `ui` is preserved by solution-service; do not add per-app runtime code.

### Dashboard chart widgets (`ui/widgets.yaml`)

Generic UI renders `type: chart` through the shared ChartWidget (not per-app React). Data must use an entity `source` and the authorized `POST /runtime/ui/data` → `entity.{target}.aggregate` path:

```yaml
- id: chart_properties_by_type
  type: chart
  title: Properties by type
  source: properties          # ui/sources.yaml entity source
  options:
    kind: bar                 # bar | line | pie | donut
    aggregate: count          # count | sum | today
    group_by: property_type   # never tenant/workspace identifiers
    # field: amount           # required for sum
    # time_bucket: month      # day | week | month for a date dimension
    # filter: { status: listed }
    empty_title: No property data available.
```

Metric tiles use the same aggregate path without `group_by` (`aggregate: count|sum|today`). Do not invent a second analytics API or marketplace-specific chart components.

Stale field/entity IDs are dropped. Overlay cannot invent views, fields, or workflows.

### UI package files and closed sets

`solution::ui` assembles the `ui/` directory into one typed `UiSection`. Two properties explain most "my UI didn't appear" reports:

**Every file is optional; every key must be typed.** `FORBIDDEN_UI_EXTENSIONS` rejects `.js .mjs .cjs .ts .tsx .jsx .html .htm .sh .py .wasm .exe .dll .so .jar .php` under `ui/` at parse time (Rule 4 — packages ship data). Conversely, `UiSection` is a **closed struct without `#[serde(flatten)]`**: a key the DTO does not declare fails `serde_json::from_value`, and the whole package is rejected with `invalid ui section: <unknown field>` — the opposite of the workflow DTO, which silently preserves unknown keys. So workflow YAML tolerates new metadata and UI YAML does not.

**Preset references are resolved before storage**, so hosts never see them. `resolve_layout_presets` rewrites each page's `layout: <preset-id>` into the layout's inline value; if the preset is undeclared it is a publish error, but a page whose preset matches has its `layout` field replaced — the tenant bundle contains no preset ids. `derive_nav_routes` overwrites routes: every navigation entry gets `/app/solutions/ui/{solution}/{entry}`, and a hand-written `ui/routes.yaml` entry survives only when its id does **not** collide with a navigation id.

Publish-time checks (all `invalid_section("ui", …)`):

| Object | Rule |
|--------|------|
| section | `name` non-empty; `logo`/`icon` pass `validate_ui_asset_ref`; every `capabilities[]` entry in `is_known_capability`; `navigation_layout` case-insensitively `topbar`\|`sidebar` |
| page | non-empty `id` **and** `title`; unique `id`; `host` ∈ `contacts`\|`automations`\|`roles`; layout kind `grid` only; `columns` 1–12; each widget placement references a declared widget and `span` 1–12 |
| widget | unique `id`; `kind` ∈ `is_known_widget_kind` — `metric`, `table`, `chart`, `kanban`, `calendar`, `status`, `map`, `timeline`, `activity`, `markdown`, `form`, `booking_link`, `cart_checkout`; a declared `source` must reference a declared source id |
| source | `kind` ∈ `runtime`\|`connector`\|`entity`; non-empty `target`; target must not be a platform `storage/*` name (ADR-003); `connector` targets must be `{connector}/{operation}` |
| navigation | unique `id`; `icon` ∈ `ALLOWED_ICONS`; `page` (defaulting to the entry id) must be a declared page |
| route | `page` must be a declared page |

`form` widgets are the one widget kind with a validated option schema (`validate_form_widgets`, called with the manifest's flow ids). A `form` widget **must not** declare `source`, must have `options`, and must declare a non-empty `options.fields`. Each field must be an object with a non-empty `name`; `type` is optional and defaults to `text`, must be a **string**, and must be in `UI_FORM_FIELD_TYPES` = `text`, `textarea`, `number`, `date`, `time`, `select`, `image`, `media`; `required` must be a boolean. A `select` field must supply `options` (array of strings, non-empty) **or** `options_source` (an id declared in `ui.sources`) — neither, or a non-declared source id, is a publish error. There is no `enum`, `relation`, `person`, `boolean`, `float` or `datetime` form-field type: a relation is a `select` fed by `options_source`, and a `person` field is host-rendered through Customer Hub rather than declared as a form input.

Submission must ride the capability layer: `options.action` is required and must declare at least one of `trigger` (a workflow id present in the manifest) or `emit` (a valid event name). A form widget that only "looks at" data with no action cannot publish.

There is **no option schema for any other widget kind**. `options` is free-form JSON, which is why widgets are *bound* rather than configured: `cart_checkout` reads its mapping from options, `chart` reads `kind`/`aggregate`/`group_by`, and any option a widget does not read is inert. Adding a new option key needs a portal consumer in the same change.

Also note `effective_capabilities`: the manifest's `capabilities` list is authoritative, and a legacy `ui/capabilities.yaml` applies **only when the manifest declares none**. Declaring both silently ignores the `ui/` list.

### Cart / checkout widget (`cart_checkout`)

`parseCartCheckoutOptions` (`cartCheckout.ts`) is a closed option map with defaults — declaring an entity name that differs from a default silently targets the wrong field rather than failing.

| Group | Options | Default |
|-------|---------|---------|
| required | `catalog_entity`, `line_entity`, `document_entity` | none — missing any renders "Cart checkout needs …" and the widget is unusable |
| catalog | `catalog_title_field`, `catalog_price_field`, `catalog_sku_field`, `catalog_category_field`, `catalog_category_entity`, `catalog_availability_field`, `catalog_unavailable_values`, `catalog_filter` | `name`, **`selling_price`** |
| line | `line_parent_field`, `line_parent_code_field`, `line_catalog_field`, `line_qty_field`, `line_price_field`, `line_description_field` | `{document_entity}_id`, `quantity`, `unit_price` |
| document | `document_status`, `document_status_field`, `document_person_field`, `document_total_field`, `document_paid_field`, `document_outstanding_field`, `document_number_field`, `document_issue_field`, `document_due_field`, `document_currency_field`, `document_guest_field`, `document_table_field`, `document_notes_field`, `document_type_field`, `document_type_default`, `document_table_when`, `document_page`, `person_required` | `status`, `total_amount`, **`invoice_number`**, `currency` |
| table | `table_entity`, `table_label_field` | `name` |
| payment | `payment_entity`, `payment_amount_field`, `payment_method_field`, `payment_status`, `payment_status_field`, `payment_date_field`, `payment_person_field`, `payment_currency_field`, `payment_document_field`, `payment_document_id_field`, `payment_guest_field`, `payment_methods` | `amount`, `method`, `status`, `currency`, `[cash, upi, card, other]` |
| allocation | `allocation_entity`, `allocation_document_field`, `allocation_payment_field`, `allocation_amount_field`, `allow_partial` | `{document_entity}_id`, `payment_id`, `amount`; `allow_partial` defaults **true when `allocation_entity` is set** |
| chrome | `submit_label`, `new_label`, `cart_title` | `Complete sale`, `New sale`, `Cart` |

`document_paid_field` / `document_outstanding_field` fall back to `paid_amount` / `outstanding_amount` only when `allocation_entity` is declared — pair them. `payment_person_field` falls back to `document_person_field`. `catalog_unavailable_values` is lowercased on parse, so the values are matched case-insensitively against the catalog field.

Restaurant Pro is the reference: `menu_item` → `order_item` → `order`, with `line_catalog_field: menu_item_id`, `document_number_field: code` (overriding the `invoice_number` default) and `catalog_price_field: price` (overriding `selling_price`).

The widget only **creates** records through the ordinary authorized `entity.*.create` path; server-side `computed` (`amount = quantity × unit_price`, `total_amount = relation_sum`) and `relation_caps` remain the integrity boundary. Never treat the widget's own totals as authoritative, and never let a checkout write to a document without an explicit confirmed workflow if the business requires confirmation.

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
  THIS tenant (any workspace). Customer Hub is tenant-global, not workspace-bound.
  Cross-tenant, nonexistent, and arbitrary ids are rejected. Tenant spoofing
  remains impossible.
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

### Catalog overlay

`catalog` is a thin mapping from this Marketplace **entity** onto a normalized catalog item, then a channel adapter (WhatsApp/Meta first). It is **not** a Real Estate feature, not FlowRunner 2, not `flow_engine/catalog.rs` (workflow selector), and not app-specific Rust.

| Field | Required | Contract |
|-------|----------|----------|
| `enabled` | no (default true) | When false, the entity is not catalog-publishable |
| `type` | yes | `product` or `service`. Qefro attribute; Meta still upserts a Product Item |
| `mapping.title` | yes | Source field for catalog title/name |
| `mapping.description` | no | Source field |
| `mapping.price` | no | Major units; adapter converts to Meta minor units |
| `mapping.currency` | no | ISO 4217 field |
| `mapping.image` | no | Public HTTPS URL field (localhost / RFC1918 rejected) |
| `mapping.url` | no | Public HTTPS URL field (a field **path**, not a literal URL in YAML) |
| `mapping.availability` | no | Source status/stock field |
| `mapping.brand` | no | Source field |
| `publish_when` | no | `{ field, in: [...] }` — records outside the list are unpublished |
| `availability` | no | Maps source values onto Meta `in stock` / `out of stock` |

Forbidden on the overlay object: `access_token`, `verify_token`, `app_secret`, `endpoint`, `base_url`, `tenant_id`, `workspace_id`, `installation_id`, `person_id`, `waba_id`, `catalog_id`, `meta_flow_id`.

Runtime path (generic, one-app-per-workspace):

1. Entity YAML `catalog:` → installed entity definition (chat-context)
2. Entity mutation `{entity}.created|updated|cancelled|status_changed` on the existing bus, or admin Publish / Sync / Unpublish
3. Catalog Service maps the record → `NormalizedCatalogItem` → WhatsApp adapter → Meta `POST /{CATALOG_ID}/products` (`allow_upsert`)
4. External ids stored in `channel_catalog_items` (tenant + workspace scoped). `retailer_id` is `{entity}:{record_id}`
5. Inbound catalog/order webhooks emit `channel.catalog.item_inquired` / `channel.catalog.order_received` only — no implicit Lead / Viewing / Order

The `catalog:` overlay is optional. WhatsApp connection and WhatsApp Flows work without a Meta Product Catalog. If a catalog is attached to the WABA, Qefro discovers it via `GET /{WABA-ID}/product_catalogs` and records `product_catalog_id` on the existing workspace WhatsApp config. If none exists, or `catalog_management` is missing, catalog stays unused (`not_connected` / `unavailable`). Qefro does not create catalogs.

Reference entities: `real-estate-pro` `property` (service), fixtures `menu_item` (product), `care_service` (service), `job_service` (service). Same overlay shape; different field mappings.

## Install settings — two shapes, one operative schema

A package declares workspace-scoped operating values in **two places that are not the same thing**:

```yaml
# manifest.yaml — Vec<SettingRef>: bare keys and/or typed specs. ALLOWLIST only.
settings:
  - key: timezone
    type: string
    default: Asia/Kolkata
    description: IANA timezone for viewing slots
  - currency                      # bare-key shorthand; implies type: string

# settings/business.yaml — SettingsSchema: nested mapping. THE OPERATIVE SCHEMA.
settings:
  timezone:
    type: string
    required: true
    default: Asia/Kolkata
    description: IANA timezone for viewing slots
```

| Aspect | manifest `settings:` | `settings/business.yaml` |
|--------|----------------------|--------------------------|
| Shape | sequence of `Key(String)` or `{key, type, required, default, description, secret}` | mapping `key → {type, required, default, description}`; scalar shorthand `key: string` also parses |
| Types | `string` \| `number` \| `boolean` \| `color` \| `url` — anything else is a **publish error** | additionally `json`/`object`/`array`, and aliases `bool`, `integer`, `float` |
| `secret: true` | supported (portal password field; value only at install time) | **not a field on `SettingSpec`** — silently ignored here, declare it on the manifest spec |
| What it drives | subset check + registry projection | the stored `solution_settings` rows, install-time validation and default merging |

Rules that bite:

- **Exactly one `settings/` file is read.** `zip_assemble::load_settings` returns the first hit in a fixed preference order (`settings/business.yaml`, `business.yml`, `settings.yaml`, `settings.yml`, `business.json`, `settings.json`), else the alphabetically-first `settings/*.{yaml,yml,json}`. A second settings file is never merged — split keys across `business.yaml` and `hours.yaml` and the `hours.yaml` keys simply do not exist.
- Manifest keys must be a **subset** of schema keys (`require_subset("settings", …)`). Every key you list in the manifest must also appear in the settings file; extra schema keys are allowed and stay unprojected in the registry manifest.
- Missing `type:` in the schema defaults to `string` silently, so `required: true` + a numeric value fails at **install** time, not publish time.
- Install is fail-closed: `validate_values` rejects a required key with no default, a type mismatch, and any key the schema does not declare. The only exempted keys are host chrome — `is_host_setting_key` = leading `_` or `navigation_layout`.
- `merge_defaults` fills declared defaults only when the caller omitted the key; caller values always win. So an app default is a fallback, never a constraint — a workspace can pin `timezone` to anything valid for its type.

Settings are read at execution time by `solution_install_settings`, which is how availability operating values reach the runtime (see **Availability declaration**). Any other tool that needs them must accept them as parameters — there is no generic "read my settings" step, so a flow cannot branch on a setting it did not pass through a `constants:` value or a tool param.

## Flow schema

```yaml
id: <flow-id>
name: <display name>
description: <string>
surfaces: [customer|staff]      # optional -- access control
enabled: true                   # optional; false skips conversation + event dispatch

# Optional Meta WhatsApp Flow overlay (presentation only).
# Not a second workflow language. FlowRunner still executes `steps`.
whatsapp_flow:
  category: APPOINTMENT_BOOKING # required; documented Meta category
  title: Book appointment       # optional display / Meta Flow title
  cta: Book                     # optional footer CTA

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

### WhatsApp Flow overlay

`whatsapp_flow` is a thin mapping from this Marketplace workflow onto Meta’s WhatsApp Flow JSON. It is **not** `communications.whatsapp.flows`, not FlowRunner 2, and not app-specific Rust.

| Field | Required | Contract |
|-------|----------|----------|
| `category` | yes | One of Meta’s documented categories: `SIGN_UP`, `SIGN_IN`, `APPOINTMENT_BOOKING`, `LEAD_GENERATION`, `CONTACT_US`, `CUSTOMER_SUPPORT`, `SURVEY`, `OTHER` |
| `title` | no | Settings label / Meta Flow title |
| `cta` | no | Footer button label |
| `workflow` | no | Allowed and ignored (workflow id is the YAML `id`) |

Forbidden on the overlay: `tenant_id`, `workspace_id`, `person_id`, `access_token`, `endpoint`, `url`, `secret`, or any other key.

Publish path: solution-service parses workflow YAML into `FlowDefinition`, whose declared fields cover only `id`, `name`, `description`, `surfaces`, `trigger`, `steps`, `constants`. Everything else — including `whatsapp_flow` — is captured by a trailing `#[serde(flatten)] extra` map and therefore **preserved verbatim** into the stored JSON (`solution_workflows.definition`). Steps work the same way: declared keys plus `extra`. Two consequences:

- Adding a new overlay or step key needs **no package-parser change**; it needs a consumer. If nothing downstream reads the key, it is inert metadata.
- Unknown keys are silently kept, not rejected. A misspelled key (`whatsappFlow`, `inputMap`) publishes cleanly and does nothing. Check the consumer, not the publish result.

If the overlay is stripped or renamed, ACS discovery finds zero overlays and Settings shows “No customer workflows declare a WhatsApp Flow overlay yet.”

Discovery path (generic, one-app-per-workspace):

1. Installed package → chat-context `workflow_definitions[].definition`
2. `POST /api/v1/admin/whatsapp/flows/sync` (Owner/Admin) compiles overlays and provisions Meta Flows on the **existing** workspace WhatsApp client
3. `GET /api/v1/admin/whatsapp/config` returns status snapshots (no tokens)

Reference workflows: `real-estate-pro` `request-viewing`, `clinic-pro` `book-appointment`, `restaurant-pro` `book-table`. Same overlay shape; different entity tools.

### Event-triggered flows

`trigger.type: event` binds an existing Business Event name to an existing Marketplace Flow. The event name is opaque metadata — the runtime does not hardcode domain events.

```yaml
trigger:
  type: event
  event: invoice.overdue
```

Dispatch (same Postgres event bus + FlowRunner as conversation/schedule — not a second bus, scheduler, workflow engine, or FlowRunner, and not Marketplace-specific Rust branches):

1. Entity mutation (RuntimeAdapter) emits a canonical envelope onto the existing bus.
2. CRM Automation, Goal Engine, and Catalog Service observe the same event (not merged into this path).
3. Resolution is workspace-scoped: only installed, enabled flows whose `trigger.event` equals the envelope name. No tenant-wide Marketplace scan. SDK event flows remain a separate subscriber path (priority unchanged).
4. FlowRunner starts with trusted refs: `event.id`, `event.name`, `entity`, `record_id` / `id`. The flow loads the record with `entity.*.get` — the full document is not auto-injected. Tenant comes from the record/envelope; workspace from trusted top-level `workspace_id`. Event facts are trusted.
5. Payload cannot override tenant, workspace, actor, or installation. `person_id` is a tenant-global Hub relation: customer channels inject session identity; staff/API may select an existing tenant Person after Hub lookup; event-triggered `entity.*.create` may copy `person_id` from a related record via field `inherit_from` after authority strip — never from payload/LLM.
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

Publish accepts exactly `solution::workflows::KNOWN_STEP_TYPES`. FlowRunner executes a narrower set — the two lists are not the same, so "it published" is not "it runs".

| Step type | Publishes | Executes | Key fields |
|-----------|-----------|----------|------------|
| `ask` | yes | yes | `field`, `message` (alias `prompt`), `choices` or `choices_from`, `title_field` / `title_fields`, `value_field`, optional `description_field`, `image_field`, `chip_prefix`, `choices_filter`, optional `empty_message`, optional `when` gate, optional `multiple`, `group_by`, `quantity`, `min_date`, `input_type`, `temporal_validation`, step-level `summary` (WhatsApp-surface only — see below) |
| `tool` | yes | yes | `tool`, `execution` (`runtime`\|`http`), `input_map`, `output`, optional `error_message` |
| `condition` | yes | yes | `when` (expression), `then` (target step id), `else` (target step id). Missing/blank target falls through to the sequential successor. |
| `challenge` | yes | yes | `via`, `identity` |
| `upload` | yes | yes | `field`, `message` |
| `delay` | yes | yes | `duration_seconds` (alias `seconds`) **or** `until` (RFC-3339) |
| `approval` | yes | yes | `message`, `approver` |
| `message` | yes | yes | `message` (alias `prompt`), supports `{{var}}` templates |
| `tag` | yes | yes | `tag` |
| `assign` | yes | yes | `field`, `value` |
| `activity` | yes | yes | `message` |
| `complete` | yes | yes | `message` / `prompt`; omit both to fall back to the default closing text (or the tool's `items_text`) |
| `branch`, `notify`, `handoff` | yes (legacy allowlist) | **no** — reaches FlowRunner's fallthrough and fails with `Unknown step type '<x>'` | — |

`entity_select` is **not** in `KNOWN_STEP_TYPES`: a package that declares it is rejected at publish with `unknown step type 'entity_select' in step '<id>'`. Note the asymmetry — the WhatsApp Flow **overlay compiler** (`domain::whatsapp_flow`) does handle `ask | entity_select` and compiles `source.entity` into an `entity.<id>.list` data tool, and its tests exercise it. So the key is understood by stage 3 but blocked at stage 1: it is available to platform-side flow definitions, not to Marketplace packages. For an app, build record pickers from `tool` (`entity.<id>.list`) + `ask` with `choices_from`.

Other rules the parser enforces: step `id`s pass `validate_package_id` and must be unique inside the flow; every `next`/`then`/`else` pointer must reference a declared step; a `tool` step whose target matches a platform storage namespace is rejected (ADR-003 — use app tools, persist via `ctx.storage` in an SDK app).

Every flow SHOULD end with a `complete` step. `next_step_id` returns `None` one past the last step, so a flow that runs off the end also finishes — but only `complete` carries a closing message.

### Rich ask fields: WhatsApp surface vs chat

`multiple`, `group_by`, `quantity`, `min_date`, `input_type`, `temporal_validation` and step-level `summary` are compiled by the **WhatsApp Flow overlay** into Meta form components (Dropdown, RadioButtonsGroup, checkboxes, date pickers with minimum-date validation). The generic chat/portal ask renderer does not consume them, so:

- declaring them is safe everywhere but only changes behaviour on the WhatsApp Flow surface;
- a flow that is *correct only* with them must be paired with `whatsapp_flow`, and its chat-path tests must not assert them;
- a field whose absence breaks the chat path belongs on the `ask` step itself (`choices_from`, `title_field`, `empty_message`), not on the overlay.

### `input_map` syntax

`input_map` values are **bare dotted paths or `$`-prefixed sources — never `{{…}}` templates.** `flow_engine::variables::build_tool_input` sends each value through `resolve_input_source`, which is a three-way match on the literal string:

```yaml
constants:
  default_status: scheduled
  page_size: 20

steps:
  - id: find_tables
    type: tool
    tool: entity.table.list
    execution: runtime
    input_map:
      status: "$literal:available"   # JSON-shaped literal keeps its JSON type
      capacity: "$literal:4"         # parses as the number 4, not "4"
      filter.capacity.$gte: min_guests   # dotted KEY nests; dotted VALUE reads a variable path
      limit: "$constants.page_size"  # flow-level constant
      table_id: table_id             # variable path, resolved by FlowVariableContext::get_path
      id: property_id
    output: tables                   # captured under this variable name
```

| Source form | Resolution | Notes |
|-------------|-----------|-------|
| `$literal:<raw>` | `parse_flow_literal` | Tries `serde_json` first: `20` → number, `true` → bool, `"x"` → string; anything unparseable stays a plain string. This is the only way to send a constant, and expand directives (`filter.*`) should be literals so they are not user-influenced. |
| `$constants.<path>` | Flow `constants:` map, dotted lookup | Trusted, design-time values. |
| `<dotted.path>` | `ctx.get_path(path)` | Conversation variables, collected answers, prior `output:` documents (`properties.items`, `person.id`). |

Three consequences that cause silent bugs:

- **Unresolved sources are dropped**, not nulls: `if let Some(v) = resolve_input_source(..)`. A typo in a path means the tool simply does not receive that parameter — assert on the built input in tests, not on the YAML.
- **`{{city}}` does not work here.** It is matched as a path whose segment is literally `{{city}}`, resolves to nothing, and the key disappears. `{{…}}` belongs to `message`, `prompt` and `complete` text (and to the tool/connector prompt machinery), which go through `resolve_template`.
- **Dots are structural on both sides.** The key is split with `set_dotted` (`filter.capacity.$gte` → `{"filter":{"capacity":{"$gte":…}}}`), so a tool parameter can never contain a literal `.`; the value is split with `get_path`, so entity/variable names containing dots are unreachable.

Array-valued and object-valued sources pass through whole — a `output: properties` document is available to later steps as `properties.items`, and `input_map: items: order_items` forwards the array.

### Condition grammar

`condition.when` and `ask.when` go through `domain::flow_execution::ConditionEvaluator`, which supports exactly one comparison per expression:

- `<path> ==|!=|>=|<=|>|< <literal>` — ordering operators require both sides numeric; equality falls back to string comparison.
- `<path> exists` / `<path> empty` — unary suffixes.
- `<path>` alone — truthiness (`Null`/missing/`false`/`0`/`""`/`[]`/`{}` are false).

No `and`/`or`, no parentheses, no function calls. Quoted right-hand sides strip their quotes; an unquoted, non-numeric right-hand side is compared as a string. A malformed or empty expression is an error, and the `condition` step's caller treats `Err` as `false` — so a broken `when` takes the `else` branch quietly rather than failing the flow.

`trigger.when` is **not** this evaluator. It is `domain::orchestration_event::eval_trigger_when`, run against the event payload rather than flow variables: same operator set and bare-path truthiness, plus an optional `payload.` / `event.payload.` prefix, and an empty expression matches. A missing left-hand path fails closed, so the flow is simply not started (no execution row, no error surfaced to anyone).

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
  empty_message: "There are no properties listed right now."  # always declare this
  choices_from: properties        # references the output variable
  title_field: title              # display label from each item
  value_field: id                 # value from each item
  chip_prefix: "Property"         # UI chip label
```

`empty_message` is the reply when the list comes back empty — nothing to
cancel, fully booked, no matches. The flow then completes gracefully and the
conversation stays with the AI. An ask step with `choices_from` and no
`empty_message` fails instead, which notifies the team of a handoff, so every
dynamic-choice ask must declare one.

### What `choices_from` actually does

`steps::choice_list_items` (the emptiness gate) and `steps::choices_from_context_var` (the chip list) read the same variable differently, and the difference is load-bearing:

- The **emptiness gate only fires for an object with an `items` array**. Tool `output:` variables are `{ items: […], total, … }`, so `output`-sourced lists get the `empty_message` / fail path. A variable that is a **bare JSON array** returns `None` from the gate, so `empty_message` never triggers for it — an empty bare array just produces a chip-less `ask`. Declare choices over tool outputs, or accept a free-text ask.
- Chips are capped at **10**, in source order, after `choices_filter` (an object of exact `field: value` equalities) drops non-matching items and after de-duplication on the resolved value. Anything beyond the tenth item is unreachable by chip, so a list that can exceed 10 needs a narrower `input_map` filter on the tool step — not a longer chip list.
- Titles come from `title_field`, or from `title_fields` (array) joined with `" · "`. With no `title_field`, the fallback order is `name` → `title` → `label` → `start_at`; if none is a **non-empty string** the item is skipped entirely (numbers, booleans and nulls never render). Chip labels are truncated to 24 characters (`21 + …`) because Meta list/row titles are short — keep the human-meaningful text in `title_field` under ~24 chars or compose it with `title_fields`.
- `value_field` defaults to `title_field`; `chip_prefix` prepends `"<prefix>:"` to the action id.

### What a chip answer writes back

Answering a `choices_from` ask is resolved in `executor.rs`, not by trusting the reply string. The reply is matched case-insensitively against each item's `value_field` (falling back to `id`, then `_id`), then against `title_field` (fallback `title`/`name`), then `code`, then prefix/substring overlaps on the title. The matched item then writes:

| Variable written | Value |
|------------------|-------|
| `<field>` | the item's canonical `value_field` / `id` / `_id` — i.e. the UUID, even when the customer answered with a title |
| `<title_field>` | the item's title, under the *title field's own name* |
| `<field>_code` | the item's `code`, when it has one |

So an ask with `field: table_id`, `title_field: table_name` and `value_field: id` leaves the flow holding `table_id` = UUID **and** `table_name` = the human label — which is why later `message` steps and `input_map` paths can reference either. No match, on a non-empty list, sets `_validation_error` ("I couldn't match that to one of the options…"), un-marks the field as collected, and **re-asks the same step**; it does not fail the flow.

Static `choices:` resolve to id+label only (`choice_item_to_action`) — none of the write-back above applies, so a static choice yields exactly the string the customer picked.

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

## Permissions vs capabilities — two different checks

```yaml
permissions:
  - workflow.execute
  - storage.read
  - storage.write
  - storage.update
  - storage.delete
capabilities:
  - theme.get
  - runtime.query
  - workflow.trigger
```

`permissions` is a **format** contract, not a closed vocabulary (`validate_permission`): exactly two dot-separated segments `resource.action`, `resource` = `a-z`/`_`/`-`, `action` = `a-z`/`_`/`*` (so `storage.*` is legal), and nothing else. Anything else is a publish error `invalid permission '<x>': expected \`resource.action\``. There is deliberately no allowlist — a new platform resource must not require editing this file to be requestable.

`capabilities` **is** a closed vocabulary (`is_known_capability`): the token must appear in `UI_HOST_CAPABILITIES` (`theme.get`, `user.get`, `tenant.get`, `workflow.trigger`, `connector.invoke`, `runtime.query`, `customer.query`, `storage.read|write|update|delete`, `marketing.read|write`, `organization.read|write|task.execute`) or `UI_PLANE_CAPABILITIES` (`runtime.read`, `workflow.execute`, `customer.read`, `storage.read|write|update|delete`, `marketing.read|write`, `organization.read|write|task.execute`). An unrecognised token is rejected at publish. Adding one is a platform change in `qefro-plugin-proto/src/ui.rs` first, and a consumer second.

`permissions` and `capabilities` are duplicate-checked separately; a workspace's effective grant is negotiated at install from metadata × role × channel × RBAC × identity × policy, so declaring both is not a contradiction and neither one grants anything by itself.

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

The seven capabilities above are what all current entity-native Marketplace Apps declare. That is the common **pattern**; the accepted **set** is closed — see *Permissions vs capabilities* above.

### Negotiation

`solution::ui::negotiate_capabilities` filters the requested list down to what the version's `permissions` (plus whether any connector is declared) actually back. Requesting a capability without the matching permission is legal at publish and **silently dropped at install**, which surfaces as a widget that renders but whose action does nothing:

| Capability | Granted when |
|------------|--------------|
| `theme.get`, `user.get`, `tenant.get`, `runtime.query` | always, for an installed UI |
| `workflow.trigger` | `permissions` includes `workflow.execute` |
| `customer.query` | `permissions` includes `customer.read` |
| `connector.invoke` | the manifest declares at least one connector |
| `storage.read` / `write` / `update` / `delete` | `permissions` includes the **identical** token |
| `marketing.read` / `write`, `organization.read` / `write` / `task.execute` | `permissions` includes the identical token |
| `runtime.read`, `workflow.execute`, `customer.read` | plane tokens — pass through when granted |

So `capabilities: [workflow.trigger]` with `permissions: [workflow.execute]` works; `capabilities: [storage.write]` with no `storage.write` permission does not. Declare the permission as well as the capability.

### Agent capabilities

A package MAY declare an `agent:` block (`enabled` + `roles.customer` / `roles.business`, each with `capabilities`, `tools`, `flows`). It is a **description**, not a grant: `KNOWN_AGENT_CAPABILITIES` tokens are checked at publish, `tools` must exist in `tools`/`http_tools`, and `flows` must exist in `flows`. Effective Agent access is still metadata × role × channel × workspace RBAC × identity × policy. Apps that declare no `agent:` block are still reachable by Agent through the runtime's entity capability projection and flow-based capabilities — see `agent-integration.md`.

Runtime entity operations: `entity.{name}.{create|list|get|update|delete|restore|availability|aggregate}`.
`aggregate` is read-only (`op: count|sum`) and is projected to Business Agent automatically.
`restore` requires update permission and `storage.update`. Soft delete uses `storage.update`;
hard/restrict delete uses `storage.delete`.

## Automation Templates

Installed Marketplace Apps may declare `automation_templates` in `manifest.yaml`. These are **presets** for existing CRM Automations (WHEN + IF + THEN). They are not executable, not a second engine, and not Agent/Goal templates.

```yaml
automation_templates:
  - id: overdue_invoice_reminder
    name: Remind customers about overdue invoices
    description: Create a follow-up when an invoice becomes overdue.
    trigger:
      event: invoice.overdue          # must be in manifest.events
    actions:
      - type: create_followup         # CRM AutomationAction vocabulary only
        due_in_hours: 24
        note: Collect payment on an overdue invoice
  - id: invoice_overdue_whatsapp
    name: WhatsApp overdue invoice reminder
    description: Message the workspace Person when an invoice becomes overdue.
    trigger:
      event: invoice.overdue
    actions:
      - type: send_whatsapp
        template: invoice_overdue     # Meta name: [a-z0-9_]+
        parameters:                   # positional Cloud API body params
          - "{{person.name}}"
          - "{{entity.invoice_number}}"
        body: |
          Hi {{person.name}}, your invoice {{entity.invoice_number}} is overdue.
  - id: send_invoice_created_webhook
    name: Send invoice-created to my system
    trigger:
      event: invoice.created
    actions:
      - type: send_webhook
        payload_mapping:              # mapping only — never URL/secret/headers
          - output_key: invoice_number
            event_parameter: invoice.invoice_number
```

### `send_whatsapp` template contract

WhatsApp Settings → Templates discovers `send_whatsapp` actions from **this workspace’s installed app** `automation_templates` (and instantiated CRM automations). It is the existing CRM `SendWhatsapp` action + workspace WhatsApp / WABA client — not a second template engine.

| Field | Required | Contract |
|-------|----------|----------|
| `type` | yes | `send_whatsapp` |
| `template` | yes for WABA provision | Meta Cloud API name `[a-z0-9_]+` |
| `parameters` | no | Ordered `{{person.*}}` / `{{entity.*}}` strings |
| `body` | no | Preview / fallback copy. Never a recipient phone |
| `delay_seconds` | no | Existing CRM delay |
| `media_url` / `media_id` | no | Existing CRM media fields |

Never declare recipient phone, WABA id, access token, or app secret in YAML. The workspace WhatsApp configuration owns those. Person is resolved from the CRM event / Hub, not from the template.

Reference `send_whatsapp` names: billing-pro `invoice_overdue`, clinic-pro `appointment_reminder`, restaurant-pro `order_ready`. An app with only `create_followup` (real-estate-pro) correctly shows “No WhatsApp templates required by installed apps yet.”

## Follow-up sequences (`manifest.followup_sequences`)

A sequence is a Goal-referenced WhatsApp template cadence, **not** an install-time campaign and **not** a CRM Automation. The Goal stores `followup_sequence_id`; the evaluator picks the step for the current attempt and executes it through the existing `SendWhatsapp` CRM action.

```yaml
followup_sequences:
  - id: lead_followup            # only id, name, description, channel, steps are allowed
    name: Lead follow-up
    description: Follow up with property leads until they respond.
    channel: whatsapp            # must be exactly "whatsapp"
    steps:                       # at least one
      - id: step_1               # only id, template, delay, parameters are allowed
        template: lead_followup_1
        delay:
          unit: hours            # minutes | hours | days — nothing else
          value: 24              # must be > 0
        parameters:
          - "{{person.name}}"
          - "{{entity.property_title}}"
```

**The publish check is weaker than the runtime check, and that is the trap.** `SolutionPackageManifest.followup_sequences` is `Vec<Value>` — the parser validates nothing. Validation happens in `domain::parse_marketplace_followup_sequences`, which runs **per Goal evaluation** against the chat context. It is fail-closed with `reject_unknown_object_keys` at all three levels (sequence, step, delay), but its callers do `.ok()?` and then only `warn!("failed to resolve follow-up step")`. So a misspelled key, an empty `template`, a `delay.unit: weeks` or `delay.value: 0` produces a package that **publishes and installs cleanly and then silently never sends** — no error reaches the portal, the customer, or the install status.

Consequences for authoring:

- Keep the closed key sets exactly: sequence `{id, name, description, channel, steps}`, step `{id, template, delay, parameters}`, delay `{unit, value}`. There is no `when`, `if`, `cancel_on`, or per-step gate — cancellation is the Goal's own success/wait conditions, not sequence metadata.
- Sequence ids and step ids must each be unique within their scope; `template` must satisfy `is_meta_whatsapp_template_name` (`[a-z0-9_]+`, same rule as `send_whatsapp`).
- Parameters must be non-empty strings; an empty entry is a parse failure for the **whole** sequence list.

Step selection uses `goal.attempt_count`: the template for an attempt comes from `resolve_followup_step(sequence, attempt_count + 1)` (1-indexed) and the next wake delay from `resolve_next_followup_delay(sequence, attempt_count)` (0-indexed) — both land on `steps[attempt_count]`, so attempt *n* sends step *n+1* and then sleeps for that step's own delay. Once `attempt_count` exceeds the declared steps, both return `None` and the Goal falls back to its deadline-derived timer. A sequence therefore caps the cadence: declare exactly as many steps as attempts you want.

`{{…}}` parameters resolve against a **synthetic** event payload built by the evaluator, containing:

- `goal_id`, `workspace_id`, `customer_id`, `attempt`, `template`
- `person.{id,name,email,phone,language}` — loaded from the Hub by `goal.customer_id`; absent when the Person lookup misses
- `entity.<field>` — the **whole record** fetched with `storage.get` on `goal.entity_type` + `goal.entity_id` (requires `storage.read`); absent when the goal has no `entity_type`/`entity_id`/installation

So `{{entity.X}}` is only valid if `X` is a field on the entity type the Goal was created against — not on any entity in the app. Real Estate Pro's `lead_followup` is a lead cadence: a lead has no property relation, so its parameters are `{{person.name}}` only. `property_name` and `listing_price` are not lead fields and must not be reintroduced. An unresolved path renders as an empty string, and the WhatsApp sender then refuses the step. Verify parameter paths against the **target entity's** declared fields, and treat an empty render as a bug in the metadata.

Rules:

- CRM discovers templates from **this workspace's installed app only** (one app per workspace).
- Selecting a template instantiates a normal `CrmAutomation`. Multiple instances of the same template are allowed and independent.
- Upgrading the app does **not** mutate already-created automations.
- Fail closed: unknown ids/events/action types, authority fields (`tenant_id`, `workspace_id`, `installation_id`, `person_id`, `customer_id`), secrets, URLs, `Authorization`, and executable code are rejected. Invalid templates must not create partial automations.
- `send_webhook` templates must not include destination URL, signing secret, headers, or `connection_id`. The workspace picks an outbound connection at instantiate/edit time.
- `send_whatsapp` templates must not include recipient address, WABA, or credentials.
- Do not add TemplateEngine2, FlowRunner2, WebhookEngine, JS/CEL, or cross-app aggregation.

## Conversation slots (`manifest.conversation_slots`)

The slot vocabulary is app metadata (`visit_type`, `guest_count`, …) — never runtime code. `ConversationSlot` accepts exactly:

| Key | Type | Contract |
|-----|------|----------|
| `id` | string, required | variable name the extraction writes |
| `labels` | string[] | synonyms the matcher looks for |
| `aliases` | string[] | additional match names |
| `kind` | string? | parser selection (table below) |
| `identity` | `person_name` \| `email` \| `phone` | marks identity-relevant extraction and links to Customer Hub resolution |
| `chip_prefix` | string? | UI chip label prefix |
| `chip_value` | `remainder` (default) \| `last_segment` | how a colon-bearing chip is cut |
| `chip_label_slot` | string? | another slot whose value labels this chip |
| `choices` | map `value → [phrases]` | closed value set for enumerated slots |
| `quantity_suffixes` | string[] | unit words that carry a numeric quantity |

Any other key is inert: `ConversationSlot` has no `deny_unknown_fields`, so a misspelled `chip_prefx` parses and does nothing.

`kind` selects the extractor:

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

Slots with `identity: person_name` link to Customer Hub identity resolution. Extraction is generic and metadata-driven: adding a new business concept means adding a slot here, not a runtime branch.

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
