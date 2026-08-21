# Human-light Merge Gate

## Status

**Accepted initial merge policy for AI-heavy development.**

This file defines the Lunowa-specific risk tiers and merge evidence contract. General test/review philosophy remains in `verification-review.md`; general agent workflow remains in `coding-agent-harness.md`.

Related sources:

- `agent-permission-model.md`
- `verification-review.md`
- `coding-agent-harness.md`
- `product/ARCHITECTURE.md`
- `product/CONTRACTS.md`

`guardrail-integrity.md` is the protected-surface enforcement layer after its dedicated signer is verified.

---

## 1. Principle

> **Human-light does not mean evidence-light.**

A human should not need to prove generated-code correctness by reading every line. Merge confidence should come from accepted behavior, executable constraints/tests, runtime evidence where relevant, fresh review when useful, and focused human judgment where the change can violate an important trust or production boundary.

Passing CI is necessary, not universally sufficient.

---

## 2. Common evidence bundle

Every non-trivial PR should make these facts easy to inspect:

```text
Behavior changed:
Risk tier:
Acceptance criteria:
Checks actually run:
Runtime/integration evidence:
Independent review:
Protected surfaces changed:
Anything not verified:
Rollback/recovery when relevant:
Durable docs changed:
```

Do not report mocked, skipped, inferred, or differently-scoped verification as if it were executed evidence.

---

## 3. Risk tiers

Risk is based on **behavior, authority, side effects, and reversibility**, not merely file type or line count. If uncertain, use the higher tier until the uncertainty is resolved.

| Tier | Typical changes | Mandatory evidence | Human role |
| --- | --- | --- | --- |
| **R0 — trivial/editorial** | typo/copy; formatting-only docs; non-semantic cleanup | relevant lightweight check; no protected/runtime/security/data change | intent + checks; full diff optional |
| **R1 — ordinary reversible** | isolated UI/component behavior; fake-data slice; low-risk refactor/helper code; accessible/responsive improvement | acceptance criteria; `Verify`; relevant targeted tests; `E2E Smoke`/Playwright/runtime evidence when user-visible; fresh reviewer for non-trivial work; no unresolved Guardrail failure | review outcome/evidence; line-by-line diff normally optional |
| **R2 — high-impact/trust-sensitive** | auth/OAuth; provider credentials/sync; send/idempotency; Lifecycle; Temporal Contract; migrations; deletion/retention; encryption; billing; CI/tooling/dependency execution policy; deployment/major architecture | reviewed design/Task Contract when needed; R1 evidence; adversarial fresh review; negative/failure/integration/migration/idempotency evidence as applicable; rollback/forward recovery; Guardrail exact-head approval when covered | inspect the **risk-bearing boundary + evidence**, not unrelated generated code |
| **R3 — irreversible/production-dangerous** | destructive production migration/data deletion; key destruction/rotation; broad prod IAM/secret change; direct prod DB repair; material payment execution; DNS/cloud-admin destruction; protection bypass | explicit human design/authorization; dry-run/staging/canary where possible; backup/recovery evidence; separate production authorization | human owns decision/execution authorization; no direct coding-agent execution by default |

The GitHub Ruleset is the mechanical floor. R2/R3 may require more evidence than the Ruleset can express.

---

## 4. Current mechanical merge floor

Once the repository-safety setup is complete, a normal merge candidate must have:

- PR-based change to `main`;
- branch up to date with protected `main` while strict status checks are enabled;
- `Verify = success`;
- `E2E Smoke = success`;
- `Guardrail Integrity = success` after that mechanism is enforced;
- no merge conflict;
- no unresolved blocking review finding;
- tier-appropriate evidence from the table above.

For R1 non-trivial and all R2 work, the independent reviewer should start from fresh context, inspect the accepted goal/current repository/diff, and challenge both conformance and the requested design itself. Fresh review is evidence, not a substitute for executable verification.

---

## 5. Testing Oracle risk

Builder-written tests can encode the same misunderstanding as builder-written code. For important/R2 changes, reduce this by using the strongest applicable external authority:

- acceptance criteria from durable product/provider contracts before implementation;
- observable behavior rather than private implementation details;
- negative/authorization/failure scenarios;
- official provider contracts for integration behavior;
- browser/runtime evidence for UX;
- database constraints/transactions for durable invariants;
- migration/idempotency/reconciliation tests for persistence/workflow behavior;
- held-out representative eval cases for AI behavior while deterministic application logic retains authority.

Builder implementation + builder tests alone are not sufficient R2 evidence.

---

## 6. Current merge authority

**No auto-merge. No agent-controlled merge.**

The human owner performs the final merge after the evidence bundle is complete. One human merge action is currently cheap, and Guardrail/permission/rollback evidence is not mature enough to remove that stop point.

Consider agent/auto-merge for R0/R1 only after all are demonstrably true:

- `main` Ruleset is active/reliable;
- Guardrail expected-source spoof tests pass;
- builder identity cannot issue human Guardrail approval;
- required CI is deterministic enough that bypass/rerun rituals are rare;
- relevant runtime/browser evidence is routinely generated;
- revert/rollback path has been exercised;
- independent review is available where required;
- real merged-change history shows the gates catch the failure modes that matter.

Do not use an arbitrary PR-count threshold as safety evidence. R2/R3 remain human-gated even if R0/R1 automation later increases.

---

## 7. Failure / flake rule

Do not rerun a failed required check until it happens to pass without understanding why it failed.

A recurring flaky gate is itself a reliability defect. Inspect the failure, distinguish product failure from infrastructure flake, fix systemic flakes, and never weaken/skip a gate merely to unblock a PR.

---

## 8. Lunowa-specific R2 promises

Treat changes to these promises as at least R2 even before their final file paths exist:

- `AI understands; rules decide state` authority boundary;
- ActionItem lifecycle authority and reopen/completion semantics;
- Temporal Contract persistence, trigger, idempotency, cancellation, reconciliation, and resurfacing;
- send idempotency / duplicate-send prevention;
- provider sync cursor/reconciliation and missed-event recovery;
- mailbox credential ownership/access;
- cross-account/scope authorization;
- email HTML/attachment trust boundary;
- user-data retention/deletion;
- any mechanism that can silently hide a real user obligation.

When these concerns get stable concrete module paths, add those paths to Guardrail Integrity without forcing human approval for unrelated UI/test work.
