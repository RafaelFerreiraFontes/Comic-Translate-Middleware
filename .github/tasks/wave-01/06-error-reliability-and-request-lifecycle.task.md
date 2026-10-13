# Wave 01 — Task 06 — Error, Reliability and Request Lifecycle

## Status

COMPLETED

## Objective

Define normalized failures, lifecycle state transitions, and reliability ownership before any provider or retry implementation.

## Why This Task Exists

SDK exceptions, stream events, and process failures must converge on one internal error contract and safe public errors, without duplicate retries or partial results presented as success.

## Dependencies

- [Task 02 — Canonical Domain Contracts](02-canonical-domain-contracts.task.md): accepted deliverable and completion evidence.
- [Task 03 — Compatibility Layer Contract](03-compatibility-layer-contract.task.md): accepted deliverable and completion evidence.
- [Task 04 — Provider Interface and Routing](04-provider-interface-and-routing.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [fastapi-reference-python.md](../../instructions/fastapi-reference-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Implementation & Testing agent](../../agents/implementation-testing.agent.md).
- [Coordinator agent](../../agents/coordinator.agent.md).
- [Google / Antigravity agent](../../agents/google-antigravity.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Architecture & Security
- Reviewer: Implementation & Testing
- Optional specialist: Google / Antigravity — only for unresolved CLI failure/termination semantics.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Exception taxonomy, HTTP/error mapping, lifecycle/state diagrams, retry/cancellation ownership, and failure-scenario specifications.

During later execution, produce the named architecture document and completion
evidence; task/index status updates follow Coordinator acceptance. The present
task-set creation does not execute this task or create its future deliverables.

### Out of Scope

No application code, FastAPI runtime endpoints, OAuth flows, OpenAI/Google provider
implementation, subprocess execution, production Pydantic models, tests/fixture
files, dependency installation, deployment/packaging, or Comic Translate source
changes. Do not modify project instructions or custom agents. No branch, commit,
or push unless separately explicitly authorized.

## Architecture Questions to Resolve

- Which failure categories are stable across SDK and subprocess transports?
- How do provider errors become internal errors and OpenAI-compatible HTTP errors?
- Who owns request IDs, deadlines, cancellation, retries, and process cleanup?
- What constitutes completion versus failed/truncated/malformed upstream output?
- Which metadata can be logged or returned safely?

## Decision required in this task

Error taxonomy/envelopes, lifecycle ownership, timeout/cancellation semantics, retry safety and bound policy, and safe observability fields.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Specific retry counts/backoff tuning, HTTP client settings, process termination APIs, concrete exception classes, log library, and executable tests.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Define an exception hierarchy rooted in `MiddlewareError` or a justified equivalent. Address ValidationError, AuthenticationError, AuthorizationError, ModelNotFoundError, UnsupportedCapabilityError, ProviderRateLimitError, ProviderUnavailableError, ProviderTimeoutError, ProviderProtocolError, and InternalError; explain naming differences.
2. Create **provider-specific failure → internal error → OpenAI-compatible HTTP error** mappings covering 400, 401, 403, 404, 429, 500, 502, 503, and 504. Specify error envelope message/type/code/param semantics where applicable, request correlation, safe details, and retry hints. Distinguish local authentication from upstream authentication failures.
3. Reconcile Task 01 error-consumption evidence and Task 03 response building. Explicitly specify actual serialized JSON, avoiding accidental framework detail-wrapper differences; never leak stack traces or raw credentials.
4. Define lifecycle: request_id → start → validation → normalization → routing → provider invocation → result normalization → response build → completion/failure. Identify owner, allowed transitions, terminal states, and cancellation transitions.
5. Define timeout/deadline propagation, cancellation ownership, cleanup and provider process termination obligations, and terminal reporting. SDK/CLI mechanisms remain later implementation details.
6. Assign one retry policy owner and document interactions with upstream SDK retries. Classify safe versus unsafe retry candidates, including rate limits, unavailable upstream, auth/validation errors, cancellation, and uncertain completion/duplicate inference risk. Define bounded attempts/deadline constraints; exact numeric retry limits remain deferred, never unbounded by default.
7. Define outcomes when streams fail before completion, terminate unexpectedly, or yield malformed data. Apply the OpenAI reference's completion requirement and CLI final-result/exit-status requirement as provider adapter responsibilities, not canonical event types.
8. Specify structured logging fields and redaction boundaries for request IDs, phase, provider identity, timing, and safe error codes. Prohibit OAuth/API credentials, raw sensitive payloads, and unfiltered stderr in logs or public errors.
9. Add written failure scenarios with expected internal category, HTTP outcome, retry eligibility, cleanup obligation, and test seam. Include success, timeout, cancellation, malformed stream, missing terminal event, and nonzero CLI exit. Do not implement retries, collectors, or tests.

## Expected Deliverables

Future output: `docs/architecture/error-and-request-lifecycle.md`.

The document must contain the exception tree, status/envelope mapping matrix, lifecycle diagram, ownership table, retry safety matrix, redaction contract, and failure-case inventory.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- Every listed failure category/status has a documented purpose and safe output.
- Cancellation/deadline propagation and process cleanup have unambiguous owners.
- Unexpected stream termination cannot produce a false successful result.
- Retry bounds/ownership and unsafe replay cases are explicit without numeric tuning.
- Implementation & Testing confirms each failure path has an independent validation seam.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Walk one success and each failure class through the lifecycle and public envelope. Check that no double retry owner exists and no terminal path leaks resources/secrets. Review future test scenarios without implementing tests.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Unclear retry ownership, success without terminal completion, unsanitized errors/logs, missing cancellation cleanup, or mismatch with accepted consumer error behavior blocks acceptance.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Provide taxonomy, lifecycle and scenario matrices to Tasks 07 and 08. Coordinate terminology with Task 05 without waiting for it as a prerequisite; escalate cross-document conflicts to the Coordinator.

Attach the accepted artifact revision, requirement/decision IDs, review findings,
validation evidence, and deferred items. A task's completion does not complete the
Wave; Task 08 and Coordinator final acceptance are required.

## Completion Report Format

```text
Task:
Status:
Artifact path and revision/hash:
Sources and versions inspected:
Prerequisites accepted:
Decisions frozen in this task:
Decisions intentionally deferred (owner / reason / trigger):
Acceptance criteria and evidence:
Validation commands / document walkthroughs / results:
Reviewer decision:
Findings (Blocking / Important / Optional) and disposition:
Known limitations:
Files changed / scope verification:
Next task(s) and handoff constraints:
Coordinator acceptance:
```

## Execution Record

```text
Task: Wave 01 — Task 06 — Error, Reliability and Request Lifecycle
Status: COMPLETED
Artifact path and revision/hash: docs/architecture/error-and-request-lifecycle.md; SHA-256 17399ff4630bef044c9721a90bf3a184df7f1ed4dcec2b73ea22263ca1e8d149
Sources and versions inspected: accepted Task 02 SHA-256 ac71a6ae853dec879f47fe6b403bb04ecf7648eb959b7b8e4c1cda56826edf4b; accepted Task 03 SHA-256 30e6d0e6788a4b5fda69225770f332a4119aea6dc3b6d35ac3373e2b981219fb; accepted Task 04 SHA-256 fbb9ff3084cdec737f8b76be7b3579291cc4be34d474c8fd07e73b492509bdff; Task 05 terminology-only coordination SHA-256 6224a1c901060ceb7a5b9ac4d276edf0e79317bf9dbd96a8f90b3eda976a286f; repository baseline e460c3f1a863f877db89b29133c594a1685da836; mandatory local instructions/references; no live provider, SDK, CLI, auth, network, or process probe
Prerequisites accepted: Tasks 02, 03, and 04 COMPLETED; Task 05 consumed only for shared terminology and remains a sibling, not a dependency
Decisions frozen in this task: ERD-001 through ERD-020; MiddlewareError taxonomy and internal auth variants; exact four-field public error envelope; deterministic HTTP mapping; lifecycle states and atomic response commit; terminal stream/CLI success; one absolute nonincreasing deadline; timeout/cancellation arbitration; unconditional no-response pre-commit cancellation; cleanup precedence and bounded safety reap; lifecycle as sole retry owner; finite effective attempt budget; positive no-acceptance retry evidence; no uncertain replay or fallback; safe correlation, logging, and redaction; Task 05 terminology crosswalk
Decisions intentionally deferred (owner / reason / trigger): ERD-D01 through ERD-D07 defer numeric timing/retry values, concrete exception/handler code, provider SDK retry settings, process-group APIs, logging stack, request-ID generator, and any future public stream/tool protocol to their documented implementation or future-contract owners and evidence triggers
Acceptance criteria and evidence: 36 sections; ERI-001 through ERI-027; ERD-001 through ERD-020; ERD-D01 through ERD-D07; ERV-001 through ERV-046; exact serialized JSON; 400/401/403/404/429/500/502/503/504 mappings; deterministic cancellation no-response rule; one retry owner; bounded retry and no uncertain replay; terminal-only InferenceResult; stream/CLI failure and cleanup scenarios; allowlist-first observability and mandatory redaction; CDI/CCI/PRI/PRD traceability
Validation commands / document walkthroughs / results: prerequisite status/hash checks passed; section and registry continuity checks passed; error-envelope, cancellation, retry, cleanup, and review-state checks passed; local links resolved; git diff --check passed; no-index artifact check produced no whitespace diagnostics; exact three-file scope and empty staged set verified; no executable tests were run or claimed because this task is documentation-only
Reviewer decision: Implementation & Testing APPROVE after required deterministic mapping/lifecycle/retry/cleanup corrections; fresh independent Architecture & Security APPROVE after unconditional cancellation-envelope correction; Google / Antigravity not used because accepted local references were sufficient and no concrete agy schema/flag/termination API was frozen
Findings (Blocking / Important / Optional) and disposition: none remaining / Implementation & Testing findings on connection-loss mapping, transition parity, finite retry accounting, cleanup, configuration split, and validation inventory were resolved; independent Architecture & Security cancellation-delivery ambiguity and residual ERV-013/014 qualifiers were resolved and re-reviewed / none remaining
Known limitations: documentation-only design; no executable implementation/tests; upstream SDK/CLI versions unpinned; exact agy event schema and platform termination API remain implementation-time evidence; numeric deadlines, attempts, backoff, and retry-after caps remain deferred but must be finite and deadline-bounded
Files changed / scope verification: Wave index Task 06 status, Task 06 execution record, and the sole Task 06 architecture artifact only; no production code, tests, Task 07 artifact, exception/retry/timeout/process/auth implementation, SDK/CLI execution, instructions, agents, dependencies, secrets, raw provider diagnostics, or staged files
Next task(s) and handoff constraints: Task 07 remains PLANNED and must not start without explicit user instruction; it owns auth mechanisms/storage while preserving this error/redaction/cancellation contract; Task 08 remains PLANNED
Coordinator acceptance: explicitly approved by the user and completed for commit on 2026-10-13; no push performed
```
