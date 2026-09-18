---
name: qefro-marketplace-app
description: Design, implement, validate, and audit Qefro Marketplace Apps using Qefro's metadata-first architecture.
---

# qefro-marketplace-app

Design, implement, validate, and audit Qefro Marketplace Apps using Qefro's metadata-first architecture.

## When to use

- Building a new Marketplace App (e.g. "Build Restaurant Pro")
- Adding a feature to an existing app (e.g. "Add reservation cancellation")
- Debugging a Marketplace App (e.g. "Why does booking fail?")
- Auditing an app for Qefro compliance
- Improving an existing app (e.g. "Make the appointment flow conversational")
- Validating production readiness

## Core principle

Marketplace Apps are **metadata-first business applications**. The canonical model is:

```
Entity -> Capability -> Flow -> Event -> UI / conversational behavior
```

The Qefro Runtime executes apps from metadata alone. No custom runtime code, no per-app dispatchers, no app-specific Rust/Go/Python servers.

## Mandatory workflow

```
REQUEST
  -> REPOSITORY AUDIT (read actual code, never guess)
  -> FIND EXISTING PATTERNS (search qefro-marketplace-apps for closest 1-3 apps)
  -> CHECK QEFRO CONTRACT (verify against runtime/platform source)
  -> DESIGN METADATA (manifest, entities, flows, events, permissions, slots)
  -> CHECK RUNTIME EXECUTION (confirm runtime already executes every construct)
  -> IMPLEMENT (write YAML metadata only)
  -> TEST (run validation suite)
  -> VALIDATE REAL USER JOURNEY (end-to-end conversational path)
  -> REPORT
```

Before implementation, answer internally:

1. What existing Qefro primitive handles this?
2. Is this already implemented somewhere?
3. Which Marketplace App demonstrates the closest pattern?
4. Is the requirement metadata-only?
5. Does the runtime already execute it?
6. Does Customer Hub already handle the identity requirement?
7. Does Agent already consume the capability?
8. Is a platform change genuinely required?

## Forbidden actions

- Do NOT implement Marketplace App business logic in Qefro Rust runtime code
- Do NOT create a separate runtime for individual Marketplace Apps
- Do NOT create a second FlowRunner
- Do NOT create Marketplace-specific dispatchers
- Do NOT create an Agent-specific Marketplace App framework
- Do NOT add `if solution == "X"` or `if entity == "Y"` in platform code
- Do NOT invent field types, flow step types, or API syntax
- Do NOT allow LLM/user parameters to control tenant, workspace, installation, person identity, or expand
- Do NOT copy domain-specific concepts between reference apps (property -> clinic, etc.)

## Reference files

Load these when you need detailed technical information:

- [Architecture](references/architecture.md) -- execution hierarchy, metadata-first principle, app categories
- [Metadata contract](references/metadata-contract.md) -- manifest, entity, flow, event, capability, permission, slot schemas
- [Runtime execution](references/runtime-execution.md) -- FlowRunner, RuntimeAdapter, entity.* CRUD, choices_from, input_map
- [Identity](references/identity.md) -- Customer Hub Person, identity resolution, person_id injection, ownership
- [Security](references/security.md) -- trust boundaries, isolation, RBAC, authority stripping, expand trust
- [Agent integration](references/agent-integration.md) -- Agent layer, capability projection, read-only tools, staging
- [Validation](references/validation.md) -- test suite, validation commands, compliance checklist
- [Reference apps](references/reference-apps.md) -- how to use qefro-marketplace-apps as a pattern corpus

## Source of truth priority

1. Current Qefro runtime/platform implementation (Rust crates in `ai-customer-support/`)
2. Current Marketplace App metadata/schema contracts ([test_packages.py](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/tests/test_packages.py))
3. Existing production Marketplace Apps ([apps/](https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps))
4. Existing tests
5. Documentation
6. Your own assumptions (last resort -- report if you must use this)

If documentation conflicts with implementation, report the conflict and follow implementation.

## Repository locations

- **Reference apps:** https://github.com/qefro-ai/qefro-marketplace-apps/tree/main/apps
- Platform runtime (local): `ai-customer-support/crates/api/src/flow_engine/`, `ai-customer-support/crates/api/src/agent/`
- Validation suite: [tests/test_packages.py](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/tests/test_packages.py)
- Validation runner: [scripts/validate_apps.py](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/scripts/validate_apps.py)
- CI pipeline: [.github/workflows/validate.yml](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/.github/workflows/validate.yml)

When the local `qefro-marketplace-apps/` directory is available, read from it directly. Otherwise, browse or clone the public repository.

## Automation Templates

Marketplace Apps may declare `automation_templates` in `manifest.yaml`. Templates are **presets** for the existing CRM Automation engine.

- CRM UI discovers **only that workspace’s installed app templates**.
- [Use Template] instantiates a normal `CrmAutomation`. Templates are not executable.
- `send_webhook` templates may include `payload_mapping` only. Never URL, secret, headers, or `connection_id` in YAML. The workspace owns the outbound connection.
- Validate fail-closed against declared events, CRM action types, entity fields, and forbidden authority/secret keys.
- Upgrading the app must not mutate already-created automations.
- Do not invent TemplateEngine2, FlowRunner2, WebhookEngine, Agent/Goal templates, JS/CEL, or `if solution ==` UI.

See [Metadata contract](references/metadata-contract.md) for the schema.
