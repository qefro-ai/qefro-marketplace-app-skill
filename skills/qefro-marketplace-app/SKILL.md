---
name: qefro-marketplace-app
description: Generate, extend, and audit metadata-only Qefro Marketplace Apps as YAML (manifest, entities, workflows, staff UI, availability, line items, related-create, import, automation, WhatsApp Flow, WhatsApp Catalog). Use when the user asks to build, scaffold, or generate a Marketplace App, vertical solution, booking app, catalog app, or HTTP integration such as a clinic, salon, restaurant, school, field service, or Shopify-style connector. Use Restaurant Pro and Real Estate Pro as pattern references, never as runtime special cases.
---

# qefro-marketplace-app

Generate Qefro Marketplace Apps as YAML metadata. The Qefro Runtime executes that metadata. Do not write app-specific Rust, Go, Python, or TypeScript.

```
Entity -> Capability -> Flow -> Event -> UI / conversational behavior
```

Write the app into `qefro-marketplace-apps/apps/<app-id>/`. From this skill repo that path is `../qefro-marketplace-apps/apps/<app-id>/`. If the apps repo is missing, clone `https://github.com/qefro-ai/qefro-marketplace-apps.git` beside this repo. Never put generated apps inside the skill repo. Do not publish, deploy, or change live data unless the user explicitly asks.

## Generate a new app

1. **Classify the app.** Entity-native apps store records in Qefro (`entities/` + `workflows/` + `ui/`). HTTP-backed apps call an external API (`connections/` + `tools/` + `workflows/`). If the request names the business, proceed. Ask only when the business, the customer job, or the staff job is missing.

2. **Pick one primary pattern and read it.** Transactions, carts, and parent/child money follow Restaurant Pro. Search, catalog, booking, and follow-ups follow Real Estate Pro. External APIs follow Shopify or Stripe. Read the matching files in [Reference apps](references/reference-apps.md), then the headings you will use in [Metadata contract](references/metadata-contract.md). Copy the structure. Rename entities and fields for this business. Do not copy domain words (property, table, menu) into a different business.

3. **State the design before writing files.** App id (lowercase, hyphens), entities, customer flows, staff flows, and which of these apply: availability, line items, related-create, catalog, WhatsApp Flow, automation, install settings. Mark the rest not applicable.

4. **Write files in this order.** Every id in `manifest.yaml` must match a file, and every file must be listed.

```
apps/<app-id>/
  manifest.yaml              # hosting: runtime, inventories, events, permissions, capabilities
  settings/business.yaml     # only when hours, slots, timezone, or other install values exist
  entities/<id>.yaml         # entity-native
  workflows/<id>.yaml        # every app; last step type: complete
  ui/pages.yaml              # entity-native staff UI
  ui/navigation.yaml
  ui/sources.yaml
  ui/layouts.yaml
  ui/widgets.yaml
  ui/theme.yaml
  connections/<id>.yaml      # HTTP-backed only
  tools/<id>.yaml            # HTTP-backed only
  README.md
```

5. **Validate, then fix, then report.** From the apps repo:

```bash
python3 scripts/validate_apps.py
python3 -m pytest tests/test_packages.py -q
```

A green YAML check does not prove the user journey. Say which reference pattern each file follows, what validation returned, and any metadata the runtime does not execute. See [Validation](references/validation.md).

## Metadata you may write

Read the matching heading in [Metadata contract](references/metadata-contract.md) before inventing a key. A key with no consumer is dropped or inert.

- **Manifest:** `id`, `name`, `version`, `hosting: runtime`, `description`, `category`, `channels`, `entities`, `flows`, `events`, `permissions`, `capabilities`. Optional: `settings`, `followup_sequences`, `automation_templates`, `conversation_slots`, `agent`, navigation layout.
- **Entity fields:** `string`, `integer`, `float`, `boolean`, `date`, `datetime`, `email`, `phone`, `url`, `uuid`, `enum` (with `enum_values`), `json`, `person` (`ref_entity: person`), `relation` (`ref_entity`), `image`, `media`. Customer-owned records use `type: person`, not a copied `customer` entity.
- **Integrity:** `required`, `unique`, `allocate_code`, `computed`, `relation_caps`, `status_events`, `on_events`, `delete.mode` (`hard` | `soft` | `restrict`), `concurrency: optimistic` (updates must send `version`).
- **Booking:** `availability` on the entity. Capacity mode is `exclusive` or `unlimited`. Operating hours live in `settings/business.yaml`.
- **Money:** child `computed` line amount, parent `relation_sum`. Never trust a total typed by the widget or the model.
- **Related create:** explicit `ui.form.related_create` field maps only. Do not copy status, codes, or dates by matching names.
- **Flows:** `ask`, `tool`, `condition`, `message`, `complete` (also `challenge`, `upload`, `delay`, `approval`, `tag`, `assign`, `activity` when the contract says they execute). Dynamic choices are `entity.<name>.list` or `entity.<name>.availability`, then `ask` with `choices_from` and `empty_message`. `input_map` values are prior fields, `$literal:`, or `$constants.`.
- **WhatsApp Flow overlay:** on the workflow, keys `category`, `title`, `cta` only. Categories: `SIGN_UP`, `SIGN_IN`, `APPOINTMENT_BOOKING`, `LEAD_GENERATION`, `CONTACT_US`, `CUSTOMER_SUPPORT`, `SURVEY`, `OTHER`.
- **Catalog overlay:** on the entity. `type` is `product` or `service`. `mapping.*` values are field names on that entity.
- **Automation presets:** `automation_templates` in the manifest. `send_whatsapp` template names match `[a-z0-9_]+`. No URLs, secrets, recipients, or connection ids.
- **Staff UI:** widget kinds and icons are closed sets. Read [Validation](references/validation.md) "Publish-stage closed sets" before adding a widget.

Entity-native tool names are `entity.<name>.<create|list|get|update|delete|restore|availability|aggregate>` with `execution: runtime`.

## Extend or audit an existing app

Read the app's YAML and the closest reference before editing. Change metadata only. Re-run the package validator and name the user journey you did not execute (browser, WhatsApp, live tenant).

For an audit, walk the coverage list below and report gaps against the contract. Do not rewrite the app unless the user asked for a fix.

## Coverage

For each app, implement or mark not applicable:

- Manifest identity, inventories, events, permissions, capabilities, channels, slots, settings, follow-ups, automation, navigation
- Entities: types, constraints, relations, Person, codes, import defaults, computed values, status events, delete and concurrency
- Availability, line items, and related-create when the business has booking, money, or parent/child records
- Staff UI pages, navigation, sources, layouts, widgets, theme
- Flows: surface, trigger, tools, dynamic choices, empty states, confirmation, cancel or reschedule
- Optional WhatsApp Flow, Catalog, and CRM presets
- Validation result and any unsupported key

## Do not

- Put business logic in the Qefro runtime, or add `if solution ==` / `if entity ==` branches
- Add a second FlowRunner, dispatcher, catalog service, or WhatsApp engine
- Invent field types, step types, or API syntax. `entity_select`, `branch`, `notify`, and `handoff` are not executable Marketplace steps
- Put tokens, URLs, tenant ids, workspace ids, or secrets in YAML
- Let the model set `tenant_id`, `workspace_id`, `installation_id`, `person_id`, or `version`
- Turn a catalog view or cart into a business record without an explicit confirmed workflow
- Use `availability.capacity.mode: limited` (not enforced; the runtime fails closed)
- Declare `hosting: custom`

## Where to read next

- [Architecture](references/architecture.md) — entity-native vs HTTP-backed, file map
- [Metadata contract](references/metadata-contract.md) — keys that publish and survive install
- [Runtime execution](references/runtime-execution.md) — FlowRunner, `input_map`, `choices_from`
- [Identity](references/identity.md) — Customer Hub Person
- [Security](references/security.md) — authority stripping and isolation
- [Agent integration](references/agent-integration.md) — capability projection
- [Validation](references/validation.md) — commands and publish closed sets
- [Reference apps](references/reference-apps.md) — Restaurant Pro and Real Estate Pro file map

Source of truth, in order: current runtime in `ai-customer-support/`, publish/install in `qefro-plugin-platform/`, portal UI in `ai-customer-support-portal/`, then `qefro-marketplace-apps` tests and apps, then these references. If a doc disagrees with the code, follow the code and say so.
