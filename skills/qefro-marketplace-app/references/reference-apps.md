# Reference Apps

## Source repository

**https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps**

When the local `qefro-marketplace-apps/` directory is available, read from it directly. Otherwise, browse or clone the public repository:

```bash
git clone https://github.com/qefro-ai/qefro-marketplace-apps.git
```

## Purpose

The [apps/](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps) directory is a **reference implementation corpus**. It demonstrates patterns, not domain logic.

**The runtime/platform contract is authoritative. Reference apps demonstrate patterns. They must NOT be blindly copied.**

## How to use reference apps

When building a new Marketplace App:

1. **Browse [apps/](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps)** for the closest 1-3 reference applications
2. **Identify the underlying Qefro pattern** (not the domain concepts)
3. **Verify that pattern against the current runtime** (check Rust source)
4. **Adapt it to the new business domain** (new entity names, new field names)
5. **Avoid copying domain-specific concepts** (property -> clinic is wrong)

## Reference app index

Two apps are the maintained, verified references for metadata-driven development: **restaurant-pro** (transactions, POS, staff UI) and **real-estate-pro** (catalog, search, scheduling, follow-ups). Read both before writing anything new — between them they cover nearly every primitive the platform accepts. The remaining apps are secondary references for a specific shape.

### Primary reference: restaurant-pro

| Pattern | File | What it proves |
|---------|------|----------------|
| Parent→child related-create prefill | `entities/order.yaml` → `ui.form.related_create.reservation_id` | Generic child-form defaults from the parent record, by explicit field map |
| Exclusive resource capacity | `entities/reservation.yaml` → `availability.booking.datetime_field: reservation_at`, `capacity.mode: exclusive`, `capacity.resource.field: table_id` | One booking per table per slot |
| Server-side line totals | `entities/order_item.yaml` → `computed: amount = quantity × unit_price` | Money never comes from the caller |
| Parent rollup | `entities/order.yaml` → `computed: total_amount = relation_sum(order_item, order_id, amount)` | Child mutations rewrite the parent total including 0 |
| `restrict` delete + optimistic concurrency | `entities/order.yaml` → `delete.mode: restrict`, `concurrency: optimistic` | Refusing deletion of referenced rows; version-checked updates |
| POS cart→document mapping | `ui/widgets.yaml` → `cart_checkout` options `menu_item` → `order_item` → `order` | Line-item checkout with no app-specific widget |
| Operating values out of entity YAML | `settings/business.yaml` | Timezone/hours/window are workspace settings, not structure |
| WhatsApp template automation | `automation_templates` → `order_ready` on `order.ready` | Template name declared in metadata, recipient stays platform-side |

### Primary reference: real-estate-pro

| Pattern | File | What it proves |
|---------|------|----------------|
| Conversational funnel with re-queried choices | `workflows/request-viewing.yaml` | `ask` → `tool` → `ask` narrowing, with `empty_message` on every dynamic ask |
| Availability as a flow primitive | same file, `entity.viewing.availability` × `kind: dates`/`slots`/`validate` | Date picker → slot picker → server revalidation before create |
| Structural exclusivity choice | `entities/viewing.yaml` → `capacity.resource.field: property_id` | Property-level exclusivity is an app decision, declared not guessed |
| Catalog overlay | `entities/property.yaml` → `catalog:` (`type: service`, mapping, `publish_when`) | Existing entity presented in WhatsApp Catalog; no second entity type |
| Multi-surface staff UI | `ui/{sources,pages,navigation,widgets,layouts}.yaml` | Declarative dashboard over entity sources |
| Install settings both shapes | `manifest.yaml` `settings:` list + `settings/business.yaml` mapping | Manifest allowlist vs operative schema |
| WhatsApp Flow overlay | `workflows/request-viewing.yaml` → `whatsapp_flow.category: APPOINTMENT_BOOKING` | Same FlowRunner, Meta-rendered form |
| Goal-referenced cadence | `manifest.yaml` `followup_sequences: lead_followup` | Multi-step WhatsApp follow-up driven by the Goal engine |
| Rescheduling | `workflows/reschedule-viewing.yaml` | Validate a new slot against the existing record |
| Read-only search flow | `workflows/search-properties.yaml` | A flow that answers without writing |

### Secondary references

**[clinic-pro](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/clinic-pro)** -- appointment workflow with date/time collection; choices_from for staff and service selection; patient entity with `person_id`; status events; `whatsapp_flow` on `book-appointment`; catalog fixture `tests/fixtures/catalog/care_service.yaml`; `send_whatsapp` `appointment_reminder`.
**Pattern value:** scheduling flows with temporal slots and dynamic choices.

**[appointment](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/appointment)** -- generic appointment scheduling; service + staff entities; time slot collection via conversation slots.
**Pattern value:** a reusable scheduling pattern without domain coupling.

**[billing-pro](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/billing-pro)** -- invoice / payment / allocation entities, overdue events, `relation_caps`, `send_whatsapp` `invoice_overdue`.
**Pattern value:** money integrity (receipt vs allocations) and partial payment.

**[education](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/education)** -- course, enrollment, student; many-to-one relations; customer-scoped enrollment with `person_id`.
**Pattern value:** registration patterns with entity relations.

**[field-service](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/field-service)** -- work order with technician relation, site entity, customer-scoped work orders, catalog fixture `tests/fixtures/catalog/job_service.yaml`.
**Pattern value:** field service with customer data isolation.

**[shopify](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/shopify)** / **[stripe](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/stripe)** -- connection OAuth/bearer metadata, HTTP tools, webhook topic mappings and HMAC verification.
**Pattern value:** external API integration without shipping code.

## Known gaps in the reference apps

These are real defects in the corpus. Copying them reproduces a bug, not a pattern — and `scripts/validate_apps.py` does not currently catch them.

| App | File | Gap |
|-----|------|-----|
| restaurant-pro | `entities/customer.yaml` | Deprecated `customer` entity kept alongside Hub `person`; do not model customers as app entities in new apps |
| restaurant-pro | `workflows/update-order-status.yaml` | Patches `order`, which declares `concurrency: optimistic`, **without** sending `version` / `expected_version` — works only while the optimistic check is not enforced on that path |
| both | any `availability.capacity.mode: limited` | Not enforced by the V1 runtime — `require_valid_config` fails closed with an explicit error; use `exclusive` or `unlimited` |
| any app | `type: entity_select` steps | Rejected at publish (`unknown step type`) even though the WhatsApp Flow compiler supports it; use `tool` + `ask`/`choices_from` |
| older apps | unplaced `type: form` / `table` widgets | A widget not referenced by any page layout parses and simply never renders |

Verify before relying on any of the above being fixed, and check the current YAML rather than this table.

### HTTP-backed integrations

**[shopify](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/shopify)** -- demonstrates:
- Connection with OAuth metadata
- HTTP tools with webhook topic mappings
- Event emission from tool responses

**Pattern value:** How to integrate an external API with OAuth and webhooks.

**[stripe](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps/stripe)** -- demonstrates:
- Payment integration with bearer auth
- HTTP tools for charges and subscriptions
- Webhook HMAC verification

**Pattern value:** How to handle payment provider integration safely.

## Pattern extraction rules

When studying a reference app:

1. **Extract the structural pattern**, not the domain vocabulary
   - Good: "entity with `scope: customer` + `type: person` field + `allocate_code`"
   - Bad: "copy the property entity with title, bedrooms, bathrooms"

2. **Extract the flow pattern**, not the business logic
   - Good: "ask step with choices_from referencing a prior tool output"
   - Bad: "copy the book_viewing flow with property_search step"

3. **Extract the event pattern**, not the domain events
   - Good: "status_events mapping entity status enum to dot-notation events"
   - Bad: "copy appointment.confirmed and appointment.cancelled events"

4. **Verify against runtime**, not against other reference apps
   - The runtime source is authoritative
   - Reference apps may contain patterns that predate current runtime capabilities

## Domain adaptation example

Building a new appointment app after studying restaurant-pro's `reservation` and real-estate-pro's `viewing`:

| Reference pattern | New app adaptation |
|---|---|
| `reservation.availability.capacity.resource.field: table_id` | `resource.field: room_id` (per-room exclusivity) or omit `resource` (one booking at a time overall) |
| `viewing.booking.datetime_field: scheduled_at` | same structural role, any field name — `starts_at` |
| `viewing: capacity.mode: exclusive` + settings `blocking_statuses` | keep `exclusive`; choose **this** business's blocking statuses |
| `request-viewing` `dates` → `slots` → `validate` → `create` | same step shape, new entity/tool names |
| `order.related_create.reservation_id.prefill` | `appointment.related_create.patient_room_id.prefill` — explicit maps only |
| `lead_followup` sequence | new sequence id, parameters citing only fields on the Goal's `entity_type` |

The **structure** is shared. The **domain vocabulary** differs. The **runtime execution** is identical, and the operating values (timezone, hours, window, blocking statuses) move to `settings/business.yaml` rather than being re-declared per entity.
