# Metadata Contract

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

# Optional status-to-event mapping:
status_events:
  <status_value>: <event.name>
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

The runtime automatically injects `person_id` from the authenticated Customer Hub session. Apps never supply this value.

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

trigger:
  type: conversation            # or: event, schedule, webhook

steps:
  - id: <step_id>
    type: <step_type>
    # type-specific fields (see below)
```

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

## Permission set

```yaml
permissions:
  - workflow.execute
  - storage.read
  - storage.write
  - storage.update
  - storage.delete
```

## Capability set

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
