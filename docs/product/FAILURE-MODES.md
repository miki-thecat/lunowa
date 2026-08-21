# Lunowa Production Failure Mode Catalogue

## Status

**Living product-specific risk catalogue.**

This document records only failure modes that are already supported by current Lunowa architecture/contracts or by the focused risk analysis completed before this change. It is intentionally **not yet an exhaustive security audit**.

Broader zero-based threat/security/reliability research should expand this catalogue later. Do not promote speculative checklist items into durable product requirements without evidence that they apply to Lunowa.

Related sources:

- `SECURITY-ARCHITECTURE.md` — durable security boundaries/invariants.
- `VERIFICATION-CONTRACTS.md` — observable acceptance contracts.
- `ARCHITECTURE.md`, `DATA-MODEL.md`, `CONTRACTS.md` — authoritative product behavior and state ownership.
- `../security-privacy.md`, `../reliability-operability.md`, `../production-readiness.md` — reusable baseline.

---

## 1. Priority and activation

### Severity

- **Critical** — plausible cross-user data exposure, credential compromise, duplicated/irreversible external effect, silent corruption of authoritative state, silent failure of Lunowa's core trust promise, or severe uncontrolled cost.
- **High** — material privacy/reliability/core-flow failure that must be prevented before affected external use.
- **Medium** — meaningful performance/operability degradation, normally recoverable without severe trust damage.

### Activation

- **FOUNDATION** — invariant/control belongs in the engineering foundation now.
- **FEATURE-GATE** — mandatory when the corresponding real feature exists.
- **PUBLIC-GATE** — mandatory before meaningful external/public use of the affected surface.
- **PAID-GATE** — mandatory when billing/entitlement becomes real.

The existence of an entry does not mean its implementation belongs in Phase 0.

---

## 2. Focused risk set

These are the ten concrete concerns analyzed before this document was created.

| ID | Failure mode | Severity | Gate | Correctness/control direction | Verification direction |
| --- | --- | --- | --- | --- | --- |
| FM-001 | Changing a URL/API resource ID exposes another user's Conversation/Message/Attachment/ActionItem | Critical | FEATURE-GATE: auth + persistence | server-side object authorization through authoritative ownership; IDs are not authorization | User A session + User B IDs for GET/PATCH/DELETE; no data leak/state change |
| FM-002 | Server API key/secret appears in client bundle, source map, public config, logs, or artifacts | Critical | FOUNDATION | server-only secret boundary; explicit public-vs-secret config | fake canary secret absent from built client assets/artifacts/logs |
| FM-003 | Bug/attacker loops an expensive endpoint and exhausts CPU/provider quota/model budget | Critical | FEATURE/PUBLIC-GATE | per-actor/operation bounds, concurrency limits, dedupe, payload limits, hard cost bounds where needed | representative burst; bounded provider calls/cost; controlled rejection; no corruption |
| FM-004 | Conversation list executes one or more extra DB queries per row (N+1) | Medium/High | FEATURE-GATE: real DB list | bounded query shape, batching/join/select only what is needed | query-count instrumentation with representative page sizes |
| FM-005 | Important filter/sort query lacks a useful index and degrades with data size | Medium | FEATURE-GATE: real DB query | query-pattern-driven indexing, not blanket indexes | realistic fixture + SQL/query plan evidence before/after |
| FM-006 | Rapid double-submit/retry creates duplicate email or another external effect | Critical | FEATURE-GATE: real send/effect | server-owned operation/idempotency identity + uniqueness/transaction semantics | parallel same-operation requests converge to one logical effect |
| FM-007 | Billing webhook is forged, duplicated, retried, delayed, or processed in unsafe order | Critical | PAID-GATE | provider authenticity verification, dedupe/idempotency, durable processing, reconciliation | invalid-signature + duplicate/retry/order/reconciliation tests |
| FM-008 | Oversized upload/attachment exhausts memory, storage, or processing time | High | FEATURE/PUBLIC-GATE: files | early server/provider-boundary size/count/type/time limits; avoid full unsafe buffering | oversized request rejected safely; memory/storage/jobs remain bounded |
| FM-009 | Two tabs/devices/jobs overwrite each other's authoritative state | Critical | FEATURE-GATE: mutable persistence | explicit version/precondition/transaction/idempotency semantics | conflicting concurrent mutation test; stale work cannot silently win |
| FM-010 | Production error exposes stack trace, token, SQL/provider response, path, or raw mail content | High/Critical | FOUNDATION/PUBLIC-GATE | stable public error envelope + sanitized server logging | induced 4xx/5xx/provider/DB failures inspected at HTTP/UI/log boundaries |

---

## 3. Existing Lunowa-specific trust failures

The following risks are already implied by accepted Lunowa product architecture/contracts and therefore belong in this catalogue even though they were not part of the original ten-item list.

| ID | Failure mode | Severity | Gate | Required invariant |
| --- | --- | --- | --- | --- |
| FM-011 | Authentication is checked, but object/action authorization is not | Critical | FEATURE-GATE: auth + persistence | every privileged read/write is re-authorized from trusted app state |
| FM-012 | Work/personal/other account data leaks across Scope in search, preview, summary, or AI context | Critical | FEATURE-GATE: multi-account retrieval | Scope/account authorization applies **before** retrieval/context exposure |
| FM-013 | Duplicate provider delivery/re-fetch duplicates Messages, ActionItems, or downstream work | High/Critical | FEATURE-GATE: Gmail sync | ingestion and downstream processing are idempotent/repeat-safe |
| FM-014 | Provider accepted a send but Lunowa times out and blindly retries | Critical | FEATURE-GATE: real send | ambiguous acceptance is distinct state; reconcile before resend |
| FM-015 | Stale Temporal Contract/background worker overwrites newer user/provider state | Critical | FEATURE-GATE: durable jobs | job re-reads authoritative current state/version; stale work no-ops or re-evaluates |
| FM-016 | AI result directly becomes lifecycle authority or privileged side effect | Critical | FEATURE-GATE: AI | AI supplies candidate facts only; deterministic rules/authorization own state/effects |
| FM-017 | Malicious email/attachment/web content is treated as trusted AI instruction | Critical | FEATURE-GATE: AI/content | communication content is untrusted data; prompt text cannot expand authorization or tool authority |
| FM-018 | AI failure/uncertainty silently hides a real user obligation | Critical | FEATURE-GATE: AI | uncertainty/missing interpretation cannot equal safe-to-hide; normal mail remains usable |
| FM-019 | Raw HTML email executes with Lunowa application authority | Critical | FEATURE-GATE: real HTML rendering | HTML is untrusted; rendering must isolate/sanitize it from app/session authority |
| FM-020 | Logs/analytics/support tooling becomes a secondary mailbox/token data leak | High/Critical | PUBLIC-GATE as real data enters system | metadata-first telemetry; secrets/tokens/raw content minimized/redacted |

---

## 4. Implementation timing

### Foundation / current runtime

Keep now:

- server/client secret boundary;
- production-safe public error behavior;
- canonical CI guardrails and protected verification checks;
- no production credentials in ordinary agent contexts;
- no browser-only enforcement for future privileged behavior.

### Auth + real persistence

Activate:

- FM-001 / FM-009 / FM-011;
- cross-user negative tests;
- ownership-aware query patterns;
- explicit concurrency strategy for mutable authoritative state;
- N+1/query-plan/index evidence only once representative queries exist.

### Gmail read/sync

Activate:

- FM-012 / FM-013;
- provider payload/cursor/reconciliation semantics;
- real HTML/attachment safety before exposing those surfaces publicly.

### Real send

Activate:

- FM-006 / FM-014;
- `SendOperation` idempotency;
- provider ambiguity and retry/reconciliation tests;
- draft preservation on failure.

### Temporal Contracts/background work

Activate:

- FM-009 / FM-015;
- stale-worker/version tests;
- durable persisted promise and reconciliation behavior;
- bounded retry/work execution.

### AI

Activate:

- FM-003 / FM-012 / FM-016 / FM-017 / FM-018;
- structured-output/domain validation;
- authorized context construction;
- prompt-injection boundary;
- usage/cost bounds.

### Attachments/uploads

Activate FM-008 and the selected product-specific limits before public use.

### Billing

Activate FM-007 only when billing is actually implemented; do not build billing security infrastructure before the product needs billing.

---

## 5. Performance rule

N+1 and indexing are important but should stay measurement-driven.

For the canonical list/search flows:

1. capture the actual query pattern;
2. use representative fixture sizes;
3. instrument query count and latency;
4. inspect the query plan when persistence exists;
5. optimize only the observed problem;
6. do not introduce caches, denormalization, search clusters, or broad index sets merely because they are common production techniques.

---

## 6. How to add a new failure mode

Add a durable entry only when it materially changes implementation/release decisions. Record:

1. **what asset/property is harmed;**
2. **what realistic trigger causes it;**
3. **what source of truth decides correct behavior;**
4. **what minimal preventive/containment control applies;**
5. **what observable verification proves the control;**
6. **when the control becomes mandatory.**

### Explicitly pending

A broader zero-based Lunowa audit is still pending. That future research should consider additional browser/API/OAuth/content-parsing/supply-chain/privacy/recovery/availability/AI/payment failure classes using current primary sources, then add only the subset that materially applies to Lunowa.

Until that audit is performed, absence from this file means **not yet catalogued**, not proven safe.