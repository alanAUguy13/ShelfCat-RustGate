# ShelfCat RustGate Engineering Charter

You are implementing ShelfCat RustGate as a real, compilable, testable Rust application.

Do not produce pseudocode, placeholders, fake success output, omitted implementations, empty handlers, TODO-only functions, `unimplemented!()`, `todo!()`, panics for normal errors, wildcard authorization, fabricated test results, or documentation claiming a capability that is not implemented and tested.

## Operating rules

1. Inspect the existing repository before modifying anything.
2. Preserve useful existing work.
3. Work in small, reviewable milestones.
4. After every milestone, run the required checks.
5. Fix failures before continuing.
6. Never state that a command passed unless it was executed successfully.
7. Never invent command output.
8. If a required tool is unavailable, report the exact missing tool and provide the exact installation command.
9. Do not weaken tests, lint configuration, authorization, validation, or security controls merely to make CI pass.
10. Do not add `|| true` to required validation steps.
11. Do not suppress warnings globally.
12. Do not use floating GitHub Action references such as `@master`.
13. Use stable, released dependencies compatible with the repository's Rust toolchain.
14. Commit `Cargo.lock`.
15. Generate the OpenAPI document from implemented Rust routes and schemas.
16. Check the generated OpenAPI document into `openapi/rustgate-v1.json`.
17. CI must regenerate the document and fail if the checked-in document differs.
18. All persistent tenant-owned rows must include `tenant_id`.
19. Every tenant-owned query must include tenant filtering.
20. Authorization decisions must never depend only on `X-Tenant-Id`.
21. The authenticated tenant claim is authoritative.
22. If `X-Tenant-Id` is present, it must equal the authenticated tenant claim unless the caller has an explicitly implemented platform administration scope.
23. Defaults must deny access.
24. Do not implement wildcard scopes such as `admin:*`.
25. Use explicit scopes.
26. Never log bearer tokens, signing keys, raw secrets, or complete sensitive evidence packages.
27. Return generic external errors and retain detailed errors in structured server logs.
28. UTC timestamps must be RFC 3339.
29. Stored hashes use lowercase SHA-256 encoded as `sha256:<64 lowercase hexadecimal characters>`.
30. Cryptographic signatures use Ed25519 and include a key identifier.
31. Hashes and signatures must be calculated over a documented canonical byte representation.
32. Never sign arbitrary in-memory JSON serialization whose field ordering is not specified.
33. Database migrations are immutable after merging.
34. Destructive database operations require an intentional, separately reviewed migration.
35. The application must shut down gracefully.
36. Health endpoints must not require authentication, but must disclose no secrets.
37. Readiness must fail when required dependencies are unavailable.
38. Liveness must only report process health.
39. API documentation must identify implemented behavior only.
40. Every changed endpoint requires integration tests.

---

# Product objective

Implement an auditable, deterministic, multi-tenant policy-decision service.

The first release must support:

1. Tenant creation and lifecycle administration.
2. Immutable, versioned policy creation.
3. Policy validation.
4. Policy signing.
5. Policy activation.
6. Deterministic decision evaluation.
7. Signed decision receipts.
8. Append-only, hash-chained ledger records.
9. Decision evidence retrieval.
10. Evidence verification.
11. Evidence export.
12. Single-decision replay.
13. Replay result retrieval.
14. Replay verification.
15. OpenAPI 3.1 generation.
16. PostgreSQL migrations.
17. OAuth 2.0 access-token validation using JWT and JWKS.
18. Explicit scope authorization.
19. Tenant isolation.
20. CI validation.
21. Container builds.
22. SBOM production.
23. Build provenance attestation.

The first release is not complete until the acceptance tests in this document pass.

---

# Technology baseline

Use:

- Rust stable, pinned in `rust-toolchain.toml`.
- Tokio.
- Axum.
- Tower and Tower HTTP.
- Serde and serde_json.
- UUID.
- Chrono with UTC only.
- SQLx with PostgreSQL, migrations, and rustls support.
- Utoipa with OpenAPI 3.1 support.
- Tracing and tracing-subscriber.
- SHA-256.
- Ed25519 from a well-maintained Rust cryptography crate.
- JWT validation with asymmetric algorithms and JWKS retrieval.
- Reqwest with rustls where outbound HTTP is required.
- Thiserror for library errors.
- Anyhow only at binary/application boundaries.
- PostgreSQL 16 or newer for local and CI testing.
- Docker Compose for local dependencies.
- GitHub Actions for CI and release automation.

Do not introduce Redis, Kafka, Kubernetes, Terraform, service meshes, or multiple separately deployed Rust services in the first implementation. First prove the domain model and transaction boundaries in one deployable API with modular crates. Split services later only when operational evidence justifies it.

Use a modular monolith for release 1:

- one API binary;
- one PostgreSQL database;
- internal crates for domain separation;
- strict module and database boundaries;
- reliable transactional behavior.

This avoids distributed partial failures while preserving future extraction boundaries.

---

# Repository layout

Create or converge on this structure:

```text
.
├── .cargo/
│   └── config.toml
├── .github/
│   ├── copilot-instructions.md
│   ├── dependabot.yml
│   └── workflows/
│       ├── ci.yml
│       ├── container.yml
│       └── release.yml
├── crates/
│   ├── common-types/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── rustgate-auth/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── rustgate-core/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── rustgate-ledger/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── rustgate-policy/
│   │   ├── Cargo.toml
│   │   └── src/
│   └── rustgate-storage/
│       ├── Cargo.toml
│       └── src/
├── services/
│   └── rustgate-api/
│       ├── Cargo.toml
│       ├── Dockerfile
│       └── src/
├── migrations/
├── openapi/
│   └── rustgate-v1.json
├── scripts/
│   ├── check.sh
│   ├── export-openapi.sh
│   └── smoke-test.sh
├── tests/
│   ├── integration/
│   └── fixtures/
├── Cargo.toml
├── Cargo.lock
├── deny.toml
├── docker-compose.yml
├── rust-toolchain.toml
├── .env.example
├── .gitattributes
├── .gitignore
├── LICENSE
├── README.md
└── SECURITY.md
```

Set `.gitattributes` to normalize Rust, SQL, JSON, YAML, and shell scripts to LF line endings.

---

# Workspace boundaries

## `common-types`

Contains serializable API and domain data types only.

It must not contain:

- database pools;
- HTTP clients;
- Axum extractors;
- tracing subscribers;
- environment loading;
- application state.

Required public types:

- `ActionRequest`
- `IdentityContext`
- `ResourceReference`
- `Decision`
- `DecisionReceipt`
- `DecisionEvidence`
- `PolicyDefinition`
- `PolicyRule`
- `PolicyCondition`
- `PolicyStatus`
- `Tenant`
- `TenantStatus`
- `AuditLedgerEntry`
- `ReplayRequest`
- `ReplayJob`
- `ReplayResult`
- `ReplayStatus`
- `EvidenceVerificationResult`
- `ReplayVerificationResult`
- `ArtifactSignature`
- `ApiError`
- `ErrorCode`
- `Page<T>`
- `PageCursor`

All externally visible types must derive the appropriate Serde and Utoipa schema traits.

Use `deny_unknown_fields` for signed, hashed, or persistence-critical contract structures where forward compatibility will not be harmed. If unknown fields are permitted, document why.

## `rustgate-core`

Contains deterministic domain logic:

- canonical serialization;
- SHA-256 hashing;
- Ed25519 signing and verification;
- decision receipt construction;
- decision hash construction;
- evidence hash construction;
- replay comparison;
- tenant-boundary validation;
- deterministic identifiers where required;
- clock and signer interfaces;
- test implementations of the clock and signer.

No database, HTTP, or environment dependencies.

## `rustgate-policy`

Contains:

- policy validation;
- policy state transitions;
- condition evaluation;
- deny-overrides evaluation;
- deny-by-default behavior;
- policy canonicalization;
- policy hash calculation.

No database or HTTP dependencies.

## `rustgate-ledger`

Contains:

- ledger-entry construction;
- chain-hash calculation;
- chain verification;
- event-type definitions;
- interfaces required by storage.

No HTTP dependencies.

## `rustgate-auth`

Contains:

- bearer-token extraction;
- JWT validation;
- issuer validation;
- audience validation;
- expiration and not-before validation;
- explicit algorithm allow-list;
- JWKS retrieval and cache;
- authenticated principal;
- scope parsing;
- tenant claim parsing;
- authorization middleware;
- explicit platform-administrator controls.

Never decode a JWT without verifying its signature.

Never accept `alg=none`.

Do not accept HMAC tokens when asymmetric JWT validation is configured.

Required claims:

- `iss`
- `sub`
- `aud`
- `exp`
- `iat`
- `scope`
- `tenant_id` for tenant-bound callers

`scope` is a space-delimited string unless the configured identity provider requires another documented representation.

## `rustgate-storage`

Contains SQLx repositories and transactions.

All tenant-owned repository methods must require a `TenantId` parameter.

Do not expose generic methods that permit callers to forget tenant filtering.

## `rustgate-api`

Contains:

- routers;
- request extractors;
- handlers;
- middleware;
- dependency wiring;
- configuration;
- health endpoints;
- OpenAPI document generation;
- graceful shutdown;
- HTTP error mapping.

Handlers must be thin. Domain logic belongs in library crates.

---

# Canonicalization contract

Cryptographic verification is impossible without stable bytes.

Create a versioned canonicalization contract named:

```text
shelfcat-jcs-v1
```

For release 1, use RFC 8785 JSON Canonicalization Scheme semantics through a maintained implementation, or implement the required subset with comprehensive conformance tests.

Every signed object must include:

```json
{
  "canonicalization": "shelfcat-jcs-v1",
  "hash_algorithm": "sha-256",
  "signature_algorithm": "ed25519",
  "key_id": "urn:shelfcat:key:<identifier>"
}
```

The signature value must be base64url without padding.

The signature input must be explicitly documented for each object.

Do not include a signature field inside the bytes being signed.

For a decision receipt, sign the canonical serialization of all receipt fields except `signature`.

For evidence, sign the canonical serialization of all evidence fields except `signature`.

For ledger entries, calculate `entry_hash` from the canonical serialization of the entry fields except `entry_hash` and `signature`.

The genesis ledger entry uses:

```text
previous_hash = sha256:0000000000000000000000000000000000000000000000000000000000000000
```

Create canonicalization golden tests with checked-in fixtures.

The same fixture must produce the same hash in debug and release builds.

---

# Policy model

Implement a deliberately bounded policy language.

Do not execute arbitrary scripts, CEL, Rego, JavaScript, SQL, shell, or user-provided regular expressions in release 1.

A policy contains ordered rules, but evaluation uses deny-overrides:

1. Select the active policy version explicitly associated with the request.
2. Validate tenant ownership.
3. Validate policy status is `ACTIVE`.
4. Evaluate all applicable rules.
5. If any applicable rule evaluates to `DENY`, return `DENY`.
6. Otherwise, if at least one applicable rule evaluates to `ALLOW`, return `ALLOW`.
7. Otherwise return `DENY` with `NO_APPLICABLE_ALLOW_RULE`.

Supported condition operators:

- `STRING_EQUALS`
- `STRING_NOT_EQUALS`
- `STRING_IN`
- `BOOLEAN_EQUALS`
- `INTEGER_EQUALS`
- `INTEGER_GREATER_THAN`
- `INTEGER_GREATER_THAN_OR_EQUAL`
- `INTEGER_LESS_THAN`
- `INTEGER_LESS_THAN_OR_EQUAL`
- `EXISTS`
- `NOT_EXISTS`

Supported condition sources:

- `identity`
- `resource`
- `context`
- `request`

Use dotted paths with a strict parser.

Reject:

- empty paths;
- path traversal;
- array wildcards;
- recursive descent;
- duplicate rule IDs;
- unsupported operators;
- type-mismatched comparisons;
- unknown effects;
- empty rule sets;
- unsigned activation;
- activation of an invalid policy.

Policy statuses:

- `DRAFT`
- `VALIDATED`
- `SIGNED`
- `ACTIVE`
- `SUPERSEDED`
- `RETIRED`

Allowed transitions:

```text
DRAFT -> VALIDATED
VALIDATED -> DRAFT
VALIDATED -> SIGNED
SIGNED -> ACTIVE
ACTIVE -> SUPERSEDED
ACTIVE -> RETIRED
SIGNED -> RETIRED
```

No other transitions are permitted.

Policy versions are immutable. Updating a policy creates a new version.

Only one active version of a policy family may exist per tenant.

Activation and superseding of the previous active version must happen in one database transaction.

---

# Tenant model

Tenant statuses:

- `ACTIVE`
- `SUSPENDED`
- `DECOMMISSIONED`

Rules:

- Only platform administrators can create tenants.
- Tenant administrators can read their own tenant.
- Only platform administrators can list all tenants.
- Suspended tenants cannot create policies, activate policies, request decisions, run replays, or export evidence.
- Suspended tenants may read existing decisions and evidence if explicitly allowed by scope.
- Decommissioned tenants cannot use normal tenant endpoints.
- Decommissioning requires an explicit reason.
- Decommissioning is irreversible through the public API.
- Physical deletion is not exposed through the public API.

Do not accept arbitrary metadata without limits.

Set documented limits for:

- tenant name;
- metadata keys;
- metadata values;
- total metadata size.

---

# Required OAuth scopes

Implement only explicit scopes:

```text
decision:create
decision:read
policy:create
policy:read
policy:validate
policy:sign
policy:activate
policy:retire
tenant:create
tenant:read
tenant:list
tenant:suspend
tenant:activate
tenant:decommission
ledger:read
evidence:read
evidence:verify
evidence:export
replay:create
replay:read
replay:verify
```

No wildcard scope.

Platform administration requires both:

1. the required explicit scope; and
2. `principal_type=platform_admin` in verified claims.

Tenant administrators cannot elevate themselves by sending headers.

---

# Required API

All routes are under `/v1`, except health and documentation endpoints.

## Health and documentation

```text
GET /health/live
GET /health/ready
GET /openapi.json
GET /docs
```

## Decisions

```text
POST /v1/decisions
GET /v1/decisions/{decision_id}
GET /v1/decisions/{decision_id}/evidence
POST /v1/decisions/{decision_id}/replay
```

## Evidence

```text
GET /v1/evidence/{decision_id}
POST /v1/evidence/{decision_id}/verify
GET /v1/evidence/{decision_id}/export
```

## Replay

```text
POST /v1/replays
GET /v1/replays/{replay_id}
GET /v1/replays/{replay_id}/result
POST /v1/replays/{replay_id}/verify
```

Use plural `/replays`. Do not create both `/replay` and `/replays`.

## Policies

```text
POST /v1/policies
GET /v1/policies
GET /v1/policies/{policy_id}
POST /v1/policies/{policy_id}/versions
GET /v1/policies/{policy_id}/versions
GET /v1/policies/{policy_id}/versions/{version}
POST /v1/policies/{policy_id}/versions/{version}/validate
POST /v1/policies/{policy_id}/versions/{version}/sign
POST /v1/policies/{policy_id}/versions/{version}/activate
POST /v1/policies/{policy_id}/versions/{version}/retire
```

## Tenants

```text
POST /v1/tenants
GET /v1/tenants
GET /v1/tenants/{tenant_id}
POST /v1/tenants/{tenant_id}/suspend
POST /v1/tenants/{tenant_id}/activate
POST /v1/tenants/{tenant_id}/decommission
```

## Ledger

```text
GET /v1/ledger/entries/{entry_id}
GET /v1/ledger/verify
```

---

# HTTP behavior

## Request headers

All authenticated tenant operations require:

```text
Authorization: Bearer <access-token>
X-Request-Id: <UUID>
X-Tenant-Id: <UUID>
```

Mutating operations also require:

```text
Idempotency-Key: <nonempty opaque value, maximum 128 characters>
```

The API may generate `X-Request-Id` when the client omits it, but the generated value must be returned.

Do not use `Idempotency-Key` on GET requests.

## Response headers

Return:

```text
X-Request-Id
X-Content-Type-Options: nosniff
Cache-Control: no-store
```

Cryptographically protected resource responses also return:

```text
Digest: sha-256=<base64 digest>
X-ShelfCat-Key-Id: <key identifier>
X-ShelfCat-Signature: <base64url signature>
```

Application-level signatures must not be described as HTTP Message Signatures unless that standard is actually implemented.

## Status codes

Use:

- `200` for successful reads, verification, and synchronous actions.
- `201` for created resources.
- `202` for accepted asynchronous replay jobs.
- `204` only when there is intentionally no response body.
- `400` for malformed requests.
- `401` for missing or invalid authentication.
- `403` for authenticated callers lacking permission.
- `404` when a resource does not exist or must be concealed across tenant boundaries.
- `409` for state transition, duplicate, active-policy, or idempotency conflicts.
- `412` for failed conditional requests.
- `415` for unsupported media types.
- `422` for syntactically valid but semantically invalid contracts.
- `429` for rate-limit enforcement when implemented.
- `500` for unexpected errors.
- `503` for unavailable dependencies.

Every documented error response must use the common error schema.

## Error structure

```json
{
  "type": "https://errors.shelfcat.example/v1/<error-code>",
  "title": "Human-readable error class",
  "status": 409,
  "code": "POLICY_STATE_CONFLICT",
  "detail": "Safe client-facing explanation",
  "request_id": "UUID",
  "timestamp": "RFC3339 UTC",
  "violations": [
    {
      "field": "rules[0].conditions[0].operator",
      "code": "UNSUPPORTED_OPERATOR",
      "message": "Safe validation message"
    }
  ]
}
```

Do not return stack traces, SQL errors, JWT details, key material, or internal filenames.

---

# Idempotency

Implement durable idempotency for every mutating endpoint.

Database record:

- tenant ID;
- authenticated subject;
- endpoint operation ID;
- idempotency key;
- canonical request hash;
- response status;
- response headers required for replay;
- response body;
- created timestamp;
- expiration timestamp.

Rules:

1. Same tenant, subject, operation, key, and request hash returns the original response.
2. Same key with a different request hash returns `409 IDEMPOTENCY_KEY_REUSE`.
3. In-progress duplicate requests return `409 IDEMPOTENCY_REQUEST_IN_PROGRESS` or wait using a documented bounded strategy.
4. Keep records for at least 24 hours.
5. The domain write and idempotency result must commit atomically when possible.
6. Never trust a client-provided request ID as an idempotency mechanism.

---

# Decision transaction

`POST /v1/decisions` must execute as one PostgreSQL transaction:

1. Validate authentication and scope.
2. Validate tenant claim and header.
3. Confirm tenant is active.
4. Claim the idempotency key.
5. Validate the request.
6. Load the exact active policy version.
7. Evaluate the request deterministically.
8. Insert the action request.
9. Insert the decision.
10. Create and insert the decision receipt.
11. Lock the tenant ledger head.
12. Create the next ledger entry.
13. Insert the ledger entry.
14. Create and insert the evidence package.
15. Store the idempotent response.
16. Commit.
17. Return only after commit.

If any required write fails, roll back the transaction and do not return a successful decision.

Use a per-tenant ledger head row or equivalent locking strategy to serialize ledger appends safely.

Add a concurrency test proving that simultaneous decisions do not create ledger forks.

---

# Replay behavior

Release 1 supports single-decision replay.

A replay stores immutable references to:

- original request;
- exact policy version;
- original decision receipt;
- policy hash;
- context hash;
- engine version;
- canonicalization version.

Replay must:

1. Load the original immutable inputs.
2. Verify their stored hashes.
3. Execute the policy engine.
4. Recalculate the decision hash.
5. Compare expected and actual hashes using constant-time comparison where appropriate.
6. Store the result.
7. Sign the replay result.
8. Write a ledger event.

Replay statuses:

- `QUEUED`
- `RUNNING`
- `PASSED`
- `FAILED`
- `ERROR`

`FAILED` means deterministic output mismatch.

`ERROR` means replay could not complete.

Do not treat execution errors as deterministic mismatches.

For release 1, replay processing may use an internal PostgreSQL-backed job loop in the API process. Use `FOR UPDATE SKIP LOCKED` and bounded retries. Do not claim high-availability job processing until tested with multiple workers.

---

# Evidence export

Evidence export must be deterministic and verifiable.

Return `application/zip`.

Archive path rules:

- UTF-8 filenames;
- no absolute paths;
- no `..`;
- no symlinks;
- fixed entry order;
- fixed ZIP timestamps;
- deterministic compression configuration.

Archive contents:

```text
manifest.json
request.json
policy.json
receipt.json
ledger-entry.json
evidence.json
replay-result.json
signatures.json
```

`replay-result.json` is included only when a replay exists.

`manifest.json` lists each file, media type, byte length, and SHA-256 hash.

The manifest is signed.

Export itself writes an auditable ledger event, but the export ledger event must not cause the evidence package being exported to change recursively.

Add deterministic export golden tests.

---

# Database schema

Create SQLx migrations for these tables:

```text
tenants
policies
policy_versions
action_requests
decisions
decision_receipts
ledger_heads
ledger_entries
decision_evidence
replay_jobs
replay_results
artifact_signatures
idempotency_records
```

Required database principles:

- UUID primary keys.
- `timestamptz`.
- explicit foreign keys.
- explicit unique constraints.
- explicit check constraints.
- hashes stored in constrained text fields.
- signatures stored separately from signed bytes.
- immutable policy versions.
- immutable decision inputs.
- immutable receipts.
- append-only ledger entries.
- no cascading deletes from tenants into evidence or ledger data.
- indexes start with `tenant_id` for tenant-scoped access patterns.
- partial unique index enforcing one active policy version per tenant and policy family.
- unique request ID per tenant.
- unique decision ID per tenant.
- unique replay ID per tenant.
- unique ledger sequence per tenant.
- unique ledger hash per tenant.
- no nullable tenant identifiers on tenant-owned data.

Use SQL triggers only when they protect invariants that cannot be enforced reliably in application code and ordinary constraints. Document every trigger.

Add migration tests that:

1. create a blank database;
2. apply every migration;
3. exercise key constraints;
4. verify expected indexes;
5. confirm migration checksums;
6. start the API against the migrated database.

Never edit an applied migration. Add new migrations.

---

# OpenAPI requirements

Generate OpenAPI 3.1 from implemented routes and Rust contract types.

The generated specification must include:

- title;
- API version;
- license;
- server template;
- tags;
- operation IDs;
- OAuth 2.0 authorization code flow where applicable;
- OAuth 2.0 client credentials flow for machine callers;
- exact scopes;
- all request parameters;
- request bodies;
- response bodies;
- response headers;
- shared errors;
- pagination;
- examples that validate against their schemas;
- security requirements per operation;
- deprecation flags;
- schema constraints;
- formats;
- maximum lengths;
- minimum values;
- enums;
- required properties.

Do not document endpoints that do not exist.

Do not document response codes handlers cannot return.

Do not use `additionalProperties: true` for signed domain contracts.

Dynamic attribute maps must use JSON scalar values only and enforce:

- maximum key count;
- maximum key length;
- maximum string-value length;
- maximum nesting depth;
- maximum serialized size.

OpenAPI generation command:

```text
cargo run -p rustgate-api --bin export-openapi
```

The command must write:

```text
openapi/rustgate-v1.json
```

CI must run the generator and then:

```text
git diff --exit-code -- openapi/rustgate-v1.json
```

Lint the generated file with Redocly.

Validate it with an independent OpenAPI parser.

Run contract tests against a live API instance.

---

# Configuration

Use environment variables, with a typed configuration structure.

Required variables:

```text
RUSTGATE_BIND_ADDR
RUSTGATE_DATABASE_URL
RUSTGATE_PUBLIC_BASE_URL
RUSTGATE_JWT_ISSUER
RUSTGATE_JWT_AUDIENCE
RUSTGATE_JWKS_URL
RUSTGATE_SIGNING_KEY_PATH
RUSTGATE_SIGNING_KEY_ID
RUSTGATE_LOG_FILTER
RUSTGATE_MAX_BODY_BYTES
RUSTGATE_DB_MAX_CONNECTIONS
RUSTGATE_REPLAY_WORKERS
RUSTGATE_REPLAY_MAX_ATTEMPTS
```

Rules:

- fail startup on missing required configuration;
- fail startup on malformed URLs;
- fail startup on insecure remote issuer/JWKS URLs outside explicit development mode;
- redact secrets in debug output;
- do not commit real secrets;
- provide `.env.example`;
- production signing keys must be mounted from a secret manager or protected file;
- local development may use a checked-out test-only key clearly marked as non-production;
- reject world-readable private-key files where platform APIs permit checking.

---

# Observability

Emit structured JSON logs in production.

Every request log includes:

- request ID;
- method;
- route pattern;
- status;
- duration;
- authenticated subject identifier;
- tenant ID when present;
- operation ID.

Never log:

- bearer tokens;
- private keys;
- complete action context;
- policy signatures;
- response signatures;
- raw evidence archives.

Expose Prometheus-compatible metrics only if implemented and tested. If metrics are not implemented, do not document them.

Required initial metrics:

- request count by route, method, and status class;
- request duration;
- decision count by effect and reason code;
- policy evaluation duration;
- replay count by status;
- replay duration;
- database-pool utilization;
- ledger append failures;
- authentication failures;
- authorization failures.

Do not put tenant IDs, user IDs, request IDs, policy IDs, or decision IDs in metric labels.

---

# Local development

`docker-compose.yml` must run:

- PostgreSQL;
- a local OpenID Connect provider suitable for development and integration testing;
- RustGate API.

Provide health checks.

Provide a bootstrap script or realm configuration for:

- a platform-admin client;
- a tenant-service client;
- documented development scopes;
- a test tenant.

Do not use development credentials in staging or production documentation.

Required developer flow:

```bash
cp .env.example .env
docker compose up --build -d
./scripts/smoke-test.sh
```

The smoke test must:

1. wait for readiness;
2. obtain a real OAuth access token from the local identity provider;
3. create a tenant;
4. create a policy;
5. validate the policy;
6. sign the policy;
7. activate the policy;
8. request one allowed decision;
9. request one denied decision;
10. retrieve evidence;
11. verify evidence;
12. start replay;
13. poll the replay result with timeout;
14. verify replay;
15. export evidence;
16. verify the archive manifest;
17. exit nonzero on any failure.

Do not parse JSON with grep. Use `jq`.

---

# Testing requirements

## Unit tests

Test:

- policy validation;
- each condition operator;
- type mismatch behavior;
- missing path behavior;
- deny overrides;
- deny by default;
- canonical serialization;
- known SHA-256 fixtures;
- signature verification;
- signature tampering;
- receipt hash construction;
- evidence hash construction;
- ledger chain construction;
- ledger chain tampering;
- replay pass;
- replay mismatch;
- every policy transition;
- scope checks;
- claim validation.

## Integration tests

Use real PostgreSQL.

Test:

- tenant creation authorization;
- cross-tenant concealment;
- cross-tenant writes;
- suspended-tenant restrictions;
- policy lifecycle;
- one-active-version invariant;
- idempotent decision retries;
- idempotency-key conflicting payload;
- decision atomicity;
- ledger atomicity;
- concurrent ledger appends;
- evidence retrieval;
- evidence tampering detection;
- replay lifecycle;
- replay mismatch recording;
- pagination stability;
- malformed JWT;
- expired JWT;
- wrong issuer;
- wrong audience;
- unknown signing key;
- insufficient scope;
- tenant header/claim mismatch;
- maximum body size;
- graceful shutdown.

## Property tests

Property-test:

- policy evaluation determinism;
- canonicalization determinism;
- ledger chain verification;
- arbitrary JSON attribute maps within defined constraints.

## Security checks

CI must include:

- `cargo fmt --all --check`
- `cargo clippy --workspace --all-targets --all-features -- -D warnings`
- `cargo test --workspace --all-features`
- dependency advisory audit;
- dependency license and source policy;
- secret scanning;
- filesystem and dependency vulnerability scanning;
- container vulnerability scanning;
- OpenAPI lint;
- migration validation;
- generated-file drift detection.

No required security check may use `continue-on-error: true`.

---

# CI workflow

Create `.github/workflows/ci.yml`.

Requirements:

- run on pull requests and pushes to `main`;
- set top-level permissions to `contents: read`;
- use concurrency cancellation for obsolete runs;
- pin actions to immutable commit SHAs;
- add comments beside pins identifying intended release tags;
- use Dependabot to update action pins;
- cache Cargo safely;
- run contract validation;
- run formatting and Clippy;
- run unit and integration tests;
- start PostgreSQL as a service;
- run migrations;
- regenerate OpenAPI and fail on drift;
- build the release binary;
- build the container without pushing;
- scan the repository and built image;
- upload test reports when useful;
- preserve least privilege.

Required jobs:

```text
contracts
rust-quality
unit-tests
migration-tests
integration-tests
security
container-build
release-gate
```

`release-gate` must depend on all required jobs and fail when any dependency fails.

Never write a JSON Schema validation command that validates a schema against itself.

If standalone JSON Schemas remain in the repository, validate:

1. each schema's meta-schema correctness; and
2. checked-in positive and negative fixture instances.

---

# Container workflow

Create `.github/workflows/container.yml`.

Run only:

- on version tags matching `v*`;
- or by explicit manual dispatch.

Permissions:

```yaml
contents: read
packages: write
id-token: write
attestations: write
```

Workflow must:

1. check out the exact tag;
2. build the container with BuildKit;
3. use OCI labels;
4. tag by semantic version and immutable commit SHA;
5. push to GHCR;
6. capture the exact image digest;
7. scan the pushed digest;
8. fail on configured critical/high vulnerability policy;
9. generate an SPDX or CycloneDX SBOM;
10. upload the SBOM;
11. create build provenance for the exact digest;
12. create an SBOM attestation if supported;
13. print verification commands in the workflow summary.

Never deploy by mutable tag.

Deployment references must use the image digest.

GitHub artifact attestations are provenance records. They do not replace application-level decision signatures.

---

# Release workflow

Create `.github/workflows/release.yml`.

Release must:

- require an existing version tag;
- verify CI passed for the tagged commit;
- verify the container digest;
- verify provenance;
- verify SBOM availability;
- run database migration checks;
- run smoke tests in an isolated environment;
- create a GitHub Release containing checksums, OpenAPI spec, SBOM, and migration notes;
- not deploy automatically unless an actual deployment environment has been configured;
- use GitHub Environments for staging and production;
- require manual approval for production;
- use OIDC for cloud authentication;
- never use long-lived cloud keys if OIDC is available.

Do not fabricate cloud deployment steps. If no target cloud and cluster are configured, stop at a releasable, attested container.

---

# Dependency policy

Create `deny.toml`.

Enforce:

- known advisory checks;
- explicitly approved licenses;
- no unknown Git sources;
- no unapproved duplicate major versions where avoidable;
- crates.io as the default registry.

Do not guess legal compatibility. Document the selected license policy and flag uncertain dependencies for human review.

Use Dependabot for:

- Cargo;
- GitHub Actions;
- Docker base images where supported.

---

# README requirements

The README must contain only commands that work.

Include:

1. purpose;
2. architecture;
3. trust boundaries;
4. prerequisites;
5. local startup;
6. database migration commands;
7. test commands;
8. OpenAPI generation;
9. token acquisition for the local identity provider;
10. smoke test;
11. container build;
12. threat-model summary;
13. release process;
14. limitations;
15. production-readiness checklist.

Explicitly state:

- this repository is not production-certified merely because CI passes;
- production signing keys must not use development key files;
- rate limiting requires a deployment-aware strategy;
- backup restoration must be tested in the target environment;
- external penetration testing is recommended before public launch.

---

# Threat model

Create `docs/threat-model.md`.

Cover:

- token theft;
- forged JWT;
- algorithm confusion;
- JWKS key rotation;
- tenant header spoofing;
- IDOR;
- cross-tenant data access;
- SQL injection;
- policy manipulation;
- policy downgrade;
- signature tampering;
- canonicalization ambiguity;
- ledger fork race;
- replay substitution;
- idempotency abuse;
- evidence archive path traversal;
- ZIP bombs;
- log leakage;
- dependency compromise;
- malicious pull requests;
- compromised CI action;
- container replacement;
- signing-key compromise;
- database administrator threat;
- backup exposure;
- denial of service.

For each, document:

- asset;
- attacker capability;
- attack path;
- prevention;
- detection;
- recovery;
- remaining risk.

---

# Definition of done

A milestone is done only when:

1. code compiles;
2. formatting passes;
3. Clippy passes with warnings denied;
4. unit tests pass;
5. integration tests pass where applicable;
6. migrations apply to a clean PostgreSQL instance;
7. generated OpenAPI has no drift;
8. OpenAPI lint passes;
9. security checks pass;
10. Docker image builds;
11. documentation matches implementation;
12. no TODO, `todo!()`, or `unimplemented!()` remains in release paths;
13. no endpoint handler returns a fabricated response;
14. no test is ignored without a written issue and justification;
15. changed files are summarized;
16. executed commands and their real results are reported.

---

# Required implementation sequence

Implement in this order.

## Milestone 1: Repository foundation

Deliver:

- workspace manifests;
- toolchain pin;
- crate skeletons;
- API binary with liveness and readiness;
- PostgreSQL connection;
- migration runner;
- structured logging;
- Docker Compose;
- baseline CI.

Acceptance:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
docker compose config
docker compose up --build -d
curl --fail http://localhost:8080/health/live
curl --fail http://localhost:8080/health/ready
```

## Milestone 2: Contracts and OpenAPI

Deliver:

- complete common types;
- validation constraints;
- API error model;
- generated OpenAPI exporter;
- checked-in OpenAPI snapshot;
- OpenAPI CI drift check.

Acceptance:

```bash
cargo run -p rustgate-api --bin export-openapi
git diff --exit-code -- openapi/rustgate-v1.json
redocly lint openapi/rustgate-v1.json
```

## Milestone 3: Authentication and authorization

Deliver:

- local OIDC provider;
- JWT/JWKS validation;
- scope middleware;
- tenant-claim enforcement;
- authentication integration tests.

Acceptance includes valid, expired, malformed, wrong-issuer, wrong-audience, unknown-key, missing-scope, and tenant-mismatch tests.

## Milestone 4: Tenant lifecycle

Deliver all tenant endpoints, migrations, repositories, scopes, idempotency, and tests.

## Milestone 5: Policy lifecycle

Deliver immutable policy versions, validation, signing, activation, retirement, state-machine tests, and one-active-version database enforcement.

## Milestone 6: Decision engine

Deliver deterministic evaluation, atomic persistence, signed receipt, ledger append, evidence creation, and concurrency tests.

## Milestone 7: Evidence

Deliver retrieval, verification, deterministic export, manifest signing, and tamper tests.

## Milestone 8: Replay

Deliver replay job processing, result retrieval, verification, signing, and mismatch tests.

## Milestone 9: Supply-chain release

Deliver container scanning, SBOM, provenance attestation, release assets, Dependabot, and documented verification commands.

## Milestone 10: System proof

Run the complete smoke test from a clean checkout.

Produce `docs/proof-report.md` containing:

- commit SHA;
- Rust version;
- dependency lockfile hash;
- migration status;
- OpenAPI hash;
- container digest;
- SBOM hash;
- test counts;
- smoke-test output;
- replay expected hash;
- replay actual hash;
- limitations and unresolved risks.

Do not call the system production-ready unless every required gate passes and the remaining operational controls have been evaluated for the intended deployment.

---

# Copy/paste prompt for Copilot Chat

```text
Implement Milestone 1 from `.github/copilot-instructions.md`.

Before editing:

1. Inspect the full repository tree.
2. Read all Cargo manifests, existing specifications, migrations, workflows, and documentation.
3. Identify existing files that should be preserved.
4. Present a concise implementation plan and the exact files you will create or modify.

Then implement Milestone 1 completely.

Requirements:

- Do not implement later milestones prematurely.
- Use a modular Rust workspace.
- Create a working Axum API.
- Connect to PostgreSQL with SQLx.
- Apply migrations at startup using an embedded SQLx migrator.
- Implement `/health/live` and `/health/ready`.
- Readiness must verify database connectivity.
- Add typed configuration and `.env.example`.
- Add structured tracing.
- Add graceful shutdown.
- Add Docker Compose for PostgreSQL and the API.
- Add a multi-stage Dockerfile that runs as a non-root user.
- Add baseline GitHub Actions CI.
- Do not add fake application responses.
- Do not use TODO, `todo!()`, `unimplemented!()`, or `|| true`.
- Do not claim commands passed unless you ran them.

After implementation, run:

cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
docker compose config
docker compose build

If Docker is available, also run:

docker compose up -d
docker compose ps
curl --fail http://localhost:8080/health/live
curl --fail http://localhost:8080/health/ready
docker compose down -v

Fix failures before stopping.

At completion, report:

1. files created;
2. files modified;
3. commands executed;
4. actual command results;
5. unresolved issues;
6. exact next milestone.

Do not say the application is production-ready.
```

---

# Reusable milestone prompt template

Replace `N` with the milestone number:

```text
Implement Milestone N from `.github/copilot-instructions.md`.

Inspect the current repository and verify all prior milestone gates still pass before making changes. Implement only the selected milestone, including migrations, domain logic, handlers, OpenAPI annotations, authorization scopes, unit tests, integration tests, documentation, and CI changes required by that milestone.

Do not use placeholders, fake responses, TODOs, ignored tests, `unimplemented!()`, relaxed security checks, or `|| true`.

After implementation:

1. run the complete repository check script;
2. run milestone-specific tests;
3. regenerate OpenAPI;
4. fail on OpenAPI drift;
5. build the container;
6. run the smoke-test portion supported by the completed milestones;
7. fix all failures.

Report actual results and remaining risks. Do not claim production readiness.
```

---

## Why this approach is credible

- The design deliberately starts as a modular monolith because decision, receipt, ledger, evidence, and idempotency records require strong transactional consistency. Splitting those writes across services before defining a durable coordination strategy would make the first release less reliable.
- OpenAPI is generated from implemented Rust types and routes, then checked into source control and compared in CI. Utoipa supports OpenAPI 3.1 and Axum integration.
- Migrations are explicit, checksummed artifacts. SQLx supports embedded migrations and warns that platform-specific line endings can affect reproducible migration hashes, which is why the design requires LF normalization.
- The release workflow uses exact image digests, SBOMs, and provenance. GitHub documents artifact attestations and digest-based deployment security.
- The instructions prohibit floating action references and oversized token permissions because workflow dependencies, tokens, secrets, script injection, and runner security are material parts of GitHub Actions security.

## Main design improvements

This charter replaces “generate everything at once” with gated, executable milestones. It also resolves several earlier weaknesses: wildcard administration scopes are removed, tenant headers are no longer trusted as identity, replay errors are separated from replay mismatches, cryptographic canonicalization is defined, ledger appends are concurrency-safe, and CI is prohibited from hiding failed validation.
