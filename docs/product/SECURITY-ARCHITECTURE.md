# Lunowa Security Architecture

## Status

**Accepted product-specific security architecture contract.**

This document specializes the reusable baseline in `../security-privacy.md` for Lunowa's actual trust boundaries. It defines durable security invariants, not an exhaustive checklist or an instruction to implement every later-stage control now.

Related sources:

- `ARCHITECTURE.md` — module/system ownership.
- `DATA-MODEL.md` — ownership and concurrency concepts.
- `CONTRACTS.md` — provider, sync, AI, lifecycle, scheduler, send, search, and error semantics.
- `FAILURE-MODES.md` — currently catalogued Lunowa risks and activation gates.
- `VERIFICATION-CONTRACTS.md` — observable acceptance contracts.
- `../security-privacy.md` — reusable security/privacy baseline.

---

## 1. Security objective

Lunowa handles communication that may contain sensitive personal/work/academic/account data and makes a trust-sensitive promise: communication may disappear from immediate attention and return later when action is needed.

The architecture must therefore preserve:

1. **data isolation** — one user/account/scope never receives another's protected content;
2. **credential isolation** — provider/application secrets remain outside untrusted client surfaces;
3. **authority isolation** — untrusted email/provider/AI content cannot grant authorization or own privileged actions;
4. **side-effect safety** — retries/double-submit/concurrency do not create duplicate or contradictory effects;
5. **lifecycle safety** — stale/uncertain work cannot silently hide or overwrite a real obligation;
6. **bounded execution** — bugs/abuse cannot create uncontrolled provider/AI/resource cost;
7. **safe observability** — errors/logs/analytics do not become a secondary sensitive-data channel.

Implement controls when the corresponding feature becomes real. Preserve the architectural boundary from the beginning.

---

## 2. Trust boundaries

### Browser / client

The browser is not trusted for:

- authorization;
- provider credentials/secrets;
- lifecycle authority;
- Temporal Contract execution;
- send idempotency;
- privileged provider mutations.

Client-visible values are assumed observable and modifiable. Browser-only validation, hidden routes, disabled buttons, or unpredictable IDs are UX aids, not correctness/security controls.

### Lunowa application/API

The server-side application boundary owns:

- authenticated Lunowa identity;
- object/action authorization;
- account/scope ownership resolution;
- privileged provider operations;
- authoritative lifecycle/Temporal Contract mutations;
- server-only secrets and credentials;
- side-effect/idempotency coordination.

Authentication answers **who** the actor is. Authorization separately answers whether that actor may perform this operation on this resource.

### Mailbox provider

Gmail and future providers are authorities for mailbox facts, not Lunowa workflow state.

Provider payloads, message bodies, HTML, attachments, sender names, headers, push notifications, and IDs are external input and must be validated/handled as untrusted data at the application boundary.

Mailbox OAuth credentials are distinct from Lunowa login/session credentials.

### AI

AI output is untrusted structured input.

AI may propose facts such as action, owner, deadline, completion/waiting signal, summary, provenance, and confidence. It must not directly:

- authorize access;
- widen account/scope retrieval;
- send/delete provider content;
- own billing/entitlement;
- execute instructions embedded in email content;
- become authoritative lifecycle state without deterministic rules.

> **AI understands. Deterministic application logic decides authoritative state and privileged effects.**

### Durable/background execution

A worker is not trusted merely because it runs server-side.

Before applying a privileged mutation or external side effect, durable jobs must re-read current authoritative state and re-check relevant ownership/version/preconditions. Stale work must become a no-op, conflict, or re-evaluation path rather than overwriting newer truth.

---

## 3. Authorization architecture

### Object-level authorization

UUIDs/unpredictable IDs do not solve authorization.

Every privileged operation using a client-controlled resource ID must scope access through trusted ownership relationships, conceptually such as:

`User -> Scope -> ConnectedAccount -> Conversation -> Message / Attachment / ActionItem / TemporalContract / Draft / SendOperation`

The exact schema may evolve; the invariant does not.

### Scope before retrieval

Account/scope authorization must apply **before** protected content enters:

- search results;
- previews/summaries;
- AI context;
- attachment/message fetches;
- background processing.

Filtering after broad cross-account retrieval is not an acceptable isolation boundary.

### Negative behavior

Cross-user/cross-account access must return no protected content and produce no protected state change. Where product semantics do not require disclosure that another user's object exists, avoid turning errors into an object-enumeration oracle.

### Database defense in depth

Application authorization remains mandatory. Database-level mechanisms such as RLS may be added selectively if they materially reduce blast radius and remain understandable/testable with the chosen persistence architecture. They are not a substitute for application authorization semantics.

---

## 4. Secret and credential architecture

Server-side secrets include, unless a provider explicitly defines a public/publishable counterpart:

- database credentials;
- OAuth client secrets;
- mailbox access/refresh tokens;
- model-provider secret keys;
- payment secret/webhook keys;
- application signing/encryption/session secrets;
- background-job/operational credentials.

They must not appear in:

- client-exposed environment namespaces;
- browser bundles/static assets/source maps;
- public runtime config;
- source control;
- normal logs/analytics;
- browser storage;
- ordinary test fixtures.

Public identifiers such as OAuth client IDs, analytics project IDs, or payment publishable keys may intentionally be public depending on provider semantics; do not confuse them with privileged credentials.

Ordinary coding-agent environments should not receive production credentials.

---

## 5. Untrusted communication content

### HTML email

Real email HTML is hostile input. It must not gain script execution, application-origin DOM authority, access to Lunowa session/credential data, or privileged navigation/action capability.

The final implementation must use an isolation/sanitization strategy appropriate to the chosen renderer and browser/runtime. Do not treat successful display as proof of safe rendering.

### Attachments

Attachments remain untrusted even when fetched from an authenticated mailbox provider.

When attachment/upload handling becomes real, define and enforce server/provider-boundary limits for the relevant subset of:

- bytes per file/request;
- file count;
- handled content types;
- processing time/memory;
- archive/decompression behavior if supported;
- storage/retention/deletion.

Do not read arbitrarily large payloads fully into memory before rejection.

### AI prompt injection

Email bodies, quoted text, attachments, and retrieved web content may contain instructions targeting the model. They are data, not trusted system instructions.

Privileged action/authorization/lifecycle gates must live outside model text so prompt injection cannot expand authority.

---

## 6. Side-effect safety and idempotency

Any non-trivially reversible external effect must define:

- logical operation identity;
- retry semantics;
- uniqueness/idempotency boundary;
- ambiguous-acceptance behavior;
- reconciliation authority.

High-risk examples include send, provider mailbox mutation, billing/entitlement mutation, account linking/unlinking, deletion/export, and durable resurfacing transitions.

UI button disabling is only a convenience control.

For email send, preserve the existing `SendOperation` contract: concurrent double-submit, transport retry, provider ambiguity, and worker retry must not blindly create duplicate messages. Provider acceptance with lost acknowledgement is an explicit ambiguous state that requires reconciliation before resend.

---

## 7. Concurrency and stale work

Assume the same logical state can be changed by:

- two browser tabs/devices;
- user action plus mailbox sync;
- incoming provider events;
- AI interpretation completion;
- Temporal Contract/background workers;
- retry/reconciliation jobs.

Critical state transitions need an explicit concurrency strategy appropriate to the invariant: version/precondition checks, transaction semantics, unique constraints, row locks, idempotency records, or deterministic reduction from authoritative state/events.

A database transaction alone does not define product conflict semantics.

Particularly important:

- stale Temporal Contract work cannot overwrite newer state;
- duplicate provider ingestion cannot duplicate normalized messages/actions;
- concurrent sends converge on one logical SendOperation;
- meaningful lifecycle transitions cannot be silently lost by accidental last-write-wins behavior.

---

## 8. Resource and cost containment

Before public exposure, materially expensive paths need explicit bounds appropriate to their cost and side effects.

Likely hotspots:

- AI interpretation/summarization;
- mailbox sync/reconciliation;
- send;
- search;
- attachment transfer/processing;
- OAuth/reconnect;
- Temporal Contract scheduling/re-evaluation;
- future billing/webhook processing.

Possible controls include per-user/account/operation rate limits, request/body limits, bounded concurrency, dedupe/coalescing, provider-aware backoff, hard execution limits, and quotas where economically necessary.

Do not rely only on after-the-fact cost alerts when a loop can spend materially faster than the operator can react.

---

## 9. Cache/search/derived-state isolation

Any cache, search index, preview/summary store, or AI-derived projection containing user communication must preserve the same authorization boundary as the authoritative source.

Derived state must not become an authorization shortcut.

Personalized cache keys must include the identity/account/scope dimensions required to prevent cross-user/cross-account reuse. Search/AI projections should resolve back to authoritative authorized records before protected content is exposed.

---

## 10. Errors, logs, analytics, and support

### Public errors

Production-facing errors should expose stable user-safe semantics and an error/request identifier when useful, not stack traces, credentials, raw SQL/provider responses, sensitive paths, or raw mailbox content.

### Logs/analytics/support

Do not emit by default:

- access/refresh tokens;
- secrets/authorization headers;
- database credentials;
- raw attachment bytes;
- unnecessary raw email bodies;
- model prompts containing unnecessary mailbox content.

Prefer metadata/event-oriented observability sufficient to diagnose failures without duplicating the user's mailbox into operational systems.

---

## 11. Failure posture

Security/reliability-sensitive failure should move toward visible, bounded, recoverable states.

Examples:

- uncertain/failed AI -> preserve uncertainty; do not silently hide an obligation;
- AI unavailable -> normal mail reading/composing remains usable;
- ambiguous send -> reconcile before blind resend;
- stale worker -> no-op/re-evaluate from current authoritative state;
- authz uncertainty -> fail closed;
- missing secret/config -> fail safely rather than substitute a client-visible fallback;
- oversized/unhandled content -> reject or use a constrained fallback.

---

## 12. Activation by feature

| Feature/stage | Security/reliability controls that become mandatory |
| --- | --- |
| Current runtime foundation | server/client secret boundary; safe public errors; CI guardrails; no production credentials in ordinary agent context |
| Auth + persistence | object authorization; cross-user negative tests; ownership-aware queries; concurrency strategy |
| Gmail read/sync | credential protection; provider input validation; idempotent ingestion/reconciliation; account/scope isolation |
| Real HTML/attachments | untrusted-content isolation plus file/resource limits before public use |
| Real send | SendOperation idempotency; double-submit/concurrency tests; ambiguous provider acceptance/reconciliation; draft preservation |
| Temporal Contracts/jobs | persisted promises; stale-worker/version protection; retry/reconciliation; bounded execution |
| AI interpretation | structured validation; authorized context construction; prompt-injection boundary; uncertainty fallback; cost bounds |
| Billing | provider authenticity verification; webhook idempotency/reconciliation; explicit commercial authority |

Do not activate later-feature infrastructure merely because it appears in this table.

---

## 13. Review triggers

Revisit this document when a change materially alters:

- auth/account recovery or ownership;
- Gmail/Microsoft scopes/token handling;
- public APIs/webhooks;
- HTML/attachment/remote-content processing;
- AI tool/action capability;
- search/caches containing mailbox content;
- durable jobs/Temporal Contracts;
- payment/entitlement;
- deletion/export/retention;
- deployment/runtime trust boundary;
- CI/secret permissions;
- a new sensitive third-party data processor.

Do not add a permanent control solely because a generic checklist contains it. Tie the control to a real Lunowa asset, boundary, failure mode, or release surface.