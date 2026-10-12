# Wave 01 — Task 02 — Canonical Domain Contracts

## Status

PLANNED

## Objective

Specify provider-independent domain schemas and invariants with CanonicalRequest as the internal request source of truth and InferenceResult as the provider output.

## Why This Task Exists

One explicit domain vocabulary prevents OpenAI Responses payloads, Google SDK objects, and agy events from becoming implicit core contracts. Existing references already require these domain boundaries.

## Dependencies

- [Task 01 — Comic Translate Protocol Baseline](01-comic-translate-protocol-baseline.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
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
- Optional specialist: Implementation & Testing — for schema feasibility and future fixture strategy only.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Architecture schema tables, type relationships, invariants, and embedded illustrative instances; no production Pydantic models.

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

- How are public fields represented without copying upstream-specific fields into the core?
- How do ordered roles, instruction precedence, mixed text/images, and output hints retain meaning?
- How are public model identity and routed upstream slug distinguished without ambiguity?
- What are the nullability, default, validation, and ownership rules for results, usage, finish reasons, and request context?

## Decision required in this task

Canonical field semantics, invariants, conceptual types, identity and result shape, and explicit extension points.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Pydantic classes/validators, serialization optimizations, detailed catalog precedence in Task 05, exact HTTP timeouts, and provider-specific payload construction.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Map every accepted Task 01 requirement to a domain field or an explicit transport-only concern. Separate columns for public API fields, canonical fields, and provider-specific fields.
2. Define `CanonicalRequest`, `CanonicalMessage`, `CanonicalContentPart`, `CanonicalTextPart`, `CanonicalImagePart`, `InferenceResult`, `UsageInfo`, `FinishReason`, `ProviderCapabilities`, and `ModelInfo`. Alternative names require a semantic crosswalk and justification.
3. For each schema give field name, conceptual type, required/optional status, default/omission/null semantics, validation invariant, owner, and source requirement. Address model identity, system/developer instructions, ordered messages, text/image parts, request identifier/context, timeout hints, output-format hints, and provider-neutral generation options.
4. Define image representation and ownership without SDK objects: media type, data/reference alternatives supported by Task 01, validation responsibilities, and preservation of content ordering. Specify invalid/unsupported representations without silently dropping images.
5. Specify result text/content, model identity, completion state, finish-reason normalization, usage units and absent/unknown usage, and safe request correlation. Do not manufacture usage or successful completion from missing data.
6. Record invariants: no provider credentials/tokens/sessions in CanonicalRequest; images remain first-class; identity remains unambiguous after routing; canonical types contain neither Responses event types nor agy JSONL event types.
7. Define the minimum structural shape of ProviderCapabilities and ModelInfo needed by Tasks 03–04. Task 05 owns detailed discovery/availability semantics, not a competing schema. Mark planned extension points explicitly.
8. Add embedded valid/invalid examples and a future validation-case inventory (text, mixed images, instructions, missing model, invalid options, missing usage). Review feasibility without creating executable models or tests.

## Expected Deliverables

Future output: `docs/architecture/canonical-domain-contracts.md`.

The document must contain a schema dictionary for all ten concepts, relationships, public/canonical/provider field crosswalk, invariant list, valid/invalid examples, and later contract-test cases.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- All Task 01 mandatory inputs and consumed outputs have a canonical or explicit transport-only mapping.
- CanonicalRequest and InferenceResult have complete structural definitions and no credentials or SDK/event coupling.
- Image/instruction ordering, model identity, optional usage, and completion semantics are explicit.
- Capability/model type shape is sufficient for routing, with Task 05's semantic ownership clearly stated.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Perform a field-by-field trace to Task 01 and paper mappings to the planned OpenAI, Google SDK, and Google CLI paths. Confirm unsupported provider capabilities can be expressed as errors without changing core types.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

A mandatory consumer field cannot be represented, credentials or provider event types appear in domain models, or an upstream transport forces a core redesign. Missing Task 01 acceptance blocks execution.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Deliver the accepted schema revision to Tasks 03, 04, and 06. Task 05 must reference these definitions; any necessary structural change returns through the Coordinator for impact review rather than silently revising them.

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
