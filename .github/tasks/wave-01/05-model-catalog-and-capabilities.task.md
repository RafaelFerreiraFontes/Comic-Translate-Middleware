# Wave 01 — Task 05 — Model Catalog and Capabilities

## Status

COMPLETED

## Objective

Define discovery, catalog merging, model availability, and effective capability semantics without treating all models or Google transports as identical.

## Why This Task Exists

Model discovery is already the project source of truth. Capability samples in the references are not proof that every model supports identical features or that agy accepts headless images.

## Dependencies

- [Task 04 — Provider Interface and Routing](04-provider-interface-and-routing.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
- [Architecture & Security agent](../../agents/architecture-security.agent.md).
- [Google / Antigravity agent](../../agents/google-antigravity.agent.md).
- [Coordinator agent](../../agents/coordinator.agent.md).
- [Implementation & Testing agent](../../agents/implementation-testing.agent.md).

Inspect the current repository, branch, working tree, existing architecture/ADRs,
and prerequisite revisions before editing. Reuse and validate existing decisions;
do not overwrite them. Reference examples describe project intent, not independent
proof of current upstream capabilities. Record source/version evidence for claims.

## Assigned Agents

- Primary: Architecture & Security
- Reviewer: Google / Antigravity
- Optional specialist: Implementation & Testing — only for catalog scenario/testability review.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

ProviderCapabilities/ModelInfo semantic refinement, ModelCatalog contract, availability/merge policy, and evidence matrix.

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

- Which capabilities belong to provider, model, or runtime/discovered state?
- How are unknown, unsupported, configured, discovered, and temporarily unavailable distinguished?
- How does catalog merging preserve namespace and transport ownership when model slugs collide?
- How do auth state and discovery failures affect listing and inference eligibility?
- How can a future provider extend the catalog without changing consumer compatibility?

## Decision required in this task

Capability ownership and precedence, unknown-state behavior, availability semantics, catalog merge rules, and discovery/invalidation responsibilities.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

Exact SDK discovery calls/CLI flags, cache TTL values, persistence mechanisms, subscription UX, and provider-specific optimization.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Reuse Task 02's ProviderCapabilities and ModelInfo shapes and Task 04's selection policy. Specify `ModelCatalog` responsibilities and the relationship between discovered/configured entries and public listing.
2. Define `supports_images`, `supports_system_prompt`, `supports_streaming`, `supports_structured_output`, and `auth_mode`; separate internal streaming support from public streaming. State provider-level defaults, per-model overrides, runtime evidence, and precedence.
3. Decide how unknown capability is represented without pretending it is true or equating it with demonstrated support. Include evidence source/version/freshness and distinguish capability from current availability.
4. Specify namespacing, duplicate names across providers, duplicate google: IDs across transports, merging/conflict policy, configured-only entries, discovered entries, authenticated/unauthenticated providers, and temporarily unavailable providers.
5. Define discovery ownership, refresh/invalidation triggers, stale-result semantics, partial discovery failure, and request-time capability revalidation. Leave numeric cache TTL and implementation tuning deferred.
6. Explicitly prohibit Plus/Pro/Ultra or other subscription names as canonical availability mechanisms. Use discovery, supported authentication state, and demonstrated capability rather than plan assumptions.
7. Require Google / Antigravity review of SDK versus CLI auth modes and capabilities, especially unconfirmed headless multimodal support. Cite official/versioned evidence where verified; mark unknown behavior and its conservative outcome.
8. Provide catalog and effective-capability tables for: same slug under openai:/google:, both Google transports, unknown CLI images, no auth, temporary outage, configured-but-undiscovered model, and mixed model capabilities.
9. Explain whether/how Task 01's public model discovery surface exposes catalog data; internal discovery remains necessary even if Comic Translate never calls /v1/models. Specify future test assertions without writing tests.

## Expected Deliverables

Future output: `docs/architecture/model-catalog-and-capabilities.md`.

The document must contain capability ownership/precedence tables, catalog schema semantics, merge and availability decision tables, provenance rules, and Google specialist findings.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- Provider, model, and runtime capability distinctions are explicit and coherent with routing.
- Unknown CLI image support is never advertised as confirmed.
- Duplicate IDs and unavailable/auth-limited providers have deterministic catalog outcomes.
- Subscription names are not availability keys; future providers need no compatibility rewrite.
- Google / Antigravity review is recorded and blocking capability assumptions are resolved.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Paper-evaluate every catalog scenario and trace effective capability to evidence and selected transport. Check compatibility with Task 02 shapes; route any proposed structural change back for explicit impact review.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Unsupported claims about CLI images/auth, conflicting discovery/configuration precedence, ambiguous duplicate IDs, or structural schema changes made without upstream review block acceptance.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Send the accepted catalog/capability tables and Google review to Task 07 and Task 08. Coordinate shared error terminology with Task 06 through the Coordinator; neither task gains a circular prerequisite.

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
Task: Wave 01 — Task 05 — Model Catalog and Capabilities
Status: COMPLETED
Artifact path and revision/hash: docs/architecture/model-catalog-and-capabilities.md; SHA-256 6224a1c901060ceb7a5b9ac4d276edf0e79317bf9dbd96a8f90b3eda976a286f
Sources and versions inspected: accepted Task 01 artifact SHA-256 7f38d5fd91962d50209bc56e68e29c1c6ea44427003c84cada78e3bfd08b7523; accepted Task 02 artifact SHA-256 ac71a6ae853dec879f47fe6b403bb04ecf7648eb959b7b8e4c1cda56826edf4b; accepted Task 03 artifact SHA-256 30e6d0e6788a4b5fda69225770f332a4119aea6dc3b6d35ac3373e2b981219fb; accepted Task 04 artifact SHA-256 fbb9ff3084cdec737f8b76be7b3579291cc4be34d474c8fd07e73b492509bdff; middleware baseline cf240aa76659259e551f6fd22b55c943c2963ed6; project OpenAI, Codex, and Google references at that baseline; upstream SDK/CLI versions unpinned and no live probes performed
Prerequisites accepted: Tasks 01 through 04 COMPLETED; all accepted artifact hashes verified before Task 05 execution
Decisions frozen in this task: MCD-001 through MCD-017; ModelCatalog ownership; adjacent catalog metadata without Task 02 schema change; namespaced identity; provider/model/runtime capability levels; tri-state capability; closed evidence admissibility/authority ranking; non-Boolean availability/discovery states; deterministic merge and conflict policy; transport/auth evidence isolation; freshness/staleness/invalidation; configured/manual inference without discovery; conservative public listing; request-time revalidation; no subscription-tier truth
Decisions intentionally deferred (owner / reason / trigger): MCD-D01 through MCD-D06 assign TTL/persistence/scheduling and evidence storage/concurrency to implementation, exact provider discovery calls and agy flags to Google specialist/implementation evidence, lifecycle/error mechanics to Task 06, auth lifecycle to Task 07, and future public catalog status/transport selectors to approved contract revision
Acceptance criteria and evidence: MCI-001 through MCI-028; MCV-001 through MCV-047; 18/18 CDI, 17/17 PRI, and 10/10 PRD traceability; deterministic capability/availability comparators and request-time transition matrix; conservative /v1/models projection; manual inference remains independent of discovery; separate Google SDK/CLI evidence stores; CLI images remain unknown; no capability, model availability, or plan-tier fabrication
Validation commands / document walkthroughs / results: 36-section and contiguous-registry checks passed; required scenario/negative-term scan passed; required Google / Antigravity review APPROVE; read-only Implementation & Testing testability review APPROVE after deterministic-state corrections; fresh independent Architecture & Security review APPROVE after closed evidence-authority correction; fresh Google evidence-ranking verification APPROVE; git diff --check passed; new-artifact no-index check produced no whitespace diagnostics; exact scope, links, empty staged set, and secret/code/later-task scans passed at approval gate
Reviewer decision: Google / Antigravity APPROVE; Implementation & Testing APPROVE; fresh independent Architecture & Security APPROVE
Findings (Blocking / Important / Optional) and disposition: none remaining / testability ambiguities in capability/availability ties, discovery outcomes, listing, auth readiness, and request-time transitions resolved and approved; architecture authority-ranking blocker resolved with Section 12 closed admissibility/rank and independently approved / future implementation should assert CLI image rejection before subprocess, separate evidence stores, and no image promotion from successful text inference
Known limitations: documentation-only paper validation; no executable tests or live provider/SDK/CLI/discovery/auth probes; upstream versions unpinned; headless agy image input remains unknown; exact discovery APIs/flags, TTL/storage/concurrency, Task 06 lifecycle/error policy, and Task 07 auth lifecycle remain deferred to their owners
Files changed / scope verification: Task 05 task record, Wave 01 Task 05 status row, and the single Task 05 architecture artifact only; no production code, tests, Task 06/07 artifacts, ModelCatalog implementation, provider calls, SDK/agy execution, auth implementation, instructions, agents, dependencies, secrets, or private credential/session access; nothing staged
Next task(s) and handoff constraints: Task 06 remains PLANNED and independent; Task 07 remains PLANNED; neither may start without explicit user instruction; Task 06 may map outcomes but must not change catalog states, evidence precedence, or fallback policy
Coordinator acceptance: explicitly approved by the user and completed for commit on 2026-10-13; no push performed
```
