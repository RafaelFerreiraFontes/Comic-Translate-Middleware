# Wave 01 — Task 01 — Comic Translate Protocol Baseline

## Status

COMPLETED

## Objective

Freeze the external consumer contract from identifiable Comic Translate source or verifiable repository research, so later contracts adapt to the consumer without source modifications.

## Why This Task Exists

The references already establish Chat Completions compatibility, image preservation, and a complete response. They do not replace evidence of the selected Comic Translate version's call sites, URL construction, model selection, or error consumption.

## Dependencies

None. This is the external-contract entry gate for Wave 01.

Do not start while a prerequisite has unresolved blocking findings. Read existing consumer research if found; none was present when this task set was created.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [fastapi-reference-python.md](../../instructions/fastapi-reference-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Coordinator agent](../../agents/coordinator.agent.md).
- [Implementation & Testing agent](../../agents/implementation-testing.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Architecture & Security
- Reviewer: Coordinator
- Optional specialist: Implementation & Testing — only for fixture and contract-test strategy review.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Read-only consumer/source research, evidence inventory, public protocol matrices, and sanitized illustrative payloads embedded in the architecture document.

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

- Which endpoints are actually invoked: `POST /v1/chat/completions`, `GET /v1/models`, or others?
- Who supplies/appends `/v1`, how are trailing slashes handled, and what headers/local authentication does the client expect?
- Which fields, roles, image representations, and response properties are actually sent/read?
- Does the client accept manual namespaced model IDs, discover/cache models, or validate them?
- What errors are surfaced, parsed, retried, or suppressed by the consumer and its client library?

## Decision required in this task

Supported consumer revision, mandatory/optional protocol behavior, base URL composition, accepted request shapes, response consumption, and evidence-backed model/error assumptions.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Provider payload mappings, canonical type details, upstream authentication implementation, and executable compatibility fixtures belong to subsequent tasks/waves.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Recheck the repository for research and locate the relevant Comic Translate source. Record source URL/local path, release or commit, file/symbol/line references, client-library version, and inspection date. If the supported version is unspecified, obtain a Coordinator decision before freezing; do not silently assume latest.
2. Trace settings/UI configuration through client construction, request creation, response reading, and error handling. Validate existing project claims rather than treating example payloads as consumer evidence. Cite client-library source when behavior is inherited.
3. Build an endpoint/base-URL table covering method/path, invocation evidence, prefix composition, trailing slash behavior, headers, and API-key expectations. Separate required consumer behavior from optional middleware surface; do not invent endpoints.
4. Build a request field table: `model`, `messages`, `temperature`, `max_completion_tokens`, `stream`, `response_format` only where evidenced. Include types, omission/default behavior, conditional use, and source; mark absent or unknown fields explicitly.
5. Document system/developer, user, and assistant roles where used; string text versus content arrays; `image_url` and base64 `data:image/...` URLs. Include text-only and image-enabled examples with synthetic content, never captured secrets.
6. Trace `choices[0].message.content` and distinguish properties consumed directly, required by client deserialization, and optional OpenAI-compatible extras. Record complete versus streamed response expectations.
7. Trace manual model entry, namespaced ID acceptance, `/v1/models` use/non-use, caching, and validation. For 400, 401, 404, 429, 500, 502, 503, and 504, record observed/code-derived behavior or explicitly state that no evidence was found.
8. Produce the compatibility matrix: **Comic Translate behavior → middleware responsibility → evidence/source → mandatory / optional**. Add confidence/unknowns separately; lack of evidence does not make a requirement optional. Give each requirement a stable ID for later traceability.
9. Submit the baseline and unresolved questions to the reviewer. Freeze the external contract only after required behavior is evidenced and blocking findings are resolved; Tasks 02–06 cannot finalize first.

## Expected Deliverables

Future output: `docs/architecture/comic-translate-protocol-contract.md`.

The document must contain the source/version inventory, endpoint/base-URL table, request/message/response schemas, model/error behavior tables, compatibility matrix, embedded sanitized examples, and explicit evidence gaps.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- Every mandatory consumer requirement has a reproducible source reference and middleware responsibility.
- URL construction and response parsing can be followed from configuration to call site.
- Text and image-enabled flows and manual/discovered model behavior are covered without invented fields.
- Required behavior is feasible with zero Comic Translate source changes.
- The Coordinator records external-contract acceptance; no blocking protocol uncertainty remains.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Independently follow representative citations and perform a paper walkthrough of one text request, one image request, one model-selection path, and one failure path. Review proposed fixture cases with Implementation & Testing only if needed; do not create fixtures/tests.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Missing/unidentifiable consumer source, unresolved supported-version choice, conflicting consumer evidence and project rules, or a requirement that appears to need consumer modifications blocks baseline freeze. Report unavailable evidence instead of fabricating behavior.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Send the accepted requirement IDs, evidence inventory, and protocol document to Task 02 and make them mandatory inputs to Tasks 03–06. Return conflicting assumptions to the Coordinator.

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
Task: Wave 01 — Task 01 — Comic Translate Protocol Baseline
Status: COMPLETED
Artifact path and revision/hash: docs/architecture/comic-translate-protocol-contract.md; SHA-256 7f38d5fd91962d50209bc56e68e29c1c6ea44427003c84cada78e3bfd08b7523
Sources and versions inspected: ogkalu2/comic-translate @ 8977b91a4f7a40c3917c5a268e9e7d78e1d818da; main; v2.8.9; requests>=2.31.0
Prerequisites accepted: none; Task 01 has no dependencies
Decisions frozen in this task: Custom base-URL composition; direct POST Chat Completions request; observed fields/messages/image shape; complete response consumption; block_N JSON semantics; manual model selection; no consumer model discovery; public Bearer behavior; generic consumer errors
Decisions intentionally deferred (owner / reason / trigger): canonical models (Task 02 / outside consumer contract / Task 02 start); compatibility response and malformed-output policy (Task 03 and Task 06 / implementation architecture / dependent task start); provider mappings and auth internals (later provider/security tasks / private boundary / applicable task start)
Acceptance criteria and evidence: compatibility matrix CT-REQ-001 through CT-REQ-022 and evidence index CT-001 through CT-010 in the artifact; zero-modification result YES
Validation commands / document walkthroughs / results: pinned-source configuration, text request, image request, response parser, manual model, and failure paths independently followed; mandatory-section check passed; final repository checks recorded at approval gate
Reviewer decision: Architecture & Security APPROVE
Findings (Blocking / Important / Optional) and disposition: none / parser and malformed-JSON corrections resolved and independently verified / none
Known limitations: source-derived rather than live runtime evidence; no dedicated upstream protocol tests found; exact installed Requests version is not pinned
Files changed / scope verification: Task 01 task record, Wave 01 Task 01 status row, and the single Task 01 architecture artifact only; no implementation or tests
Next task(s) and handoff constraints: Task 02 remains PLANNED and must not start without explicit user instruction
Coordinator acceptance: approved by the user on 2026-10-12; Task 01 completed for commit
```
