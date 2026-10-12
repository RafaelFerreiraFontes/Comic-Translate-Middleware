# Wave 01 — Task 03 — Compatibility Layer Contract

## Status

PLANNED

## Objective

Specify deterministic Chat Completions-to-canonical normalization and canonical-to-Chat Completions response building without concrete provider dependencies.

## Why This Task Exists

The public contract must remain stable even when an upstream requires parameter filtering or internal streaming. The references require provider-agnostic compatibility and thin HTTP transport.

## Dependencies

- [Task 01 — Comic Translate Protocol Baseline](01-comic-translate-protocol-baseline.task.md): accepted deliverable and completion evidence.
- [Task 02 — Canonical Domain Contracts](02-canonical-domain-contracts.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [fastapi-reference-python.md](../../instructions/fastapi-reference-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Implementation & Testing agent](../../agents/implementation-testing.agent.md).
- [Coordinator agent](../../agents/coordinator.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Architecture & Security
- Reviewer: Implementation & Testing
- Optional specialist: None.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Request/response mapping tables, parameter policy, streaming boundary, and written acceptance examples for future tests.

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

- How are supported roles and mixed content normalized while preserving meaning and ordering?
- Which parameters are canonical, provider-mappable, ignored for a specific provider, or rejected?
- Which component enforces provider-specific restrictions without contaminating public normalization?
- What response properties are required by consumer behavior versus compatibility extras?
- How does internal streaming produce a complete public response?

## Decision required in this task

Normalization and response-building semantics, public parameter classification, policy ownership, and public/internal streaming boundary.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Provider SDK calls, exact event collectors, FastAPI endpoints, serialization implementation, and executable contract tests.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Specify `ChatCompletionRequest → normalize → CanonicalRequest` and `InferenceResult → build → ChatCompletionResponse`, with input/output preconditions, postconditions, and errors.
2. Map system, developer, user, and assistant roles only as supported by Task 01. State instruction ordering/precedence, content preservation, and unsupported-role behavior.
3. Walk through `{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,..."}}` into a canonical image part, including media type/data validation and ordering. No OpenAI or Google SDK object may enter the canonical representation.
4. Create a parameter matrix with public field, evidence, canonical mapping, provider-mappable option, provider-policy owner, ignored-for-specific-provider rule, rejected case, and observable outcome. Preserve omission/default semantics.
5. Use `temperature` and `max_completion_tokens` as reference-backed SIWC filtering cases, validated against Task 01. Public acceptance must not imply blind upstream forwarding. Specify filtering at the upstream adapter/provider policy boundary; generic compatibility must not branch on concrete provider classes.
6. Clarify placement of protocol conversion helpers mentioned in the OpenAI reference versus provider execution. Helpers must stay free of concrete provider imports and upstream I/O; report any unresolved document-placement conflict to the Coordinator.
7. Specify `choices[0].message.content` and `id`, `object`, `created`, `model`, `choices`, `finish_reason`, and `usage`. Classify required/optional using Task 01, including client deserialization needs, and map each to a source/default rule.
8. Separate upstream/internal streaming from public response streaming. Define collection to a complete InferenceResult when the consumer expects a full response. SIWC retains `store=False`, `stream=True`; completion/failure events remain upstream details. Do not promise public streaming without evidence.
9. Add written input → canonical → result → public-output examples and negative cases, with acceptance assertions for future contract tests; leave executable fixtures and tests to later waves.

## Expected Deliverables

Future output: `docs/architecture/compatibility-contract.md`.

The document must contain both mapping contracts, role/content tables, parameter classification matrix, response property table, streaming boundary diagram, and written compatibility cases.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- Every Task 01 mandatory field and Task 02 invariant is accounted for.
- Images and instructions survive normalization; full responses expose valid choices[0].message.content.
- Unsupported SIWC parameters can be filtered without provider-specific compatibility imports.
- Public streaming promises match consumer evidence and differ explicitly from upstream streaming.
- Implementation & Testing confirms the mappings are independently testable without implementing them.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Trace text, mixed-image, omitted-option, unsupported-option, and upstream-stream failure examples through the mappings. Review schema/example consistency and dependency direction; future test cases must specify expected outcomes.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Concrete provider imports, upstream SDK calls in compatibility/API design, silent image loss, unknown required response fields, or a streaming assumption conflicting with Task 01 blocks acceptance.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Send accepted mapping tables and test-case descriptions to Tasks 04 and 06 through the Coordinator. Task 08 must verify these mappings against the final provider and error contracts.

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
