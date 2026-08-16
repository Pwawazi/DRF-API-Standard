# AGENTS.md — Django REST Framework API Standard

## Purpose

This repository uses Django REST Framework (DRF) to build production-grade REST APIs.

Whenever creating, extending, refactoring, or reviewing an API, follow this document as the default engineering standard.

The goal is to produce APIs that are:

- Resource-oriented and predictable
- Stateless at the HTTP/API layer
- Secure by default
- Consistent across endpoints
- Cacheable where appropriate
- Observable and diagnosable
- Efficient against the database
- Explicitly validated
- Idempotent where the HTTP/business semantics require it
- Versionable without unnecessary churn
- Testable
- Suitable for production deployment

Do not treat this document as a requirement to over-engineer simple endpoints. Prefer the simplest design that satisfies these constraints.

---

## 0. Rule Priority

This playbook is intentionally opinionated. Apply rules in this order:

### Non-negotiable

These must not be weakened without an explicit architectural or security decision:

- Authentication and authorization
- Tenant/object isolation
- Server-side validation
- Secret protection
- Safe handling of sensitive data
- Correct HTTP semantics
- Database integrity constraints
- Transactional integrity
- Production-safe migrations
- Security logging/audit requirements where applicable
- Tests for important security and business behavior

### Default

These are the normal project conventions:

- `/api/v1/` versioning
- Resource-oriented URLs
- DRF serializers and ViewSets/generic views
- Pagination for collections
- Filtering/search/ordering where useful
- OpenAPI documentation
- Structured errors
- Structured logging
- Query optimization
- Explicit permissions
- Automated tests

### Use when justified

These should be introduced because the workload or domain requires them, not because the checklist contains them:

- HATEOAS
- HTTP/application caching
- Cursor pagination
- Idempotency keys
- Background processing
- Outbox/event patterns
- Advanced concurrency controls
- Additional service/domain abstraction
- Distributed tracing
- Complex rate-limiting infrastructure

Do not introduce architecture merely to satisfy a theoretical REST checklist.


## 1. Core REST Principles

Design APIs around resources, not actions.

Prefer:

- `GET /api/v1/customers/`
- `GET /api/v1/customers/{id}/`
- `POST /api/v1/customers/`
- `PUT /api/v1/customers/{id}/`
- `PATCH /api/v1/customers/{id}/`
- `DELETE /api/v1/customers/{id}/`

Avoid RPC-style endpoints such as:

- `/createCustomer`
- `/getCustomer`
- `/deleteCustomer`
- `/customers/{id}/activateCustomer`

If an operation is genuinely an action rather than CRUD on a resource, use a subordinate/action endpoint only when a resource-oriented representation would be materially worse.

Example:

`POST /api/v1/invoices/{id}/send/`

should be used only when "send" is a genuine domain command.

### REST constraints to preserve

1. Client-server separation
2. Stateless requests
3. Cacheability where safe
4. Uniform interface
5. Layered architecture
6. Optional code-on-demand only when there is a compelling reason

Do not store server-side conversational/request state merely to make the API easier to implement.

---

## 1A. Practical Interpretation of REST

REST is an architectural style, not a compliance checklist.

Prioritize:

1. Correct HTTP semantics
2. Resource-oriented interfaces
3. Stateless requests
4. Clear representations
5. Cacheability where appropriate
6. Layered architecture
7. Client/server separation

Do not implement a REST feature merely because it exists in a theoretical checklist.

### Code-on-Demand

Code-on-demand is optional and normally not relevant to a JSON/DRF API. Do not introduce executable code delivery merely to claim REST compliance.

### HATEOAS

HATEOAS is optional unless the product explicitly requires hypermedia-driven clients. Use hypermedia where it provides meaningful discoverability or workflow navigation. Do not add meaningless links solely for compliance.

The practical goal is a robust HTTP API, not theoretical purity.


## 2. Resource Naming

Use plural nouns for collections.

Good:

- `/users/`
- `/properties/`
- `/properties/{id}/units/`
- `/payments/`

Avoid:

- `/getUsers/`
- `/user/`
- `/fetch-properties/`

Use lowercase kebab-case only if the project explicitly standardizes on it; otherwise use lowercase plural nouns.

Keep URLs stable and meaningful.

Avoid exposing database implementation details in URLs.

Do not use database table names merely because they exist.

---

## 3. HTTP Methods and Semantics

Use HTTP methods according to their intended semantics.

| Method | Typical use | Expected property |
|---|---|---|
| GET | Retrieve resource(s) | Safe, cacheable where appropriate |
| POST | Create or trigger a domain operation | Not inherently idempotent |
| PUT | Replace a resource | Idempotent |
| PATCH | Partially modify a resource | May be non-idempotent; design carefully |
| DELETE | Remove a resource | Idempotent in normal HTTP semantics |

Do not use `POST` for ordinary retrieval.

Do not return `200 OK` for every operation.

Use appropriate status codes, including:

- `200 OK`
- `201 Created`
- `202 Accepted` for genuinely asynchronous work
- `204 No Content`
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `405 Method Not Allowed`
- `409 Conflict`
- `422 Unprocessable Content` where the API convention calls for semantic validation errors
- `429 Too Many Requests`
- `500 Internal Server Error`

Never expose raw stack traces or internal exception details to clients in production.

---

## 4. DRF Architecture

Use DRF's resource-oriented abstractions.

Preferred default:

- `ModelSerializer` or explicit `Serializer`
- `ModelViewSet` when standard CRUD is appropriate
- `GenericViewSet` + mixins when only some CRUD operations are required
- `APIView` when the endpoint does not map cleanly to a resource or requires unusually custom request handling
- DRF routers for conventional ViewSet routing

Do not force every endpoint into `ModelViewSet`.

Keep responsibilities separated:

### Models

Responsible for:

- Data structure
- Database constraints
- Core domain invariants
- Model-level behavior

Do not put HTTP/request concerns in models.

### Serializers

Responsible for:

- Input validation
- Output representation
- Field-level and object-level validation
- Representation-specific transformations

Do not rely exclusively on frontend validation.

All client-controlled input must be validated server-side.

### Views / ViewSets

Responsible for:

- HTTP orchestration
- Authentication/authorization integration
- Selecting serializers
- Selecting querysets
- Calling domain/application services where needed
- Returning HTTP responses

Do not turn ViewSets into large business-logic dumping grounds.

### Services / domain layer

Use a service/application layer when business workflows span multiple models, transactions, external services, or side effects.

Example:

`services/payment_service.py`

rather than putting a 150-line payment workflow inside `PaymentViewSet.create()`.

### Permissions

Use DRF permission classes for authorization.

Prefer explicit, reusable permissions such as:

- `IsAuthenticated`
- object-level permissions
- role/tenant-aware custom permissions

Do not implement authorization only inside serializers or frontend code.

---

## 5. Authentication and Authorization

Authentication identifies the caller.

Authorization determines what that caller may do.

Never confuse the two.

Default production APIs should require authentication unless an endpoint is explicitly public.

Use the project's established authentication mechanism. If none exists, stop and make the authentication decision explicit rather than silently introducing an insecure mechanism.

For session-based authentication, correctly handle CSRF protection.

For token/JWT-based authentication:

- Never log tokens
- Never put credentials in URLs
- Never hard-code secrets
- Use HTTPS/TLS in production
- Keep token lifetimes and refresh behavior explicit
- Revoke/rotate credentials where the architecture supports it

Apply permissions at both:

1. Endpoint/action level
2. Object/queryset level where necessary

A user must never receive another tenant's objects merely because they guessed an object's ID.

---

## 6. Multi-Tenant / Ownership Boundaries

If the application is multi-tenant, tenant isolation is a security boundary.

Every queryset that exposes tenant-owned data must be scoped to the current tenant/user context.

Prefer:

```python
def get_queryset(self):
    return Property.objects.filter(tenant=self.request.user.tenant)
```

over retrieving all objects and checking ownership later.

Never rely on the frontend to provide the correct tenant identifier.

Never trust a client-supplied `tenant_id` for authorization.

When creating objects, derive ownership from authenticated context where possible.

---

## 7. Querysets and Database Performance

Treat database performance as part of API design.

For every endpoint, consider:

- N+1 queries
- `select_related()`
- `prefetch_related()`
- appropriate indexes
- queryset filtering
- pagination
- unnecessary serializer fields
- expensive annotations
- large unbounded result sets

Do not fetch an entire table merely to filter it in Python.

Do not iterate over large querysets in application memory when the database can perform the operation.

Use database constraints for important invariants such as:

- uniqueness
- foreign-key integrity
- check constraints
- valid state combinations

Avoid duplicate validation logic when a database constraint is the authoritative invariant.

When adding a query-heavy endpoint, inspect query count in tests or profiling where practical.

---

## 8. Pagination

Collection endpoints must be paginated unless there is a documented reason not to paginate.

Never expose an unbounded collection from a production API.

Choose a pagination strategy appropriate to the workload:

- Page-number pagination for ordinary administrative/business APIs
- Limit/offset when clients require offset semantics
- Cursor pagination for large or frequently changing datasets

Expose consistent pagination metadata.

Example shape:

```json
{
  "count": 120,
  "next": "...",
  "previous": null,
  "results": []
}
```

Do not create a different pagination format for every endpoint.

---

## 9. Filtering, Searching and Ordering

Collection endpoints should support filtering when it provides real product value.

Use standard query parameters.

Examples:

- `?status=active`
- `?property=123`
- `?search=nairobi`
- `?ordering=-created_at`

Do not invent inconsistent query parameter conventions.

Validate filter fields and ordering fields.

Never allow arbitrary model fields to become queryable or orderable without considering data exposure and query performance.

---

## 10. Representations

JSON is the default representation unless the API has a documented requirement for another representation.

Responses should be stable and predictable.

Use consistent field naming.

Prefer:

```json
{
  "id": 123,
  "name": "Example",
  "created_at": "2026-08-16T01:00:00Z"
}
```

Avoid leaking:

- database internals
- password hashes
- secrets
- internal service credentials
- private infrastructure details
- fields that the caller is not authorized to see

Do not blindly serialize every model field.

Explicitly decide what belongs in the public API representation.

---

## 11. Validation

Validate all externally supplied data.

Validation belongs at the API boundary and, where appropriate, deeper in the domain/database layers.

Validate:

- required fields
- types
- ranges
- formats
- relationships
- business rules
- state transitions
- permissions

Return machine-readable validation errors.

Do not silently coerce invalid input when doing so could hide client defects or cause data corruption.

---

## 12. Idempotency and Safe Mutations

Design mutations according to their semantics.

For operations where clients may retry requests because of:

- network failures
- timeouts
- mobile connectivity
- queue retries
- reverse proxy retries

consider an idempotency mechanism.

For high-value or externally side-effecting operations such as payments, provisioning, notifications, or irreversible workflows, prefer an explicit idempotency key.

Example:

`Idempotency-Key: <client-generated-key>`

The implementation must ensure the same logical request does not produce duplicate side effects.

Do not claim an endpoint is idempotent merely because it usually works that way.

---

## 13. Caching

Cache only data that is safe to cache.

Consider:

- HTTP cache headers
- Django cache framework
- Redis
- application-level caching
- query/result caching

Do not cache sensitive, user-specific responses in shared caches unless the cache key and headers guarantee isolation.

Cache invalidation must be deliberate.

Do not introduce caching simply because the API diagram contains "cacheable".

Measure first when performance is uncertain.

---

## 14. Security Baseline

Every production API must consider:

### TLS

Production API traffic must use HTTPS/TLS.

### CORS

Configure CORS explicitly.

Do not use:

```text
Access-Control-Allow-Origin: *
```

for authenticated browser APIs unless the security model explicitly permits it.

Do not use CORS as an authentication mechanism.

### CSRF

If using cookie/session authentication, configure CSRF correctly.

### Authentication

Require authentication for non-public resources.

### Authorization

Enforce object and tenant boundaries server-side.

### Input validation

Validate all untrusted input.

### Rate limiting

Use throttling for:

- authentication endpoints
- expensive endpoints
- public APIs
- abuse-sensitive operations

DRF throttling is an application-level control, not a complete DDoS/brute-force defense. Infrastructure-level protection may also be required.

### Secrets

Never commit:

- passwords
- API keys
- JWT secrets
- private keys
- database credentials

Use environment/configuration secret management.

### Logging

Never log:

- passwords
- access tokens
- refresh tokens
- authorization headers
- sensitive personal data unless explicitly required and protected

---

## 15. Error Handling

Use one consistent API error strategy.

Errors should be:

- predictable
- machine-readable
- safe to expose
- useful to developers
- correlated with server-side logs where appropriate

Prefer a structured shape such as:

```json
{
  "detail": "Resource not found."
}
```

For validation:

```json
{
  "email": [
    "Enter a valid email address."
  ]
}
```

If the project defines a custom error envelope, use it consistently across the API.

Do not create ad-hoc error formats per endpoint.

---

## 16. API Versioning

Version public APIs when compatibility matters.

Default convention:

`/api/v1/...`

Do not introduce a new API version for every minor change.

Backward-compatible changes should normally remain within the existing version.

Breaking changes require a deliberate versioning decision.

Versioning must cover:

- URLs/routes
- serializers/representations
- documentation
- tests
- deprecation strategy

---

## 17. HATEOAS / Hypermedia

Use hypermedia links when they materially improve discoverability or workflow navigation.

Do not add meaningless links simply to satisfy a theoretical REST checklist.

If implemented, links should be:

- correct
- stable
- permission-aware
- useful to the client

Pagination links such as `next` and `previous` are encouraged where applicable.

---

## 18. API Documentation

Every public endpoint must be documented.

Documentation should cover:

- endpoint
- HTTP method
- authentication requirements
- permissions
- request parameters
- request body
- response body
- status codes
- validation errors
- pagination
- filtering
- ordering
- example requests/responses

Prefer OpenAPI-compatible schema generation.

Keep documentation close to the implementation so it does not drift.

---

## 18A. API Contract and OpenAPI

Every production API must have a machine-readable OpenAPI schema.

Preferred DRF implementation:

- `drf-spectacular`, unless the project has an established equivalent
- Explicit request and response serializers
- Explicit authentication/security schemes
- Documented status codes
- Documented validation errors
- Pagination schemas
- Filtering/search/ordering parameters
- Representative request/response examples

The OpenAPI schema is an API contract, not merely documentation.

Where practical:

- Validate schema generation in CI
- Detect unintended breaking changes
- Keep examples executable or testable
- Keep generated documentation aligned with the implementation

Do not manually maintain a second API contract that can drift from the DRF implementation.


## 18B. Response and Error Contract

Use one response/error convention across the API.

### Success responses

Use the HTTP status code and response representation to communicate success. Do not wrap every response in an unnecessary envelope such as `{"success": true, "data": {}}` unless the project has deliberately standardized on that format.

### Errors

Use a consistent machine-readable error structure. At minimum, clients should be able to distinguish:

- authentication failure
- authorization failure
- validation failure
- not found
- conflict
- rate limiting
- unexpected server failure

Validation errors should identify relevant field(s) where applicable.

Never expose stack traces, SQL/database errors, filesystem paths, secret values, internal service topology, or unnecessary implementation details to clients.

If a custom error envelope is adopted, use it consistently across all endpoints and document it in OpenAPI.


## 19. Observability and Monitoring

Production APIs must be observable.

At minimum capture:

- request method
- route
- status code
- latency
- request/correlation ID
- authenticated principal where safe
- error details server-side
- database/query performance where practical

Do not log sensitive request bodies by default.

Use structured logging where possible.

Metrics should make it possible to answer:

- Is the API healthy?
- Which endpoints are slow?
- Which endpoints are failing?
- What is the error rate?
- Are rate limits being triggered?
- Is database latency the bottleneck?

For distributed systems, propagate a correlation/request ID across service boundaries.

---

## 20. Monitoring and Health Endpoints

Provide appropriate operational endpoints, for example:

- `/health/live`
- `/health/ready`

Liveness should answer whether the process is alive.

Readiness should answer whether the service is ready to receive traffic and may include dependency checks.

Do not expose internal diagnostics, stack traces, database credentials, or sensitive dependency details through public health endpoints.

---

## 21. Testing Standard

Every new endpoint should have tests covering, as applicable:

### Happy path

- successful retrieval
- successful creation
- successful update
- successful deletion

### Validation

- missing required fields
- invalid values
- invalid relationships
- invalid state transitions

### Authentication

- unauthenticated access
- authenticated access

### Authorization

- allowed user
- forbidden user
- object ownership
- tenant isolation

### HTTP semantics

- correct status codes
- correct HTTP methods
- unsupported methods rejected

### Query behavior

- filtering
- ordering
- pagination
- authorization-scoped querysets

### Regression

Every bug fix should include a regression test when practical.

Prefer API-level tests for endpoint behavior and focused unit tests for complex domain logic.

---

## 22. Transactions and Side Effects

Use database transactions for multi-step operations that must be atomic.

Example:

```python
from django.db import transaction

@transaction.atomic
def perform_operation(...):
    ...
```

Do not perform irreversible external side effects inside a transaction without considering rollback behavior.

For workflows involving:

- email
- payment providers
- webhooks
- queues
- external APIs

consider transaction boundaries and an outbox/event pattern where reliability requires it.

---

## 23. Background Work

Do not make API requests wait for long-running work unnecessarily.

Use asynchronous/background processing for work such as:

- email delivery
- report generation
- bulk imports
- large exports
- expensive third-party API operations
- long-running data processing

Return `202 Accepted` when a request has been accepted for asynchronous processing and the API contract supports that model.

The client should have a reliable way to determine job status where necessary.

---

## 24. URL and Routing Conventions

Use a clear namespace:

```text
/api/v1/
```

Then resource paths:

```text
/api/v1/properties/
/api/v1/properties/{id}/
/api/v1/properties/{id}/units/
```

Avoid deep nesting beyond what expresses a meaningful ownership relationship.

Prefer query parameters over excessive URL nesting for filtering.

Example:

Prefer:

`GET /api/v1/units/?property=123`

over:

`GET /api/v1/properties/123/buildings/456/floors/3/units/`

unless the nested relationship is itself an important public resource boundary.

---

## 24A. Database Migration Standards

Django migrations are production deployment artifacts.

Rules:

- Never modify an already-applied migration to change production history.
- Keep schema changes and large data migrations carefully separated where appropriate.
- Review migrations that can lock large production tables.
- Avoid destructive migrations in the same deployment as application code that still references the removed field/table.
- Prefer expand → migrate → contract for breaking schema changes.
- Add indexes and constraints deliberately.
- Consider concurrent index creation for large PostgreSQL tables where appropriate.
- Make migrations reversible where practical.
- Test important migrations against realistic data volumes.
- Do not put slow external API calls into migrations.
- Data migrations should be deterministic and safe to rerun where practical.

Before removing a column, endpoint, or model field, verify that application code and clients no longer depend on it.


## 25. Serializers and Query Optimization Must Agree

Avoid serializer patterns that silently trigger N+1 queries.

When a serializer accesses:

```python
obj.owner.name
obj.property.name
obj.units.all()
```

ensure the ViewSet/queryset preloads the necessary relationships.

Typical approach:

```python
queryset = Property.objects.select_related("owner").prefetch_related("units")
```

Do not solve database performance problems by hiding fields from serializers unless the field genuinely should not be returned.

---

## 25A. Concurrency and Race Conditions

Assume that two requests can modify the same resource simultaneously.

For stateful or high-value operations, explicitly consider:

- race conditions
- duplicate submissions
- concurrent state transitions
- lost updates
- unique constraint races
- transaction isolation
- row locking
- optimistic concurrency

Use database primitives where appropriate:

```python
select_for_update()
F(...)
transaction.atomic()
```

Do not rely on a read-then-write pattern as concurrency protection when multiple workers can execute the operation concurrently.

For APIs that require optimistic concurrency, consider version fields or conditional requests such as `ETag` / `If-Match`.

Database constraints are often the final authority for uniqueness and integrity. Catch and translate constraint violations into the API's documented conflict/error response.


## 25B. Resource State Machines

When a resource has lifecycle states, define legal transitions explicitly.

Example:

```text
DRAFT → ACTIVE → SUSPENDED → ARCHIVED
```

Do not allow clients to arbitrarily mutate a state field if the domain has meaningful transition rules.

For stateful resources:

- Define valid states
- Define legal transitions
- Validate transitions server-side
- Authorize transitions independently where necessary
- Make transitions atomic
- Test invalid transitions
- Consider idempotency for retryable transitions
- Record important transitions in audit logs

Prefer explicit domain operations for complex transitions, for example:

```text
POST /api/v1/invoices/{id}/approve/
```

rather than an unrestricted status mutation when approval has business consequences.


## 25C. Auditability

Application logs and business audit trails are different concerns.

For security-sensitive or business-critical actions, consider an immutable audit record containing:

- actor/principal
- action
- resource type
- resource identifier
- timestamp
- request/correlation ID
- relevant before/after state or change summary
- tenant/context where applicable

Audit records must not contain secrets.

Common candidates include permission changes, user administration, financial state changes, payment operations, tenant administration, destructive actions, and security configuration changes.

Do not rely on ordinary application logs as the sole source of business audit history when regulatory, operational, or forensic requirements exist.


## 26. Business Logic

Do not implement complex business rules as scattered `if` statements across serializers, views, and signals.

For meaningful workflows:

1. Validate request data
2. Authorize the operation
3. Enter the appropriate transaction boundary
4. Execute domain/application logic
5. Persist state
6. Schedule side effects safely
7. Return a stable representation

Keep domain rules reusable outside HTTP where practical.

---

## 26A. Application Services and Domain Boundaries

A service layer is a tool, not a mandatory abstraction.

Introduce an application/domain service when logic:

- spans multiple models
- requires a transaction
- coordinates external services
- contains a meaningful business workflow
- is reused outside a single HTTP action
- would otherwise make a ViewSet difficult to understand

Avoid creating services that merely wrap trivial ORM calls or move CRUD into another file without improving separation of concerns.

For substantial domains, keep business rules close to the domain and keep HTTP concerns at the API boundary.


## 27. Django Admin Is Not the Public API

Django Admin is an operational interface.

Do not expose admin serializers, models, or internal actions directly as API contracts without deliberately designing the public representation.

The public API is a product boundary.

---

## 28. Backward Compatibility

Do not casually rename or remove public fields.

Before changing an existing API:

- determine whether clients depend on it
- prefer additive changes
- deprecate before removing when appropriate
- update documentation
- update tests
- communicate breaking changes

Avoid changing response types or semantics silently.

---

## 29. Code Quality

Follow normal Python/Django engineering standards.

Prefer:

- small cohesive classes/functions
- explicit dependencies
- type hints where useful
- clear names
- reusable permission classes
- reusable serializers
- reusable filters
- service functions for complex workflows
- database constraints for data invariants

Avoid:

- massive ViewSets
- duplicated authorization logic
- duplicated validation
- hidden side effects
- raw SQL without justification
- unnecessary abstractions
- generic "utils.py" dumping grounds

Do not introduce a new dependency when the Django/DRF standard library or existing project dependency already solves the problem adequately.

---

## 30. Default Project Structure

When starting a new DRF API, prefer a structure similar to:

```text
project/
├── manage.py
├── config/
│   ├── settings/
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── apps/
│   └── <domain>/
│       ├── admin.py
│       ├── apps.py
│       ├── filters.py
│       ├── models.py
│       ├── permissions.py
│       ├── serializers.py
│       ├── services.py
│       ├── urls.py
│       ├── views.py
│       ├── tests/
│       │   ├── test_models.py
│       │   ├── test_serializers.py
│       │   ├── test_permissions.py
│       │   └── test_api.py
│       └── migrations/
└── requirements/
```

Adapt the structure when the project is small enough that this would create unnecessary ceremony.

---

## 31. Default DRF Configuration Expectations

When configuring DRF, explicitly consider:

- renderer classes
- parser classes
- authentication classes
- permission classes
- pagination
- filtering
- ordering
- throttling
- versioning
- exception handling
- schema generation

Do not blindly accept insecure DRF defaults for a production API.

In particular, explicitly decide the default permission policy rather than accidentally leaving production resources open.

---

## 31A. CI and Release Gates

A production API should have automated gates for:

- formatting/linting
- static analysis where adopted
- unit/API tests
- migration consistency
- OpenAPI schema generation
- dependency/security checks where adopted
- deployment configuration validation

At minimum, CI should prevent merging code that:

- fails the test suite
- contains unapplied migration changes
- breaks required API contracts
- introduces obvious secret leakage
- fails the project's formatting/linting rules

Keep CI fast enough for routine development; use deeper checks on protected branches/releases where appropriate.


## 32. Implementation Workflow for the Coding Agent

Whenever asked to create or modify a DRF API:

### Step 1 — Understand the resource

Identify:

- resource name
- relationships
- ownership
- tenant boundaries
- lifecycle/state
- CRUD operations
- special domain actions

### Step 2 — Design the API

Define:

- URLs
- HTTP methods
- request representations
- response representations
- status codes
- filtering
- ordering
- pagination
- authentication
- permissions
- error behavior

Do this before writing substantial code.

### Step 3 — Design the data layer

Identify:

- models
- constraints
- indexes
- relationships
- transaction boundaries

### Step 4 — Implement

Prefer:

- serializers for validation/representation
- ViewSets/generic views for HTTP behavior
- permissions for authorization
- querysets for data isolation
- services for complex workflows

### Step 5 — Secure

Verify:

- authentication
- authorization
- object ownership
- tenant isolation
- CSRF where applicable
- CORS
- throttling
- secret handling
- sensitive logging

### Step 6 — Optimize

Check:

- N+1 queries
- indexes
- pagination
- unnecessary fields
- expensive database operations

### Step 7 — Test

Add endpoint, authorization, validation, and regression tests.

### Step 8 — Document

Update API documentation/schema and examples.

### Step 9 — Review

Before declaring the task complete, verify that the implementation follows this AGENTS.md.

---

## 33. Definition of Done

A DRF API change is not complete merely because the endpoint returns data.

The change is complete when:

- [ ] Resource naming is consistent
- [ ] HTTP methods have correct semantics
- [ ] Authentication is explicit
- [ ] Authorization is enforced
- [ ] Tenant/object isolation is enforced where applicable
- [ ] Input validation is implemented
- [ ] Appropriate status codes are returned
- [ ] Error responses are consistent
- [ ] Collection endpoints are paginated
- [ ] Filtering/search/ordering are deliberate
- [ ] Querysets avoid N+1 queries
- [ ] Appropriate indexes/constraints exist
- [ ] Idempotency has been considered
- [ ] Caching has been considered
- [ ] Rate limiting has been considered
- [ ] Sensitive data is not leaked
- [ ] Logging is safe and useful
- [ ] Tests cover success and failure paths
- [ ] OpenAPI schema is updated and valid
- [ ] API documentation/examples are updated
- [ ] API error contract remains consistent
- [ ] Database migrations are production-safe
- [ ] Concurrency/race conditions have been considered
- [ ] State transitions are explicit where applicable
- [ ] Auditability has been considered for business/security-critical actions
- [ ] Versioning and backward compatibility have been considered
- [ ] CI/release checks pass
- [ ] Production deployment implications have been considered

---

## 34. Agent Behavior

When requirements are ambiguous:

1. Prefer established project conventions.
2. Prefer REST/HTTP semantics over arbitrary conventions.
3. Prefer secure defaults.
4. Prefer backward-compatible changes.
5. Prefer simple designs over speculative abstractions.
6. Ask a clarification question when a decision materially affects API compatibility, security, data integrity, or architecture.
7. Do not silently make breaking API changes.
8. Do not introduce dependencies or infrastructure without justification.
9. Do not mark an API implementation complete without tests for the important security and behavior paths.
10. Treat the priority model in Section 0 as the decision hierarchy.
11. Prefer existing project conventions when they do not violate non-negotiable rules.
12. Before adding caching, asynchronous processing, HATEOAS, a service abstraction, or other advanced architecture, establish the concrete requirement that justifies it.
13. When changing a database schema, consider migration safety, deployment ordering, locking, rollback, and backward compatibility.
14. When modifying stateful or high-value resources, explicitly evaluate concurrency, idempotency, and state transitions.

When an existing project conflicts with this document, preserve existing public API contracts unless the task explicitly requests a breaking change. Improve internal implementation incrementally.

---

## 35. Reference Model

The API architecture should conceptually follow this chain:

REST
→ Principles
→ Resource-oriented interface
→ HTTP methods
→ Representations
→ Stateless/client-server/layered architecture
→ Pagination/filtering/ordering
→ Resource naming
→ Versioning
→ Security
→ Authentication/authorization
→ Validation
→ Idempotency
→ CORS/CSRF/TLS
→ Rate limiting
→ Logging/monitoring
→ Caching
→ Documentation
→ Testing

This is the baseline. Individual projects may extend it, but should not casually weaken it.


## 35A. Final Engineering Review

Before declaring a DRF API task complete, perform a final review from these perspectives:

### API contract

- Are URLs resource-oriented?
- Are HTTP methods semantically correct?
- Are status codes correct?
- Are request/response schemas documented?
- Is OpenAPI accurate?

### Security

- Can an unauthenticated user access anything they should not?
- Can an authenticated user access another user's or tenant's data?
- Can a client forge ownership or tenant identifiers?
- Are secrets and sensitive fields protected?
- Are CORS/CSRF/TLS settings appropriate?
- Are abuse-sensitive endpoints throttled?

### Data integrity

- Are constraints enforced at the database where appropriate?
- Are transactions correct?
- Could concurrent requests corrupt state?
- Are state transitions valid?
- Are migrations safe for production?

### Performance

- Is the collection paginated?
- Are querysets properly scoped?
- Are N+1 queries avoided?
- Are indexes appropriate?
- Is unnecessary data being serialized?

### Reliability

- Can the operation safely be retried?
- Is idempotency required?
- Are external side effects coordinated with transactions?
- Should work be asynchronous?

### Observability

- Can failures be diagnosed from logs/metrics?
- Is correlation/request tracing available where needed?
- Are sensitive values excluded from logs?
- Are important business/security actions auditable?

### Compatibility

- Is this backward-compatible?
- If not, is a new API version or migration strategy required?
- Are existing clients protected during rollout?

### Testing

- Does the test suite cover success, failure, validation, authorization, isolation, and concurrency-sensitive behavior where applicable?

If any answer is unclear, do not silently assume the safest-looking implementation. Resolve the ambiguity or choose the least risky design consistent with existing project conventions.
