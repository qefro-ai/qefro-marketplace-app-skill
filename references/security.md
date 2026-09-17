# Security

## Three-level isolation hierarchy

```
tenant_id -> workspace_id -> installation_id
```

Every data row is triple-keyed. All queries enforce triple-scoped filtering.

## Authority field stripping (C2 contract)

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs`

`FORBIDDEN_AUTHORITY_KEYS`: `tenant_id`, `organization_id`, `workspace_id`, `installation_id`, `person_id`, `customer_id`, `created_by`, `updated_by`

`strip_authority_overrides()` removes ALL these keys plus `"context"` from LLM/user-supplied parameters before runtime processing. Called on every tool invocation.

`person_id` is stripped from untrusted input so the LLM cannot invent Hub ids. Portal/staff API may re-stamp a declared `type: person` field only after Hub lookup proves the Person exists in this tenant+workspace. That is a scoped relation bind, not caller-controlled authority.

**Defense in depth:**
1. Schema stripping -- authority fields removed from LLM tool definitions (LLM never sees them)
2. Runtime stripping -- `strip_authority_overrides()` removes them even if LLM includes them
3. Executor enforcement -- called at the executor layer before any runtime processing

## Expand trust boundary (C3 contract)

**File:** `ai-customer-support/crates/api/src/flow_engine/runtime_adapter.rs`

### Relation field reference vs. expand directive

These are fundamentally different operations:

- **Relation field reference**: setting a foreign key value in create/update parameters (e.g., `{"property_id": "APT-1001"}`). The LLM can supply these (after authority stripping). This is the data layer.
- **Expand directive**: an execution control (`expand: {"property_id": "property"}`) telling storage to dereference/join the relation and return related data inline. This is the read-layer. Only flow constants may supply this.

### Source authority, not content correctness

**The trust boundary is about source authority, not content correctness.** Even a perfectly correct expand from the LLM is silently discarded because it comes from an untrusted source.

- **Trusted expand**: from `step.config.constants.expand` (admin-authored flow definition, never touched by LLM)
- **Untrusted expand**: any expand from LLM/user parameters -- completely stripped regardless of correctness

`apply_expand_trust_boundary()`:
1. Strips `expand` from untrusted parameters entirely (`doc.remove("expand")`)
2. Takes trusted expand from flow constants
3. Validates against entity metadata via `validate_expand()`
4. Each field must be a declared `type: relation`, target must match `ref_entity`
5. Fail-closed: any invalid entry rejects the entire expand map; missing metadata rejects all expand

### Dedicated extraction path

Expand is extracted directly from `step.config.constants.expand` in `steps.rs`, bypassing the general `input_map` mechanism. This is a deliberate architectural choice -- expand is treated as a first-class security concern, not just another input parameter.

## Customer identity injection (C1 contract)

For customer-scoped entities (`scope: customer` + `type: person` field):
- `person_id` injected from authenticated session identity
- Never from LLM parameters
- Fail-closed: if identity missing, operation rejected

## Positive integer coercion (C2 contract)

`limit`/`skip` parameters coerced to positive integers. Prevents injection of negative values or non-numeric strings.

## Tenant isolation enforcement

**File:** `ai-customer-support/crates/api/src/authz.rs`

- `require_same_tenant()` -- compares resource tenant_id against JWT-derived tenant_id
- `require_workspace_access()` -- returns opaque 404 (not 403) to avoid leaking workspace existence
- Default-deny: access granted only when explicitly verified
- Never trust client-supplied tenant/workspace/team/role IDs

## Organization RBAC

**File:** `ai-customer-support/crates/db/src/org_rbac.rs`

Role hierarchy: Owner > Admin > Member
- Owner/Admin bypass team checks (but remain tenant-scoped)
- Member requires team membership
- `user_has_workspace_access()` -- two-step: verify workspace belongs to user's tenant, then check role

## Installation-scoped App RBAC

**File:** `ai-customer-support/crates/api/src/app_rbac_authz.rs`

- Every permission check is `(tenant_id, workspace_id, installation_id)` scoped
- `require_entity_permission()` -- called before every entity CRUD operation
- System roles (Admin, Staff) seeded via `ensure_install_seeded()`
- Fail-closed: if RBAC catalog cannot be loaded, permissions default to empty

## Service-to-service auth

**File:** `qefro-plugin-platform/crates/qefro-plugin-common/src/auth.rs`

Three modes: Off, Log, Enforce. Production requires Enforce mode with non-empty tokens. Constant-time byte comparison (`constant_time_eq`) prevents timing attacks.

## Prompt injection defense

**File:** `ai-customer-support/crates/api/src/agent/fencing.rs`

- `FENCE_LABEL = "qefro_untrusted_data"` -- wraps all untrusted data
- `sanitize_untrusted_text()` -- strips invisible chars, control chars, fence markers, special tokens, turn indicators
- `redact_sensitive_json()` -- redacts passwords, tokens, api_keys AND authority keys before model context
- `fence_json_for_model()` -- redact -> serialize -> fence pipeline
- `MAX_FENCED_CHARS = 12,000` limits exposure
- `contains_injection_attempt()` -- detects common injection phrases

## Marketplace package safety

**File:** `qefro-marketplace-apps/tests/test_packages.py`

Structural validations:
- Entity field safety -- forbids fields named `workspace_id`, `tenant_id`, `installation_id`, `organization_id`
- HTTP tool placeholder safety -- `IDENTITY_OVERRIDE_KEYS` check prevents identity override via placeholders
- Cross-tenant authority isolation -- no package declares `workspace_id:` or `tenant_id:` as literal values
- Customer identity placeholder scoping -- only `person.email`, `person.phone`, `person.external_id` permitted
- Connection safety -- forbids `url`, `base_url`, `access_token`, `api_key`, `secret`, `password` in connections
- Embedded secret scan -- regex for `sk_live_`, `sk_test_`, `whsec_`, `xoxb-`, etc.
- LLM identity override prevention -- customer tools cannot have unqualified LLM parameters matching identity keys

## Action policy

**File:** `ai-customer-support/crates/api/src/agent/policy.rs`

```rust
pub enum ActionClass { Read, Write, Destructive, Financial, ExternalCommunication }
pub enum ActionDecision { Allow, Stage, Deny }
```

Priority: Financial > Destructive > ExternalCommunication > Read > Write

- Read -> Allow
- Non-read + staging enabled -> Stage (requires human approval)
- Non-read + allow_writes + Write class -> Allow
- Otherwise -> Deny

## Staging lifecycle

**File:** `ai-customer-support/crates/api/src/agent/staging.rs`

1. Agent proposes mutation -> `is_stageable_tool()` check
2. `build_new_pending_action()` creates DB record with sanitized args, fingerprint
3. Human approves -> `reauthorize_pending_action()` re-checks ALL authorization:
   - Tenant/workspace match
   - Expiry check
   - Status = pending
   - Argument fingerprint unchanged
   - Tool exists and method unchanged
   - Surface callable from chat
   - Role allowed
   - RBAC allowed
   - Capability allowed
   - Gates pass
   - Action class still non-read
4. 15 denial reasons in `ReauthDenial` enum

## Rules for Marketplace Apps

1. Never add `if solution == "X"` or `if entity == "Y"` in platform code
2. Domain-specific behavior belongs in Marketplace metadata
3. LLM/user parameters must not control tenant, workspace, installation, person identity, or expand
4. Trusted flow metadata/constants may define expand behavior
5. All isolation is runtime-enforced, not package-declared
6. Packages cannot override authority fields
7. Fail-closed at every layer
