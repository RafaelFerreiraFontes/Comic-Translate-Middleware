# Wave 01 — Task 04 — Provider Interface and Routing

## Status

PLANNED

## Objective

Freeze the common provider abstraction and deterministic namespaced routing boundaries, including coexistence of Google SDK and CLI paths.

## Why This Task Exists

Provider selection must not leak into compatibility or endpoint logic. Existing instructions already require CanonicalRequest/InferenceResult, namespaces, and separate Google transports.

## Dependencies

- [Task 02 — Canonical Domain Contracts](02-canonical-domain-contracts.task.md): accepted deliverable and completion evidence.
- [Task 03 — Compatibility Layer Contract](03-compatibility-layer-contract.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [openai-codex-cli.md](../../instructions/openai-codex-cli.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Coordinator agent](../../agents/coordinator.agent.md).
- [Google / Antigravity agent](../../agents/google-antigravity.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Architecture & Security
- Reviewer: Coordinator
- Optional specialist: Google / Antigravity — only if Google transport ownership needs capability clarification.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Interface semantics, dependency diagram, routing decision table, and registration/configuration ownership; no provider or router implementation.

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

- What are the preconditions, results, and failure responsibilities of discovery, inference, and capabilities?
- How are ProviderId and ModelId distinguished from transport identity and upstream slug?
- How does configuration resolve a google: model when both SDK and CLI are configured?
- Is a ProviderRegistry needed, and if so, what is its minimal responsibility?
- Where does capability validation happen without a reverse dependency on compatibility?

## Decision required in this task

Provider/interface responsibilities, identity grammar, deterministic dispatch/configuration ownership, minimal registry need, and capability-check boundary.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Concrete provider classes, SDK client tuning, plugin loading implementation, subprocess flags, catalog cache mechanics, and provider performance optimization.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Define `BaseProvider`, `ProviderRouter`, `ProviderRegistry` (or justify a simpler equivalent), `ProviderId`, and `ModelId`. Record ownership and allowed dependencies for api, compatibility, providers, auth, models, and config.
2. Preserve the conceptual contract: async `list_models() -> list[ModelInfo]`, async `infer(request: CanonicalRequest) -> InferenceResult`, and `capabilities -> ProviderCapabilities`. Define async/resource expectations, success/failure semantics, and how dependencies are supplied.
3. Assign provider responsibility: canonical request → provider protocol → provider execution → canonical result. HTTP transport only validates/resolves/calls/returns; it performs no SDK calls or agy subprocess execution.
4. Assign router responsibility: parse namespaced model → select configured provider transport → extract slug → validate capability → dispatch through abstraction. Specify malformed IDs, missing namespaces, unknown namespaces/models, unavailable configuration, and unsupported capabilities.
5. Show explicit routes `openai:gpt-... → OpenAI provider` and `google:gemini-... → configured Google provider path`. Retain original public identity for response/catalog correlation after extracting the upstream slug.
6. Define deterministic configuration ownership when AntigravitySdkProvider and AntigravityCliProvider coexist. Document collision/ambiguity handling, selection precedence, and whether any fallback is permitted; never infer transport from a subscription name or silently switch authentication/billing paths.
7. If a registry is adopted, limit it to registration/lookup/discovery of provider abstractions, not business logic. Distinguish registration from remote model discovery.
8. Document forbidden dependencies: compatibility → concrete provider imports; API endpoint → SDK calls; API endpoint → agy subprocess calls. Codex CLI is a development tool, not an inference transport.
9. Provide routing examples for both Google configurations, duplicate slug, absent provider, unsupported images, and a hypothetical future provider. Reserve detailed catalog semantics for Task 05 and failure taxonomy for Task 06.

## Expected Deliverables

Future output: `docs/architecture/provider-contract-and-routing.md`.

The document must include conceptual signatures, component responsibility/dependency tables, ID grammar, routing decision table, Google transport configuration policy, and extension/mockability examples.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- The common contract consumes CanonicalRequest and returns InferenceResult.
- Namespaced IDs resolve deterministically; bare/ambiguous public IDs have an explicit error outcome.
- Both Google paths coexist without credential sharing or compatibility changes.
- A future provider can be registered behind the abstraction without editing compatibility behavior.
- No concrete provider, SDK, or subprocess is introduced into HTTP/compatibility boundaries.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Walk through all routing examples, including collisions and a hypothetical new provider. Confirm the schema from Task 02 can preserve model identity and Task 03 can remain unchanged.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Ambiguous Google transport ownership, provider-specific coupling, bare model routing, or a proposal for Codex inference blocks the contract. Do not fill configuration gaps with unsupported transport assumptions.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Supply interface/routing revision and scenario table to Tasks 05 and 06. Both can proceed after this task is accepted; Task 07 requires their accepted outputs.

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
