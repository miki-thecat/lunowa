# Lunowa Security and Reliability Verification Contracts

## Status

**Accepted product-specific verification contract.**

This document turns the currently accepted high-value security/reliability invariants into observable acceptance criteria. It is intentionally narrower than a full production-security test plan.

A contract becomes mandatory when its corresponding real product surface exists. Do not build later-phase test machinery merely to satisfy an inactive contract.

Related sources:

- `SECURITY-ARCHITECTURE.md` — trust boundaries and security invariants.
- `FAILURE-MODES.md` — current risk catalogue and activation gates.
- `CONTRACTS.md` — provider/domain/send/job semantics.
- `DATA-MODEL.md` — ownership/concurrency concepts.
- `../verification-review.md` — reusable verification baseline.

---

## 1. Verification rule

For high-risk behavior, use the evidence chain:

> **Invariant/spec -> implementation -> observable evidence -> independent review when the change can weaken the guardrail itself.**

A green build does not prove authorization, idempotency, concurrency, provider ambiguity, data isolation, or prompt-injection boundaries.

Prefer the narrowest realistic test layer that proves the actual boundary.

---

## 2. Foundation contracts

### VC-001 — Server secret is absent from client output

**Applies now. Severity: Critical.**

Given a fake canary value supplied only through a server-only secret path, a production-oriented build must not place that value in browser JavaScript, static assets, source maps, public runtime config, or HTML payloads.

Use a fake canary, never a real production key.

Maps to FM-002.

### VC-002 — Public errors do not leak sensitive internals

**Applies now; expand as integrations activate. Severity: High/Critical.**

Induce representative malformed requests and application failures. Later add DB/provider failures.

The public HTTP/UI surface must not expose stack traces, credentials/tokens, raw authorization headers, raw SQL/provider payloads containing sensitive data, unnecessary raw mail content, or sensitive internal paths.

Maps to FM-010.

### VC-003 — Guardrail changes receive stronger review

**Applies now.**

Changes to the following require explicit/fresh review proportional to risk:

- `.github/workflows/**`;
- canonical verify/test/build scripts;
- auth/session/authorization;
- secrets/configuration boundaries;
- database ownership/migrations;
- SendOperation/idempotency;
- Temporal Contract scheduling/reconciliation;
- billing/webhook verification;
- HTML/attachment isolation;
- AI privileged-action boundaries;
- deployment/security/caching boundaries.

Tests must not be weakened/deleted merely to make verification green.

---

## 3. Authorization and scope contracts

### VC-010 — Cross-user object access is denied

**Activates with auth + persistence. Severity: Critical.**

Create User A and User B with independent data. Using User A's valid session plus User B's valid identifier:

- GET returns no protected content;
- PATCH/PUT changes nothing;
- DELETE deletes nothing;
- nested Message/Attachment/ActionItem/TemporalContract/Draft/SendOperation access also remains denied when those resources exist.

Verify denied writes leave authoritative state unchanged.

Maps to FM-001 / FM-011.

### VC-011 — Scope is enforced before retrieval/context assembly

**Activates with multiple accounts/scopes and search/AI. Severity: Critical.**

Given Scope A and Scope B:

- Scope A search returns no Scope B content;
- Scope A preview/summary/context construction contains no Scope B content;
- changing client filters/IDs cannot widen the server-authorized set;
- stale/cache-derived data cannot reintroduce Scope B content into Scope A.

Test the actual retrieval/context boundary, not only final UI filtering.

Maps to FM-012.

---

## 4. Resource/cost contracts

### VC-020 — Expensive endpoint remains bounded under repeated calls

**Activates per expensive endpoint. Severity: Critical/High.**

For AI, sync, send, search, attachment processing, OAuth/reconnect, Temporal Contract execution, or another materially expensive path, define a representative burst and verify:

- concurrency remains within the intended bound;
- duplicate identical work is not multiplied unnecessarily;
- provider/model calls remain bounded;
- controlled rejection/queue/already-running behavior occurs instead of process collapse;
- no state corruption occurs.

Do not use one arbitrary global limit for every endpoint.

Maps to FM-003.

---

## 5. Database performance contracts

### VC-030 — Canonical list query count is bounded

**Activates with the real DB-backed conversation list. Severity: Medium/High.**

Using representative fixtures such as 50 and 500 rows, instrument one canonical list load.

Query count must stay bounded by the request/page shape and must not grow approximately one query per rendered row because of N+1 access.

The goal is bounded predictable access, not forcing one giant SQL query.

Maps to FM-004.

### VC-031 — Important filter/sort index is evidence-driven

**Activates when representative DB queries exist. Severity: Medium.**

For an important filter/sort path:

1. capture the actual query;
2. use a realistic fixture size;
3. inspect query plan/timing;
4. add/change an index only when evidence supports it;
5. verify the improvement and acceptable write/storage cost.

Do not create indexes for every filterable column by default.

Maps to FM-005.

---

## 6. Send and idempotency contracts

### VC-040 — Concurrent double-submit produces one logical send

**Activates with real send. Severity: Critical.**

Given one Draft and one logical send operation, submit the same operation concurrently multiple times.

Expected:

- one logical `SendOperation` owns the effect;
- at most one provider send is initiated for that logical operation;
- all callers converge on the same operation state;
- correctness does not depend on a disabled UI button.

Maps to FM-006.

### VC-041 — Ambiguous provider acceptance is not blindly retried

**Activates with real send. Severity: Critical.**

Simulate:

1. explicit provider rejection;
2. explicit success;
3. provider acceptance followed by lost/timeout acknowledgement;
4. process/worker retry after case 3.

Case 3 must enter an explicit ambiguous/reconciliation path. Case 4 must not issue an unconditional second send merely because the first call timed out.

Maps to FM-014 and the existing `SendOperation` contract.

### VC-042 — Failed send preserves recoverable draft content

**Activates with real send. Severity: High.**

Force provider/network failure after submit. The user's draft content remains recoverable and the UI exposes an accurate pending/failed/ambiguous state.

---

## 7. Concurrency contracts

### VC-050 — Conflicting writes do not silently lose authoritative state

**Activates with mutable persisted lifecycle/action state. Severity: Critical.**

Execute conflicting writes from two tabs/devices against the same authoritative object/version.

Expected behavior must be explicit: deterministic reduction/transaction semantics or stale-write rejection/re-evaluation. Silent last-write-wins is unacceptable where it can erase a meaningful lifecycle transition.

Maps to FM-009.

### VC-051 — Stale Temporal Contract worker cannot overwrite newer state

**Activates with durable Temporal Contracts. Severity: Critical.**

Arrange:

1. worker reads state/version X;
2. user/provider event creates newer state/version Y;
3. stale worker resumes.

The stale worker must no-op, conflict, or re-evaluate from current state; it must not overwrite Y.

Maps to FM-015.

---

## 8. Provider sync contracts

### VC-060 — Duplicate provider ingestion is repeat-safe

**Activates with Gmail/provider sync. Severity: High/Critical.**

Deliver the same provider message/change more than once and repeat the incremental fetch.

Expected:

- one normalized Message identity;
- no duplicate Conversation membership solely from duplicate delivery;
- no duplicate ActionItem solely from duplicate ingestion;
- downstream work is deduped or safe to repeat.

Maps to FM-013.

### VC-061 — Durable product state does not depend on one transient provider notification

**Activates when push/incremental sync becomes real. Severity: Critical for Temporal Contract resurfacing.**

Delay/omit one notification and later perform the accepted reconciliation/incremental-sync path. The mailbox change must still be discoverable; a missed transient notification must not permanently hide provider truth.

---

## 9. File/attachment contract

### VC-070 — Oversized file is rejected before unsafe buffering

**Activates with direct upload or server-side attachment processing. Severity: High.**

Submit a payload materially above the documented maximum (a controlled 200MB case is acceptable when safe/practical).

Verify:

- server/provider boundary rejects or constrains it before full unsafe in-memory buffering;
- memory/storage/background work remain bounded;
- no orphaned durable artifact/job is left behind;
- response is stable and user-safe.

Client-side validation alone does not satisfy this contract.

Maps to FM-008.

---

## 10. Billing webhook contract

### VC-080 — Billing webhook processing is authentic, idempotent, and reconcilable

**Activates only when billing exists. Severity: Critical.**

Test at minimum:

- invalid/missing provider authenticity/signature evidence -> no entitlement mutation;
- same event delivered repeatedly -> one logical commercial transition;
- retry/delay/order variation -> no blind duplicate transition;
- controlled local/provider drift -> reconciliation converges to the accepted commercial authority.

The exact tests must use the selected payment provider's official webhook semantics at implementation time.

Maps to FM-007.

---

## 11. HTML email / untrusted content contracts

### VC-090 — HTML email cannot execute with Lunowa application authority

**Activates before real HTML email rendering is considered production-ready. Severity: Critical.**

Use malicious fixtures appropriate to the chosen renderer/sanitizer/isolation strategy.

Expected:

- message content cannot execute arbitrary script in Lunowa's trusted app context;
- cannot read Lunowa session/credential data through the rendered message;
- cannot mutate the application outside its intended content boundary;
- blocked/unsupported content fails safely.

Maps to FM-019.

---

## 12. AI contracts

### VC-100 — AI output cannot expand authorization or own privileged effects

**Activates with AI interpretation/tool use. Severity: Critical.**

Feed email content that instructs the model to reveal other messages, ignore policy, send/delete content, or change lifecycle state.

Even if the model produces adversarial/wrong output:

- authorized data boundaries do not expand;
- no privileged external action occurs solely because email content requested it;
- deterministic application logic still owns authoritative lifecycle state/effects.

Maps to FM-016 / FM-017.

### VC-101 — Invalid/uncertain AI output cannot silently hide an obligation

**Activates with AI interpretation. Severity: Critical.**

Feed malformed, missing, contradictory, low-confidence, or failed model output.

Expected:

- schema/domain validation rejects or preserves uncertainty;
- invalid output does not become authoritative lifecycle state;
- missing interpretation is not interpreted as safe-to-hide;
- normal mail reading/composing remains usable if AI is unavailable.

Maps to FM-018.

### VC-102 — AI context obeys scope authorization before provider transmission

**Activates with AI + multiple account/scope data. Severity: Critical.**

Inspect the actual context/request assembly path. Unauthorized user/account/scope content must be absent **before** the request is sent to the model provider.

Maps to FM-012.

---

## 13. Logging/privacy contract

### VC-110 — Secrets and unnecessary raw mailbox content are absent from normal telemetry

**Activates progressively as real data enters the system. Severity: High/Critical.**

Generate representative auth/provider/AI/send failures and inspect actual structured logs/analytics payloads.

Normal telemetry must not emit:

- provider access/refresh tokens;
- OAuth/client/session secrets;
- full authorization headers;
- raw attachment bytes;
- unnecessary full mailbox bodies;
- database credentials.

Maps to FM-020 and FM-002.

---

## 14. Test-layer guidance

Use the narrowest layer that proves the behavior:

- **domain/unit** — lifecycle reducer, version/precondition logic, AI structured-output validation;
- **integration with real repository/DB boundary** — authorization, idempotency, concurrency, query counts;
- **adapter/contract** — Gmail/provider mapping, duplicate delivery, error/retry semantics;
- **browser E2E** — error leakage, scope switching, send/draft UX, HTML isolation where browser behavior matters;
- **build/artifact check** — secret-canary leakage;
- **load/abuse** — resource/cost/concurrency bounds;
- **runtime/provider sandbox/manual evidence** — OAuth, real webhook verification, deployment/cache/header behavior when a mock cannot prove the boundary.

A unit test of an authorization helper does not prove every endpoint uses it.

---

## 15. Definition of done for a high-risk implementation task

A high-risk task is complete only when the relevant subset is true:

1. the authoritative invariant/spec is identified;
2. the relevant Failure Mode is identified;
3. positive behavior is verified;
4. negative/adversarial behavior is verified;
5. retry/concurrency behavior is verified when repeat/race is possible;
6. public error/log output is inspected when sensitive data is involved;
7. canonical repository verification passes;
8. required CI evidence is green once branch protection is active;
9. guardrail-sensitive changes receive fresh review;
10. durable docs are updated if accepted architecture/contract changed.

Do not mark a contract verified when it is only planned, inferred from documentation, or mocked at a boundary that cannot prove the real behavior.

---

## 16. Explicitly pending

This is not the final exhaustive security verification matrix. After the planned broader zero-based Lunowa security/reliability research, add only the additional contracts that materially apply to the actual product and stack.