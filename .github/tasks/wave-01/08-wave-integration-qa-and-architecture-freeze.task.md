# Wave 01 — Task 08 — Wave Integration QA and Architecture Freeze

## Status

PLANNED

## Objective

Independently verify all Wave 01 contracts, resolve integration findings, and publish the architecture freeze accepted by the Coordinator.

## Why This Task Exists

Individually accepted documents can still disagree at boundaries. Subsequent waves need a single traceable release of compatible decisions, not seven unrelated design reports.

## Dependencies

- [Task 01 — Comic Translate Protocol Baseline](01-comic-translate-protocol-baseline.task.md): accepted deliverable and completion evidence.
- [Task 02 — Canonical Domain Contracts](02-canonical-domain-contracts.task.md): accepted deliverable and completion evidence.
- [Task 03 — Compatibility Layer Contract](03-compatibility-layer-contract.task.md): accepted deliverable and completion evidence.
- [Task 04 — Provider Interface and Routing](04-provider-interface-and-routing.task.md): accepted deliverable and completion evidence.
- [Task 05 — Model Catalog and Capabilities](05-model-catalog-and-capabilities.task.md): accepted deliverable and completion evidence.
- [Task 06 — Error, Reliability and Request Lifecycle](06-error-reliability-and-request-lifecycle.task.md): accepted deliverable and completion evidence.
- [Task 07 — Security and Authentication Boundaries](07-security-and-auth-boundaries.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [clean-code-python.md](../../instructions/clean-code-python.md).
- [fastapi-reference-python.md](../../instructions/fastapi-reference-python.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [openai-codex-cli.md](../../instructions/openai-codex-cli.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
- [Coordinator agent](../../agents/coordinator.agent.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Implementation & Testing agent](../../agents/implementation-testing.agent.md).
- [Google / Antigravity agent](../../agents/google-antigravity.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Coordinator
- Reviewer: Architecture & Security
- Optional specialist: Implementation & Testing — testability gate; Google / Antigravity — Google capability evidence if review gaps remain.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Integrated document QA, traceability, independent review, freeze summary, and change-control rules; no runtime implementation.

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

- Do all documents describe the same consumer-to-provider-to-consumer flow?
- Are schema, routing, capabilities, errors, and auth ownership mutually consistent?
- Can each boundary be independently mocked without changing the public protocol?
- Which decisions are frozen, which are deferred, and which unknowns block implementation?
- How are later breaking changes proposed and affected contracts revalidated?

## Decision required in this task

Integrated contract release, gate disposition, freeze/defer classification, breaking-change rules, and readiness for later implementation waves.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Provider-specific performance tuning, exact HTTP/retry settings, OAuth UX, CLI installation UX, packaging, desktop integration, and implementation/test code; no blocking boundary decision may be deferred.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Collect accepted versions of all seven deliverables and completion reports. Record paths, revision/hash identifiers, evidence versions, authors, review decisions, and unresolved findings.
2. Build a traceability table: Task 01 requirement → canonical field → compatibility rule → provider/router obligation → capability/error/security rule → proposed future test. Include text, images, model selection, parameter filtering, and failure paths.
3. Have Architecture & Security independently perform every gate below. The reviewer must not self-approve authored decisions: use a fresh review assignment/context not responsible for the reviewed artifacts, disclose authorship, and have the Coordinator arrange independent coverage. Reviewer output uses APPROVE / APPROVE WITH REQUIRED CHANGES / REJECT and Blocking / Important / Optional findings.
4. Run paper end-to-end scenarios for text and image requests through OpenAI and both Google paths, with unsupported CLI images handled honestly; include timeout, cancellation, discovery outage, and authentication unavailable. No provider is executed.
5. Record each gate as PASS / FAIL / BLOCKED with linked evidence and owner. Route required corrections back to the owning task via the Coordinator; re-review impacted downstream decisions. Do not patch away conflicts in a summary.
6. Publish frozen decisions, deferred decisions, known unknowns, provider-specific questions deferred to later waves, and breaking-change rules. Assign every deferral an owner, target later wave/domain, trigger, and reason it cannot invalidate the frozen boundary.
7. Define breaking-change procedure: proposal and consumer impact → architecture/security review → affected contract/document updates → later test impact → Coordinator acceptance. Existing instructions remain authoritative; changes to them require separately authorized scope.
8. Record independent review acceptance and Coordinator final acceptance only after all gates pass and blocking/required findings close. A nonblocking provider detail may remain deferred only with a safe boundary and explicit rationale.

## Expected Deliverables

Future output: `docs/architecture/wave-01-architecture-freeze.md`.

The freeze document must contain the artifact/version manifest, traceability table, scenario results, per-gate evidence, independent review findings/disposition, frozen/deferred/unknown decision registers, breaking-change rules, and Coordinator acceptance.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- Every gate below passes with evidence and no unresolved blocking architecture/security question.
- All seven input documents and completion reports are accepted and version-identifiable.
- Independent Architecture & Security review passes without self-approval.
- Deferred details have owners and cannot silently alter frozen contracts.
- No provider implementation or Comic Translate modification was introduced.
- Coordinator explicitly accepts Wave QA; only then may the Wave be marked complete.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Apply the explicit gate table below, verify all cross-document references and scenarios, and inspect the entire Wave diff/scope. Testability is a design review of future mocks/assertions, not creation or execution of tests.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Integration Gates

| Gate | Required evidence and pass condition |
| --- | --- |
| External Protocol Gate | Task 01 source/version evidence supports every mandatory behavior; URL, headers, request and response consumption align. Zero Comic Translate source modifications. |
| Canonical Contract Gate | CanonicalRequest and InferenceResult are provider-independent, preserve images/instructions/identity, and suffice for planned transports without credentials or upstream event types. |
| Compatibility Gate | Both Chat Completions mappings are complete; public behavior and provider-specific filtering are separate; no concrete provider coupling. |
| Provider Gate | A hypothetical additional provider fits the common abstraction and routing/configuration policy without changing compatibility logic. |
| Model Gate | Namespaced IDs, catalog merging, discovery, model/provider/runtime capabilities, unknown states, and Google transport ownership agree. |
| Error Gate | Representative upstream failures map to internal categories and safe OpenAI-compatible envelopes; deadlines, cancellation, retries, and terminal completion have single owners. |
| Security Gate | No private credential access, secret logging, undocumented authentication APIs, or public binding by default; 127.0.0.1 and auth ownership are preserved. |
| Google Architecture Gate | AntigravitySdkProvider and AntigravityCliProvider remain separate authentication/capability paths; unknown headless images are not advertised as supported. |
| OpenAI Architecture Gate | Chat Completions public API → CanonicalRequest → Responses internally where required remains viable; SIWC store=False / stream=True and filtering hold; Codex CLI is absent from inference. |
| Testability Gate | Each API/compatibility/router/provider/auth boundary has a documented mock seam, scenario input, expected output/error, and later test owner. No executable tests are required in this Wave. |
| Architecture Freeze Gate | Frozen/deferred/unknown registers and change-control rules are complete, independent review passes, required findings close, and Coordinator acceptance is recorded. |

## Stop Conditions

Any missing prerequisite acceptance, failing/blocked gate, author self-approval, unsupported upstream capability claim, unresolved contract conflict, or scope leak prevents architecture freeze and Wave completion.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Return the accepted freeze manifest and gate report to the Coordinator for later-wave planning. Carry forward deferred owners and constraints; completing this planning file alone never means Wave 01 has executed.

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
