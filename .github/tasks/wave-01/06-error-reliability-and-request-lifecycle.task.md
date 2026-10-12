# Wave 01 — Task 06 — Error, Reliability and Request Lifecycle

## Status

PLANNED

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
