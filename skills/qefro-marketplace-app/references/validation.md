# Validation

## Source repository

**https://github.com/qefro-ai/qefro-marketplace-apps**

## Running the validation suite

```bash
# Clone if not available locally:
git clone https://github.com/qefro-ai/qefro-marketplace-apps.git

# From the qefro-marketplace-apps directory:
python3 scripts/validate_apps.py

# Or directly with pytest:
cd qefro-marketplace-apps && python3 -m pytest tests/test_packages.py -v
```

## Test suite structure

**File:** [tests/test_packages.py](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/tests/test_packages.py). Use the current suite rather than a frozen line/test count.

### Relevant test classes

- `PackageConventionTests`, `NewAppCoverageTests`, `CompatibilityContractTests` -- structure, field/step/trigger compatibility and app coverage
- `RestaurantProTests` -- booking/takeaway flows, item metadata, reservation availability and related-order defaults
- `RealEstateAvailabilitySettingsTests`, `RealEstateDomainTests` -- install settings, booking/rescheduling, property/viewing/lead relationships
- `CatalogOverlayContractTests`, `AutomationTemplateTests` -- shared catalog mapping and CRM preset safety
- `MarketplaceArchitectureTests` -- absence of embedded app runtimes

Package tests check declared metadata, not all rendered or executed behavior. Add regression tests for a new contract in the actual generic runtime/portal consumer as well.

## What the validation suite checks

### Structural conventions

| Check | Description |
|-------|-------------|
| `hosting: runtime` | All apps must declare runtime hosting |
| No secrets | YAML files must not contain API keys, tokens, passwords |
| No code files | No `.py`, `.js`, `.ts`, `.go`, `.rs` source files allowed |
| Entity field types | Must be from the allowed set |
| Workflow shape | Every flow must end with a `complete` step |
| WhatsApp Flow overlay | If `whatsapp_flow` is present: documented Meta category; no secrets/authority/URLs |
| Catalog overlay | If `catalog` is present: `type` product\|service; `mapping.title` required and names a declared field; no tokens/WABA/catalog/authority keys |
| send_whatsapp templates | Template name `[a-z0-9_]+`; no recipient/WABA/credentials in YAML |
| Allocate code | Prefix 2-4 characters, start must be numeric |
| UI conventions | UI files must follow naming and structural conventions |

### Security validations

| Check | Description |
|-------|-------------|
| Entity field safety | Forbids fields named `workspace_id`, `tenant_id`, `installation_id`, `organization_id` |
| HTTP tool placeholder safety | `IDENTITY_OVERRIDE_KEYS` check prevents identity override |
| Cross-tenant authority | No package declares `workspace_id:` or `tenant_id:` as literals |
| Customer identity scoping | Only `person.email`, `person.phone`, `person.external_id` permitted |
| Connection safety | Forbids `url`, `base_url`, `access_token`, `api_key`, `secret`, `password` |
| Embedded secret scan | Regex for `sk_live_`, `sk_test_`, `whsec_`, `xoxb-`, etc. |
| LLM identity override | Customer tools cannot have unqualified LLM params matching identity keys |
| Customer tool identity | Customer-facing tools MUST have `identity.require_any` and `ownership` |
| Webhook metadata | HMAC/signature config must be metadata-only |
| Flow constants | `$literal:` prefix must be used correctly |

## Publish-stage closed sets

These gates are enforced by the package parser, not by `validate_apps.py`. Local suite green + publish red is the usual symptom of missing one. Paths are relative to `qefro-plugin-platform/`.

| Gate | Enforced at | Rejects |
|---|---|---|
| Step types | `services/solution-service/src/solution/workflows.rs:180` vs `KNOWN_STEP_TYPES` (15 entries) | any other `type:` — including `entity_select`, which the WhatsApp Flow overlay compiler understands but publish never sees |
| Flow trigger | `workflows.rs:149` | no `trigger.type` and no `event`/`cron` to infer one; `event` trigger without `trigger.event`; `schedule` without `trigger.cron` |
| Flow shape | `workflows.rs:165,174,202,212` | empty `steps`; duplicate step id; `next` pointing at an undeclared step; no `complete` step anywhere |
| Flow constants | `workflows.rs:217,227` | non-alphanumeric constant key; non-scalar constant value; `input_map` `$constants.<x>` with no such constant |
| ADR-003 storage ban | `workflows.rs:189`, `ui.rs:238` | `tool` step or `ui` source whose target is a forbidden platform `storage/*` name |
| Identity authority in `input_map` | `workflows.rs:247` | a source path that is exactly `workspace_id`, `tenant_id`, `person_id` or `customer_id` |
| Manifest setting types | `manifest.rs:653` | anything outside `string \| number \| boolean \| color \| url` on `manifest.yaml` `settings:`. `settings/business.yaml` is parsed by a **wider** grammar (`settings.rs:42`, also `bool`, `integer`, `float`, `json`, `object`, `array`) |
| Install values | `settings.rs:139` | required setting missing with no default; value failing its declared type; unknown setting key (host `navigation_layout` and `_`-prefixed keys exempt) |
| Manifest capabilities | `manifest.rs:645` -> `qefro_plugin_proto::is_known_capability` | a token outside the closed host-capability set |
| Agent capabilities | `manifest.rs:160` `KNOWN_AGENT_CAPABILITIES` | a token outside: `knowledge, orders, analytics, customer_insights, inventory, crm, customers, reservations, bookings, promotions, pricing, refunds, properties, listings, leads, viewings, enquiries`. `agent.roles.*.tools` must exist in `tools`/`http_tools`; `.flows` must exist in `flows` |
| Permissions | `manifest.rs:830` | not exactly two `.`-separated segments; `resource` outside `[a-z_-]`; `action` outside `[a-z_*]` |
| Automation templates | `manifest.rs:110,122` (`deny_unknown_fields`) | any extra key on `automation_templates:` or its `trigger:` — a typo there fails the package, unlike `manifest.yaml` at large |
| Widget kinds | `ui.rs:202` vs `crates/qefro-plugin-proto/src/ui.rs:12` | kinds outside `metric, table, chart, kanban, calendar, status, map, timeline, activity, markdown, form, booking_link, cart_checkout` |
| Form field types | `ui.rs:431` vs `UI_FORM_FIELD_TYPES` (proto `ui.rs:33`) | `text, textarea, number, date, time, select, image, media` only; missing `type` defaults to `text`; non-string `type`, non-boolean `required`, `select` without `options`/`options_source`, non-string options, `options_source` not declared in `ui.sources` |
| `form` widget action | `ui.rs:378,501,510,521` | a `form` widget that declares a `source`; missing `options.action`; action without `trigger` or `emit`; `action.trigger` naming a workflow absent from `manifest.flows` |
| Icons | `ui.rs:18` `ALLOWED_ICONS` | 20-name set only (`home, calendar, chef-hat, receipt, users, user, file-text, bar-chart, settings, package, clipboard, credit-card, activity, list, clock, map, stethoscope, wallet, report, workflow`) |
| Host pages | `ui.rs:141` vs `UI_HOST_PAGES` | `page.host` outside `contacts \| automations \| roles` |
| Source kinds | `ui.rs:221` | `source.kind` outside `runtime \| connector \| entity`; empty `target`; `connector` target not shaped `{connector}/{operation}` |
| Layout | `ui.rs:311` | `layout.kind != grid`; columns outside 1-12; widget span outside 1-12; page referencing an undeclared layout preset or widget; duplicate page/widget/navigation id; nav or route pointing at an undeclared page |
| `navigation_layout` | `ui.rs:117` | value outside `topbar \| sidebar` |
| Code in `ui/` | `ui.rs:43` `FORBIDDEN_UI_EXTENSIONS` | 16 executable/template extensions under `ui/` |

Two asymmetries to remember:

1. `KNOWN_STEP_TYPES` is **wider** than what FlowRunner executes (`branch`/`notify`/`handoff` publish clean and fail at runtime), and **narrower** than what the WhatsApp Flow overlay compiler accepts (`entity_select`). A green publish proves nothing about either.
2. `manifest.yaml` is closed *structurally* but has no `deny_unknown_fields` — a typo'd manifest key is silently dropped and never reaches the consumer. Only the per-DTO tables in `metadata-contract.md` say which keys survive.

## Compliance checklist

For every Marketplace App implementation, verify where applicable:

```
[ ] Manifest valid (hosting: runtime, all required fields present)
[ ] Entities valid (field types from canonical set, person_id for customer-scoped)
[ ] Relations valid (ref_entity points to existing entity)
[ ] Capabilities valid (resource.action format; effective grant/permission checked, no closed app vocabulary)
[ ] Flows valid (every flow ends with complete step)
[ ] whatsapp_flow overlay (if used) is thin, documented category, no secrets — persisted after publish
[ ] catalog overlay (if used) is on the entity, product|service, mapping fields exist, no tokens/WABA ids — same mapper for product and service
[ ] catalog inquiry/order does not implicitly create Lead/Viewing/Order
[ ] send_whatsapp automation templates (if used) use Meta-legal names and no recipient/credentials
[ ] Events valid (dot-notation, referenced by status_events)
[ ] Permissions valid (from known permission set)
[ ] Conversation slots valid (kind from known set)
[ ] choices_from works (output variable referenced by ask step, empty_message declared)
[ ] Customer identity works (person_id auto-injected, not hardcoded)
[ ] Runtime execution works (entity.* tools resolve correctly)
[ ] Tenant/workspace/installation isolation works (no authority fields in params)
[ ] RBAC works (permissions declared, entity ops gated)
[ ] Agent read-only capability projection works (list + get tools generated)
[ ] No unauthorized LLM parameters (authority fields stripped)
[ ] No domain-specific runtime hardcoding (no if solution == "X")
[ ] Existing tests pass (python scripts/validate_apps.py)
[ ] Publish-stage closed sets clear (see "Publish-stage closed sets" above; the local suite does not check them)
[ ] Realistic end-to-end journey validated (conversational path works)
```

## Metadata development regression matrix

Use Restaurant Pro and Real Estate Pro as two different domains, not two sets of platform branches. Also test a renamed custom entity/FK/field fixture where a generic contract changes.

| Area | Required checks when touched |
|---|---|
| Manifest/settings | Inventory IDs resolve; schema defaults and install values have compatible types; optional feature keys survive publish/install projection |
| Entity integrity | Required/enum/unique validation; defaults; computed values after create/update/delete; optimistic conflicts; restrict/soft-delete behavior |
| Identity/relations | Customer session Person is injected; staff binds only an existing tenant Person; wrong-tenant and inaccessible relations fail through real execution |
| Availability | App settings drive hours/slots; no availability returns a useful reply; cancellation frees capacity; rescheduling excludes the current record; booking revalidates the selected slot |
| Line items | Item IDs/quantities validated server-side; prices/totals are not trusted from submitted UI/channel fields; empty/invalid selections and failed save/retry covered |
| Related creation | Open from parent, view configured context, edit defaults, save exact document; fixed FK, no copied parent status/code or unrelated date; cancel/reopen, missing values, relation clear/change, standalone behavior |
| Related-create safety | Unknown/mismatched/protected fields ignored; `0`/`false` retained; package mapping survives customization but customization cannot inject one |
| Staff UI | Pages/nav/sources/layout bindings resolve; supported widgets only; query/filter/sort, empty/error/retry and create/edit/delete journeys; contact/automation routes stay host-owned |
| Channels | Dynamic choices, descriptions/images, explicit empty states, confirmation/cancel paths; separate item/quantity screens if configured; WhatsApp compilation and completion use the same FlowRunner |
| Events/reactions | Declared events and placeholders resolve; CRM templates are presets; no implicit business writes from Catalog; follow-up conditions/cancellation paths tested |
| Presentation | Desktop/mobile/landscape overflow, keyboard/accessible labels, editable defaults, currency/media, chart aggregation, print; no console errors or unintended network calls |

Run checks from the workspace root (dependencies must already be installed):

```bash
python3 -B qefro-marketplace-apps/scripts/validate_apps.py
python3 -B qefro-marketplace-apps/tests/test_packages.py RestaurantProTests RealEstateAvailabilitySettingsTests RealEstateDomainTests
npm --prefix ai-customer-support-portal test -- EntityCrud.test.tsx fieldModel.test.ts uiCustomization.test.ts fastCreate.architecture.test.ts
npm --prefix ai-customer-support-portal run build
```

For staff UI changes, start the local host, then use the browser. The existing smoke harness loads the actual app YAML with synthetic identity and in-memory APIs, blocks external requests, and reports console/API failures:

```bash
npm --prefix ai-customer-support-portal run dev -- --host 127.0.0.1 --port 5178
```

In another terminal:

```bash
node ai-customer-support-portal/scripts/ui-polish-smoke.mjs --base-url=http://127.0.0.1:5178 --app=restaurant-pro,real-estate-pro --viewport=desktop,mobile,landscape --channel=chrome
```

Do not describe mocked browser tests as production WhatsApp or live backend validation. Report unavailable browser/channel checks explicitly. For runtime changes, run the affected Rust execution tests, including CREATE/LIST/UPDATE authority and ownership tests rather than helper-only assertions.

## Packaging and skill synchronization

Validate before packaging an app locally with `bash qefro-marketplace-apps/scripts/build_app_zip.sh <app-id>`. Inspect the archive for `manifest.yaml`, entity/workflow/UI YAML and the absence of code or secrets. Packaging is not publishing or deployment; require explicit approval for live actions.

This skill's canonical tree is `skills/qefro-marketplace-app/` in `qefro-marketplace-app-skill`. Cursor (`.cursor/skills`, `.agents/skills`) and Claude (`.claude/skills`) load that tree through symlinks. Edit the canonical files only. Keep `SKILL.md` under 500 lines, frontmatter single-line, and reference links one level deep. Do not package app source, credentials, caches, or test outputs into the skill repo.

## CI pipeline

**File:** [.github/workflows/validate.yml](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/.github/workflows/validate.yml)

Runs `python3 scripts/validate_apps.py` on push/PR to main.

## Common validation failures

1. **Missing `complete` step** -- every flow must end with `type: complete`
2. **Invalid field type** -- use only canonical types from metadata-contract.md
3. **Missing person_id** -- customer-scoped entities need `type: person` + `ref_entity: person`
4. **Authority field in entity** -- never declare `workspace_id`, `tenant_id` etc. as entity fields
5. **Secret in connection** -- connections are config metadata, not credential stores
6. **Invalid capability** -- use well-formed `resource.action` names and verify runtime support/grants; do not invent a fixed vocabulary gate
7. **Missing identity on customer tool** -- customer-facing tools need `identity.require_any`
