# Reference Apps

## Purpose

The `qefro-marketplace-apps/apps/` repository is a **reference implementation corpus**. It demonstrates patterns, not domain logic.

**The runtime/platform contract is authoritative. Reference apps demonstrate patterns. They must NOT be blindly copied.**

## How to use reference apps

When building a new Marketplace App:

1. **Search qefro-marketplace-apps** for the closest 1-3 reference applications
2. **Identify the underlying Qefro pattern** (not the domain concepts)
3. **Verify that pattern against the current runtime** (check Rust source)
4. **Adapt it to the new business domain** (new entity names, new field names)
5. **Avoid copying domain-specific concepts** (property -> clinic is wrong)

## Reference app index

### Entity-heavy applications

**real-estate-pro** -- demonstrates:
- Multiple related entities (property, viewing, lead)
- Conversational search via choices_from
- Customer-scoped entities with person_id
- Viewing workflow with scheduling

**Pattern value:** How to structure a multi-entity app with relations and customer scope.

### Appointment applications

**clinic-pro** -- demonstrates:
- Appointment workflow with date/time collection
- choices_from for staff and service selection
- Patient entity with person_id binding
- Status events (confirmed, cancelled, completed)

**Pattern value:** How to build scheduling flows with temporal slots and dynamic choices.

### Booking applications

**appointment** -- demonstrates:
- Generic appointment scheduling
- Service + staff member entities
- Time slot collection via conversation slots

**Pattern value:** How to build a reusable scheduling pattern without domain-specific coupling.

### Conversational applications

**restaurant-pro** -- demonstrates:
- Reservation workflow with conversational steps
- Table management entity
- Multi-step booking with condition branching

**Pattern value:** How to build a multi-step conversational flow with branching logic.

### Relation-heavy applications

**education** -- demonstrates:
- Course, enrollment, student entities
- Many-to-one relations (enrollment -> course)
- Customer-scoped enrollment with person_id

**Pattern value:** How to model enrollment/registration patterns with entity relations.

### Customer-scoped applications

**field-service** -- demonstrates:
- Work order entity with technician relation
- Site entity for location tracking
- Customer-scoped work orders

**Pattern value:** How to build field service patterns with customer data isolation.

### HTTP-backed integrations

**shopify** -- demonstrates:
- Connection with OAuth metadata
- HTTP tools with webhook topic mappings
- Event emission from tool responses

**Pattern value:** How to integrate an external API with OAuth and webhooks.

**stripe** -- demonstrates:
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

Building "Restaurant Pro" after studying "Clinic Pro":

| Clinic Pro pattern | Restaurant Pro adaptation |
|---|---|
| patient (scope: customer) | diner (scope: customer) |
| appointment entity | reservation entity |
| service entity | table_type entity |
| staff_member entity | waiter entity |
| appointment.confirmed event | reservation.confirmed event |
| book_appointment flow | make_reservation flow |

The **structure** is the same. The **domain vocabulary** is different. The **runtime execution** is identical.
