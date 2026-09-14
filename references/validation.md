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

**File:** [tests/test_packages.py](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/tests/test_packages.py) (~625 lines)

### Test classes

- `PackageConventionTests` -- enforces structural conventions
- `NewAppCoverageTests` -- ensures new apps meet coverage requirements

## What the validation suite checks

### Structural conventions

| Check | Description |
|-------|-------------|
| `hosting: runtime` | All apps must declare runtime hosting |
| No secrets | YAML files must not contain API keys, tokens, passwords |
| No code files | No `.py`, `.js`, `.ts`, `.go`, `.rs` source files allowed |
| Entity field types | Must be from the allowed set |
| Workflow shape | Every flow must end with a `complete` step |
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

## Compliance checklist

For every Marketplace App implementation, verify where applicable:

```
[ ] Manifest valid (hosting: runtime, all required fields present)
[ ] Entities valid (field types from canonical set, person_id for customer-scoped)
[ ] Relations valid (ref_entity points to existing entity)
[ ] Capabilities valid (from known capability set)
[ ] Flows valid (every flow ends with complete step)
[ ] Events valid (dot-notation, referenced by status_events)
[ ] Permissions valid (from known permission set)
[ ] Conversation slots valid (kind from known set)
[ ] choices_from works (output variable referenced by ask step)
[ ] Customer identity works (person_id auto-injected, not hardcoded)
[ ] Runtime execution works (entity.* tools resolve correctly)
[ ] Tenant/workspace/installation isolation works (no authority fields in params)
[ ] RBAC works (permissions declared, entity ops gated)
[ ] Agent read-only capability projection works (list + get tools generated)
[ ] No unauthorized LLM parameters (authority fields stripped)
[ ] No domain-specific runtime hardcoding (no if solution == "X")
[ ] Existing tests pass (python scripts/validate_apps.py)
[ ] Realistic end-to-end journey validated (conversational path works)
```

## CI pipeline

**File:** [.github/workflows/validate.yml](https://github.com/qefro-ai/qefro-marketplace-apps/blob/main/.github/workflows/validate.yml)

Runs `python3 scripts/validate_apps.py` on push/PR to main.

## Common validation failures

1. **Missing `complete` step** -- every flow must end with `type: complete`
2. **Invalid field type** -- use only canonical types from metadata-contract.md
3. **Missing person_id** -- customer-scoped entities need `type: person` + `ref_entity: person`
4. **Authority field in entity** -- never declare `workspace_id`, `tenant_id` etc. as entity fields
5. **Secret in connection** -- connections are config metadata, not credential stores
6. **Invalid capability** -- use only capabilities from the known set
7. **Missing identity on customer tool** -- customer-facing tools need `identity.require_any`
