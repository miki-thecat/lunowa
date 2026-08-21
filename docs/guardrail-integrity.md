# Guardrail Integrity

Lunowa is developed with substantial AI coding-agent execution and limited line-by-line human review. Stable verification and security controls therefore need protection from the same agents whose changes they evaluate.

This document defines the minimal **Guardrail Integrity** policy for the current solo-developer repository.

## Threat model

A generated change can appear healthy while weakening the system that judges it. Examples include:

- replacing a real CI command with a no-op;
- deleting or weakening lint/type/test configuration;
- changing GitHub Actions so required checks always pass;
- broadening OAuth/auth/security behavior without explicit judgment;
- changing database migration or deletion behavior;
- adding dependency/install-script capability without review;
- modifying architecture/security source-of-truth docs to normalize an unsafe implementation.

The policy does **not** assume malicious intent. Accidental optimization toward “make CI green” is sufficient reason to isolate these controls.

## Core rule

Ordinary product changes should be able to merge from mechanical evidence without line-by-line human review.

Changes to a **protected surface** require an explicit human judgment event tied to the exact PR head SHA in addition to normal CI.

The approval is not evidence that every changed line is correct. It means a human intentionally accepted that this PR is allowed to modify a high-impact boundary and has reviewed the relevant risk/evidence.

## Protected surfaces — current baseline

The enforcement workflow is the executable source of truth for exact path matching. The baseline protects these categories:

### Repository / agent / verification control

- `.github/**`
- `AGENTS.md`
- `package.json`
- `pnpm-lock.yaml`
- `pnpm-workspace.yaml`
- `.gitignore`
- `.env.example`
- `.node-version`
- TypeScript / ESLint / Next / PostCSS / Vitest / Playwright configuration
- `docs/coding-agent-harness.md`
- `docs/verification-review.md`
- this document

### Durable architecture / security decisions

- `docs/security-privacy.md`
- `docs/product/ARCHITECTURE.md`
- `docs/product/CONTRACTS.md`
- `docs/product/DATA-MODEL.md`
- `docs/product/TECH-STACK.md`
- `docs/decisions/**`

### Security / persistence / privileged product code

Current/future paths under:

- `drizzle/**`
- `migrations/**`
- `src/db/**`
- `src/proxy.ts`
- `src/**` path segments named `auth`, `oauth`, `security`, `billing`, `payments`, `encryption`, `crypto`, or `migrations`

Do not protect every test or application file. That would turn the Human-light model back into manual review of ordinary feature work. Test **infrastructure** is protected; ordinary behavior tests remain part of normal implementation evidence.

Update the protected set when the repository acquires a new high-impact boundary. Do not expand it for one-off anxiety without a repeatable failure mode.

## Enforcement architecture

`.github/workflows/guardrail-integrity.yml` runs from the trusted **base repository/default-branch context** on PR metadata events.

It intentionally:

- does **not** checkout PR code;
- does **not** install PR dependencies;
- does **not** execute PR scripts/configuration;
- does **not** use application/provider secrets;
- reads PR metadata/files/comments through the ordinary read-only `GITHUB_TOKEN`;
- posts the `Guardrail Integrity` commit status using a **dedicated GitHub App installation token**, not `GITHUB_TOKEN`.

The workflow uses `pull_request_target` only for this metadata-policy purpose. Using `pull_request_target` for build/test execution remains prohibited unless separately designed and reviewed.

Once the workflow is on `main`, a PR that edits the guardrail workflow itself is still evaluated by the pre-existing trusted version from `main`.

### Why the signer is a separate GitHub App

A commit-status context is not tamper-resistant merely because the trusted base workflow produced it first. GitHub evaluates the latest status for a context, and repository write-capable workflows can request `statuses: write`. If the trusted policy used the same GitHub Actions identity as PR-controlled workflows, a PR could potentially post a later same-name success status.

Therefore the Ruleset must not trust `Guardrail Integrity` from “any source” or from the generic GitHub Actions App. The status must be signed by a dedicated GitHub App whose credential is unavailable to PR-controlled workflows, and the Ruleset must bind the required status to that App as its **expected source**.

## Signer credential isolation

The dedicated App credential is stored in the GitHub Environment:

```text
guardrail-trust
```

Required environment configuration:

- repository: public `miki-thecat/lunowa`;
- deployment branches/tags: **Selected branches and tags**;
- allowed branch: **`main` only**;
- no `refs/pull/*/merge` rule;
- environment secret: `GUARDRAIL_APP_PRIVATE_KEY`;
- repository or environment variable: `GUARDRAIL_APP_CLIENT_ID`;
- administrators should not casually bypass environment branch protection.

The workflow references the environment with `deployment: false`; the environment is being used as a credential boundary rather than as a product deployment.

This separation depends on GitHub's event refs:

```text
pull_request        -> refs/pull/<number>/merge
pull_request_target -> refs/heads/main (base/default branch)
```

Environment deployment-branch rules are matched against `GITHUB_REF`. Therefore an ordinary PR-controlled workflow cannot obtain the App private key simply by adding `environment: guardrail-trust`; its PR merge ref does not satisfy the `main`-only policy.

### Dedicated GitHub App permissions

Create a private GitHub App dedicated to this purpose and install it only on the Lunowa repository.

Keep permissions minimal:

- Repository metadata: read (implicit/required by GitHub App platform)
- Commit statuses: **read and write**
- all other repository/organization permissions: none unless GitHub requires a narrower supporting permission
- webhooks: not required for this design

The workflow uses GitHub's official `actions/create-github-app-token` action, pinned to an immutable commit, and requests only `statuses: write` for the installation token.

The App private key is a **guardrail credential**, not an application secret. Coding agents must never receive it.

## Approval protocol

If no protected path changed, the dedicated App posts:

```text
Guardrail Integrity = success
```

If a protected path changed, it posts failure until the repository owner comments on that PR with exactly:

```text
guardrail-approved:<FULL_PR_HEAD_SHA>
```

Example:

```text
guardrail-approved:0123456789abcdef0123456789abcdef01234567
```

The approval must:

- come from the repository owner;
- have GitHub author association `OWNER`;
- match the current full PR head SHA exactly.

A later push changes the head SHA, so an older approval no longer authorizes the new version. The owner must intentionally approve the new head again.

### Agent rule

Builder/reviewer agents MUST NOT create, edit, or imitate the `guardrail-approved:<sha>` approval comment.

The Agent Permission Model must preserve this boundary by keeping ordinary coding-agent credentials away from repository-owner comment/administration capability where practical.

## Merge gate

After this mechanism is verified, the `main` Ruleset should require all of:

- `Verify`
- `E2E Smoke`
- `Guardrail Integrity`

Required-status source configuration:

- `Verify` -> GitHub Actions
- `E2E Smoke` -> GitHub Actions
- `Guardrail Integrity` -> **the dedicated Guardrail GitHub App**

Do not select “any source” for `Guardrail Integrity`.

## Failure-safe behavior

If Guardrail Integrity cannot evaluate the PR, obtain its isolated signer credential, or post its status, it must not manufacture success. Missing/failed required status should block merge once the Ruleset is active.

Do not add path filters that can cause the required check not to run on some PRs.

## What this does not solve

Guardrail Integrity is one layer, not proof of correctness. It does not replace:

- acceptance criteria;
- domain/integration/E2E tests;
- security analysis;
- dependency review;
- independent review where worthwhile;
- runtime evidence;
- rollback/recovery planning;
- human judgment for high-impact product/security changes.

It also relies on two permission boundaries:

1. coding agents cannot obtain the `guardrail-trust` Environment secret;
2. coding agents cannot impersonate the repository-owner approval comment.

These boundaries are part of the Agent Permission Model and must be re-audited if the development toolchain or GitHub connection changes.

## Bootstrap / verification before enforcement

Do not merge or require the Guardrail status merely because the workflow file exists. Establish the external trust boundary first.

1. Create/install the dedicated Guardrail GitHub App with only commit-status write capability.
2. Create the `guardrail-trust` Environment.
3. Restrict the Environment to the `main` branch only.
4. Store the App private key in `GUARDRAIL_APP_PRIVATE_KEY` and its Client ID in `GUARDRAIL_APP_CLIENT_ID`.
5. Merge the trusted base workflow after normal CI and focused review.
6. Open an ordinary non-protected test PR and confirm the dedicated App sets `Guardrail Integrity = success`.
7. Open a disposable PR that changes a protected surface and confirm the App sets `failure` without approval.
8. Add an exact owner approval for that test head SHA and confirm the App changes the status to `success`.
9. Push another commit and confirm the old approval no longer authorizes the new SHA.
10. Attempt a disposable PR workflow that requests `statuses: write` and posts a same-name success with `GITHUB_TOKEN`; confirm the Ruleset expected-source selection distinguishes it from the dedicated App status.
11. Close all disposable test PRs without merging.
12. Only after this evidence, require the three statuses in the `main` Ruleset.

## Repository ownership caveat

The current approval rule assumes this repository remains owned by the personal account `miki-thecat`, where GitHub marks the owner's comments with association `OWNER`.

If Lunowa moves to an organization or gains additional trusted human maintainers, redesign the approver identity rule before relying on this mechanism. Do not silently broaden approval to any collaborator.
