# Wave 01 — Task 07 — Security and Authentication Boundaries

## Status

PLANNED

## Objective

Freeze credential ownership, authentication interfaces, and local-service security boundaries without implementing OAuth or touching credentials.

## Why This Task Exists

The OpenAI and Google paths have different authentication owners. Catalog and request processing need safe status interfaces, never credential scraping or transport-layer auth logic.

## Dependencies

- [Task 04 — Provider Interface and Routing](04-provider-interface-and-routing.task.md): accepted deliverable and completion evidence.
- [Task 05 — Model Catalog and Capabilities](05-model-catalog-and-capabilities.task.md): accepted deliverable and completion evidence.
- [Task 06 — Error, Reliability and Request Lifecycle](06-error-reliability-and-request-lifecycle.task.md): accepted deliverable and completion evidence.

Do not start while a prerequisite has unresolved blocking findings. Read the accepted architecture artifacts named by each prerequisite and its completion report. Task 01's accepted external contract is a transitive prerequisite for all later tasks.

## Required Reading

- [Wave index and execution rules](README.md).
- [Project instructions](../../instructions/project.md).
- [openai-sdk-python.md](../../instructions/openai-sdk-python.md).
- [openai-codex-cli.md](../../instructions/openai-codex-cli.md).
- [google-antigravity-sdk-python.md](../../instructions/google-antigravity-sdk-python.md).
- [fastapi-reference-python.md](../../instructions/fastapi-reference-python.md).
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
- Optional specialist: Google / Antigravity — for any unresolved documented CLI session-status boundary.
- Coordinator: Coordinator

Use the existing agent configuration; do not duplicate or change model assignments.
The Coordinator assigns only the necessary reviewers and owns final task acceptance.
Reviewer findings must be independent of the author where independent review is required.

## Scope

### In Scope

Credential ownership matrix, conceptual auth/status interfaces, trust-boundary diagrams, and threat/mitigation review.

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

- Which component owns OAuth orchestration, PKCE state, refresh, storage, auth status, and logout?
- What may providers consume from auth services, and what must never enter canonical requests or API responses?
- How is CLI session availability represented through documented high-level mechanisms?
- How are local administrative operations isolated from /v1 compatibility traffic?
- Which threat controls are fixed architecture and which details can safely wait?

## Decision required in this task

Credential lifecycle ownership, safe auth/status interfaces, local administrative boundary, and mandatory threat controls.

Preserve zero Comic Translate modifications, provider-agnostic compatibility,
CanonicalRequest → provider → InferenceResult, and namespaced public model IDs.
All project security rules apply, including 127.0.0.1 by default and no private
credential extraction, undocumented authentication APIs, or secret logging.

## Decision intentionally deferred

OAuth implementation and UI polish, exact SDK setup, CLI installation/login UX, secure-store implementation details, packaging, and desktop integration.

Record each deferral with owner, rationale, and later validation trigger. A blocking
contract question cannot be relabeled as an implementation detail to pass this task.

## Required Work

1. Document an ownership table for `auth/`, `providers/`, `api/`, and `config/`: OAuth orchestration, PKCE state, credential persistence, concurrent token refresh, provider auth status, logout, and resource cleanup.
2. Preserve the OpenAI path: official Python openai SDK (prefer AsyncOpenAI), official Sign in with ChatGPT/SIWC OAuth + PKCE, secure system credential storage, and provider execution through supported APIs. Confirm Responses/SIWC restrictions remain upstream-owned.
3. Prohibit ~/.codex/auth.json parsing, Codex credential reuse/extraction, Codex CLI inference, and undocumented ChatGPT/Codex backend APIs. Codex remains a development tool only.
4. Separate Google SDK `GEMINI_API_KEY` / Vertex / ADC authentication from personal-plan `agy` CLI authentication. The CLI owns browser login, session, and keyring; no SDK reuse of private CLI credentials.
5. Define high-level documented status/login/logout boundaries for the CLI, including honest unavailable/manual guidance if no supported mechanism exists. Never inspect private Antigravity keyring entries, scrape Google OAuth tokens, or call private Cloud Code/auth APIs.
6. Define conceptual local-only /auth/* administrative endpoints versus /v1/* compatibility endpoints; no endpoint implementation. Preserve 127.0.0.1 default binding and local API-key secrecy. Credentials never enter CanonicalRequest, catalog entries, user-facing status, or responses.
7. Produce a threat matrix covering token leakage, structured logs, URLs/query strings, browser callback state and replay, credential persistence, refresh races/rotation, local API exposure, and subprocess environment/argument/stdout/stderr leakage. Assign control owner, failure behavior, residual risk, and future validation case.
8. Define secure credential-store access as middleware-owned secrets only; reconcile status/expiry/unavailability transitions with Task 05 and errors/cancellation with Task 06. Do not include real tokens or inspect actual stores.
9. Record secret-safe sequence diagrams for OpenAI login/refresh/logout and Google CLI status delegation. Use symbolic placeholders; document boundaries rather than undocumented endpoint details.

## Expected Deliverables

Future output: `docs/architecture/security-and-auth-boundaries.md`.

The document must contain auth/credential ownership matrix, conceptual interfaces and sequences, trust boundaries, threat/control matrix, forbidden-access list, and safe status/error rules.

Use prose, tables, diagrams, and embedded synthetic examples. Include sources,
decisions/rationale, unresolved/deferred items, reviewer disposition, and the
completion report below. Do not create this architecture document while merely
planning the task set. If a documentation convention appears before execution,
the Coordinator must record any path migration and update dependent references.

## Acceptance Criteria

- All credential lifecycle operations have owners separate from transport and canonical data.
- OpenAI SIWC/PKCE and Google SDK/CLI authentication remain distinct and supported.
- Private credential access, secret logging, undocumented auth APIs, and Codex inference are explicitly prohibited.
- Local-only auth endpoints and 127.0.0.1 remain fixed boundaries.
- Each threat has an owned mitigation and later validation case; no blocking security finding remains.
- Existing instructions, user changes, and unrelated files remain untouched.

## Validation / Quality Gates

Review every trust-boundary crossing for secret exposure, including cancellation, refresh concurrency, logout, and CLI environment propagation. Cross-check catalog/auth states and error outputs; use document review only, with no login or keyring access.

Check document links, field/term consistency, evidence traceability, scope, and
secret-free examples. Review the final diff and run `git diff --check` when Git
is usable; account for untracked documents explicitly. If unavailable, compare
filesystem contents/hashes and report the limitation. Ruff, mypy/pyright, and pytest
are not applicable to documentation-only execution; no code/test work is authorized.
Do not claim a future implementation test has passed from a paper review.

## Stop Conditions

Any proposal to extract credentials, log secrets, use undocumented auth backends, expose administration publicly, or rely on unverified CLI status mechanisms blocks acceptance. Conflicting lifecycle ownership returns to the Coordinator.

Stop the affected work and report to the Coordinator if instructions/references
conflict, prerequisites change materially, implementation is needed to finish,
or a security boundary would be violated. Preserve evidence and report the missing
decision; do not silently rewrite project rules or mark blocked work complete.

## Handoff

Submit the accepted security boundary document, threat findings, and residual nonblocking unknowns to Task 08. Any upstream contract correction must reopen the affected task before freeze.

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
