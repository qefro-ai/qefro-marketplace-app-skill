# Identity -- Customer Hub

## Person entity

**File:** `ai-customer-support/crates/domain/src/person.rs`

The Person is the unified customer memory record. Canonical identity for all Marketplace Apps.

### Key fields

`id` (UUID), `tenant_id`, `workspace_id`, `name`, `email`, `phone`, `status`, `assigned_to`, `merged_into`, `conversation_count`, `source`, `language`, `timezone`, `avatar_url`

### PersonStatus lifecycle

```
Anonymous -> Lead -> Qualified -> Customer -> Inactive
```

### PersonIdentity

Binds `(channel, identifier)` to a `person_id`, scoped by `tenant_id + workspace_id`.

Unique constraint: `(tenant_id, workspace_id, channel, identifier)`

### PersonChannel enum

`Whatsapp`, `Website`, `Email`, `Instagram`, `Telegram`, `Sms`, `Manual`, `Api`

### Identity priority (merge precedence)

`IDENTITY_PRIORITY`: Whatsapp > Email > Sms > Instagram > Telegram > Website > Api > Manual

## Identity resolution

**File:** `ai-customer-support/crates/db/src/persons.rs`

### Normalization

- `normalize_email()` -- trim + lowercase
- `normalize_phone_e164()` -- strips trunk `0`, applies Indian mobile heuristic (10 digits starting 6-9 -> `+91`), returns `+digits`
- `normalize_identifier()` -- dispatches to email/phone/trim based on channel

### Resolution flow

1. `find_by_identity()` -- iterates phone identity candidates, updates `last_seen_at`, heals legacy identifiers to canonical E.164
2. `upsert_identity()` -- ON CONFLICT upsert on unique identity constraint
3. `merge()` -- full transactional merge: migrates all child records (identities, attributes, notes, tags, activities, conversation links), marks `loser.merged_into = survivor`

## PersonService

**File:** `ai-customer-support/crates/api/src/services/person.rs`

### Key methods

- `require_phone_or_email()` -- every Person must have phone or email
- `resolve_or_create(channel, identifier)` -- canonical inbound resolution
  - WhatsApp/Sms -> sets phone
  - Email -> sets email
  - Website/Api/Instagram/Telegram/Manual -> **lookup only**, never creates from session identity alone
- `resolve_and_link_conversation()` -- resolve_or_create + link conversation to Person
- `capture_lead_from_widget()` -- widget lead capture -> Person with `status=Lead`
- `merge()` -- idempotent (checks `merged_into`), emits `person.merged` event
- `emit_person_event()` -- emits bus events: `person.created`, `person.updated`, `person.status.changed`, `person.merged`, `person.assigned`

## person_id injection into Marketplace Apps

### Inbound chat attach

**File:** `ai-customer-support/crates/api/src/services/person_attach.rs`

- `attach_whatsapp_person()` -- phone-based attach for WhatsApp
- `attach_chat_person()` -- email-only attach for non-WhatsApp channels
- `seed_whatsapp_known_customer()` -- loads Hub profile, seeds conversation vars (phone, customer_phone, customer_name, name, email)

### Flow execution injection

In `runtime_adapter.rs` `execute_storage()`:

1. Check if entity has `scope: customer` + `type: person` field (C1 contract)
2. If customer-scoped on a customer channel: inject `person_id` from `auth_ctx.person_id`
3. Fail closed: if identity is missing on a customer channel, reject the operation
4. Portal/staff API: caller may select an existing Hub Person; runtime stores it
   only after tenant+workspace Hub lookup. LLM/agent payloads are still stripped.
5. Auto-fill customer name fields from Person record

### SDK tool injection

Before invoking an SDK tool, the runtime loads a Person snapshot:
- `load_sdk_person_snapshot()` -- falls back from `auth_person_id` -> conversation.person_id -> None
- `build_sdk_person_snapshot()` -- Identity kind (minimal) or full kind (with status, attributes, tags, activities)

## Ownership and visibility contract (C1)

### Three-tier visibility model

| Tier | Condition | Customer channels | Portal channel |
|------|-----------|-------------------|----------------|
| **Customer-scoped** | `scope: customer` + `type: person` field | person_id injected/filtered; fail-closed without identity | Bypass — sees all workspace records |
| **Shared/Default** | No `scope: customer` declaration | No person filter; visible to all | No person filter |
| **Staff/Default** | No `scope: customer` (even with person fields) | No person filter | No person filter |

The model is binary at the metadata level (`scope: customer` vs absent) with a runtime bifurcation based on identity channel.

### Detection function

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs`

`entity_person_field()` returns `Some(field_name)` only when BOTH conditions are met:
1. Entity declares `scope: customer`
2. Entity has at least one field with `type: person`

Returns `None` for entities without `scope: customer`, even if they have `type: person` fields.

### Portal bypass

Portal users (business owners, `IdentityChannel::Portal`) bypass person-scoping entirely — they manage all workspace records. The check:

```rust
let is_portal = auth_ctx.verified_identity
    .as_ref()
    .map(|vi| matches!(vi.channel, domain::IdentityChannel::Portal))
    .unwrap_or(false);
```

When `is_portal` is true, person_id injection and filtering are skipped.

### Entity scope in Agent capability projection

**File:** `ai-customer-support/crates/api/src/agent/entity_capabilities.rs`

```rust
pub enum EntityScope {
    Default,   // Staff/admin-scoped
    Customer,  // Runtime injects person_id filtering
}
```

Parsed case-insensitively from entity metadata `scope` field. Defaults to `Default` when absent.

### HTTP tool ownership (response-level filtering)

**File:** `ai-customer-support/crates/api/src/flow_engine/http_policy.rs`

For HTTP-backed tools, a separate ownership spec filters response records post-hoc:

```yaml
ownership:
  identity: person.email          # which identity value to compare
  paths: [email, customer.email]  # which record fields to match against
  also:                           # additional OR clauses
    - identity: person.phone
      paths: [phone_number]
```

`apply_ownership()` filters list responses to only records owned by the caller. Identity values come from `HttpIdentityBindings` (server-resolved from Customer Hub, never from LLM). Phone matching uses digit-only comparison.

### Surface enforcement (ToolSurface)

```rust
pub enum ToolSurface {
    Staff,     // Portal
    Customer,  // WhatsApp, Widget, API
}
```

Tools declare allowed surfaces in metadata: `{ "surfaces": ["customer"] }`. Empty/missing means allowed everywhere (backward compatible).

### Installation-scoped App RBAC

**File:** `ai-customer-support/crates/api/src/app_rbac_authz.rs`

Additional authorization layer on top of ownership:
- Applies only to Portal staff and API callers with portal user UUIDs
- WhatsApp/Widget customer channels bypass app RBAC (they use C1 person-scoping)
- Requires a non-nil `installation_id`
- Checks `{entity}.{view|create|update|delete}` permissions
- Fail-closed: if RBAC catalog cannot be loaded, permissions default to empty

## Identity in conversation variables

Person context flows into conversation variables:
- `person_id` -- UUID
- `customer_name` / `name` -- from Person.name
- `customer_phone` / `phone` -- from Person.phone
- `email` -- from Person.email

These are available in `{{variable}}` references in flow steps.

## External identities

**File:** `ai-customer-support/crates/db/migrations/20260819200000_person_external_identities.sql`

Maps Person to external business customer IDs. Unique constraints enable bidirectional lookup. Used by SDK tools to link Marketplace App customer records to Hub Person.

## Identity lookup for business tools

**File:** `ai-customer-support/crates/api/src/tool_executor/identity_lookup.rs`

`resolve_lookup_attributes()` fills conversation_vars from channel identity:
- Precedence: verified identity -> conversation vars -> tool input -> ask user
- `customer_id` must NOT be person UUID or phone number (validated via `domain::usable_external_customer_id()`)

## Rules for Marketplace Apps

1. Do NOT create a second identity system inside a Marketplace App
2. Do NOT ask users for identity information that Qefro already resolves
3. Do NOT allow the LLM to supply authoritative person IDs
4. Use `type: person` + `ref_entity: person` for customer identity binding
5. Use `scope: customer` for customer-facing entities
6. Trust the runtime to inject person_id automatically
