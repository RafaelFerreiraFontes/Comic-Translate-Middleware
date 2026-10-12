# MVP Instructions — Comic Translate Middleware

## 1. Purpose

This document is the scope contract for the first usable MVP. It defines required
product outcomes, exclusions, acceptable limitations, blocking failures, and
completion evidence. It adds a scope and product-delivery layer; it does not
replace `project.md` or the specialized technical references.

Architecture, security, protocol correctness, data integrity, and capability
honesty remain mandatory even when an MVP shortcut would reduce implementation
work. Creating this document does not start a Wave or authorize implementation.

## 2. MVP Product Definition

The MVP is a local-first OpenAI-compatible bridge between Comic Translate and
supported upstream AI providers. It succeeds when Comic Translate can configure
the local middleware as an OpenAI-compatible service and send actual translation
inference requests through supported providers with **ZERO source-code
modifications**. The middleware adapts to Comic Translate.

Required provider paths remain in MVP scope:

```text
Comic Translate
       │ OpenAI-compatible API
       ▼
Comic Translate Middleware
       ├── OpenAI
       │     ├── API key
       │     └── Sign in with ChatGPT / SIWC
       └── Google
             ├── Antigravity SDK → API key / Vertex / ADC
             └── agy CLI → personal Google / Antigravity plan
```

An API skeleton, mock server, OAuth demo, provider proof of concept, or architecture
document alone is not a usable MVP. Approved implementation Waves specify the
delivery and validation details for these paths; they cannot silently omit a path
or treat an interface stub as completed provider support. An unavailable upstream
surface or validation environment must be reported, not replaced by a private API
or recorded as a passing gate.

## 3. MVP Success Scenario

A user follows documented local setup, configures supported authentication,
selects an available namespaced model, and points Comic Translate at the middleware
using its existing configuration. A real translation request completes through:

```text
Comic Translate
        ↓
POST /v1/chat/completions
        ↓
Compatibility Layer
        ↓
CanonicalRequest
        ↓
Provider Router
        ↓
Selected Provider
        ↓
InferenceResult
        ↓
Chat Completions-compatible response
        ↓
Comic Translate
```

Comic Translate consumes the returned translation successfully. Supported image
requests retain image context; unsupported requests fail clearly. Validation
records the consumer version, provider/auth path, model, sanitized request/result
evidence, and limitations. No credentials belong in that evidence.

## 4. Mandatory MVP Scope

### 4.1 OpenAI-Compatible Public API

The minimum **project expectation**, subject to confirmation and refinement by
Wave 01 Task 01, is:

```text
GET  /health
GET  /v1/models
POST /v1/chat/completions
```

This is not a claim that Comic Translate calls every listed endpoint. Distinguish
local health/discovery product needs from the **protocol contract confirmed by
Wave 01**. Wave 01 evidence wins over assumptions about paths, base URL `/v1`
composition, headers, fields, model listing, and response consumption. Conflicts
requiring scope changes follow section 11; do not silently rewrite requirements.

Public `/v1/*` behavior remains OpenAI-compatible. Administrative authentication
operations belong under local-only `/auth/*`. Do not freeze their exact contracts
before Wave validation. Produce the complete response expected by the confirmed
consumer, including compatible `choices[0].message.content`.

### 4.2 Canonical Compatibility Layer

Normalize Comic Translate input through `compatibility/` into provider-independent
`CanonicalRequest`. Providers consume that contract and return `InferenceResult`;
compatibility builds the public response. Preserve text, instructions, images,
model identity, and supported options according to the Wave 01 contracts.

Canonical types and provider isolation are mandatory MVP structure, not optional
cleanup for a later release. See section 9 for prohibited shortcuts.

### 4.3 Provider Routing

Route through the common provider abstraction using unambiguous public IDs:
`openai:<model>` and `google:<model>`. Configuration must deterministically select
the Google SDK or CLI transport when both can serve a model. Do not route ambiguous
bare public names, infer availability from subscription names, or add automatic
provider failover. Wave 01 freezes selection and capability-validation semantics.

### 4.4 OpenAI Provider

Use the official Python `openai` SDK; prefer `AsyncOpenAI`. Support the API-key
path through `OPENAI_API_KEY` and the official Sign in with ChatGPT / SIWC path.
ChatGPT-plan access is a core MVP objective and remains required unless the user
explicitly removes it from scope.

For SIWC, use the Responses API with `store=False` and `stream=True`. Collect the
upstream stream until `response.completed`, handle failure or premature termination
as errors, and return the complete canonical result for Chat Completions response
construction. Preserve image inputs. Filter unsupported plan/Responses parameters
instead of blindly forwarding fields such as `temperature` or
`max_completion_tokens` from the public request.

Never read `~/.codex/auth.json`, reuse Codex CLI credentials, spawn `codex exec`
for inference, or call private ChatGPT APIs. Codex CLI is a **development tool
only**. The specialized OpenAI references govern supported auth and API details.

### 4.5 Google Antigravity SDK Provider

Keep `AntigravitySdkProvider` distinct from the CLI provider. Its supported
authentication architecture uses `GEMINI_API_KEY` or Vertex / ADC through official
mechanisms. It consumes `CanonicalRequest` and produces `InferenceResult`.
Multimodal input is part of the intended SDK path; validate actual model support
and perform image conversion within the upstream integration boundary. SDK objects
must not enter the canonical core or couple the compatibility layer to the SDK.

### 4.6 Google Antigravity CLI Provider

Use the official `agy` headless interface for personal Google/Antigravity plan
access. The CLI owns browser authentication, session, keyring, and credential
lifecycle. The middleware never extracts those credentials or reuses them in the SDK.

The MVP provider must handle binary discovery, process invocation/isolation,
stdout, stderr, JSON/NDJSON parsing, exit codes, timeout, cancellation, terminal
result status, and normalized errors. Do not assume success from partial output.
Validate supported flags and output contracts in the appropriate implementation
Wave rather than inventing CLI functionality.

If headless image input is not officially or experimentally verified, declaring
`supports_images = false` is acceptable only when explicit and correctly enforced.
Reject image requests on that path; never silently discard images. Unavailable
CLI multimodality does not block the whole MVP when the SDK path provides working
Google multimodality. The CLI's supported text/personal-plan path still remains
in scope.

### 4.7 Authentication

Where applicable, require OAuth 2.0, PKCE, a loopback callback, secure credential
storage, safe refresh-token rotation, concurrency protection, logout semantics,
and safe authentication status. Middleware-owned credential lifecycle belongs in
`auth/`; HTTP handlers remain thin. Google CLI credentials remain owned by `agy`.

Potential local administrative routes include `/auth/openai/login`,
`/auth/openai/callback`, `/auth/openai/status`, and `/auth/openai/logout`.
Analogous Google status/control operations use only supported high-level mechanisms;
manual CLI login/logout guidance is acceptable where automation is unsupported.
Wave execution determines exact contracts without extracting private credentials.

### 4.8 Model Discovery and Capabilities

Discover models and capabilities wherever supported. Hard-coded subscription-plan
names such as Plus, Pro, or Ultra are never the source of truth. Where discovery
is unavailable, explicitly distinguish configured information from verified support.

Represent `ModelInfo`, `ProviderCapabilities`, provider availability, model
availability, and authentication state coherently. Distinguish provider, model,
and runtime capability differences; do not advertise unknown capabilities as
confirmed. Wave 01 freezes the exact schemas, namespaces, and catalog semantics.

### 4.9 Multimodal Input

Preserve text plus `image_url` and `data:image/...;base64,...` content as
provider-neutral canonical parts. Never silently discard images or turn an
unsupported image request into a misleading text-only success. Any text-only
fallback must be explicitly requested and documented; otherwise reject unsupported
capabilities clearly.

At least one fully supported intended production provider path must demonstrate
end-to-end image preservation and inference before MVP completion. Preferred
paths are OpenAI and Antigravity SDK, subject to availability and actual capability
validation. This minimum demonstration does not remove other scoped provider paths.

### 4.10 Error Handling and Reliability

Normalize failures through:

```text
Provider error → Internal middleware error → OpenAI-compatible HTTP error
```

Cover invalid requests, authentication and authorization failures, unavailable or
missing models, unsupported capabilities, rate limits/quota, provider unavailability,
timeouts, malformed provider responses, and internal failures. Use appropriate
400, 401, 403, 404, 429, 500, 502, 503, and 504 outcomes under the Wave 01 mapping.
Include a request ID when available.

Define and enforce timeout, cancellation, stream-completion, and subprocess-cleanup
behavior. Any retries must have explicit ownership and bounds under the approved
architecture. Never expose credentials, raw sensitive upstream output, or stack
traces in responses or logs.

### 4.11 Configuration and Local Runtime

Provide a locally runnable FastAPI service with documented setup and configuration.
Bind to `127.0.0.1` by default; public `0.0.0.0` binding is not an MVP default or
requirement. Environment variables and safe local configuration must support basic
use without committing secrets. No hosted infrastructure is required.

### 4.12 Testing and Quality Gates

Require appropriate unit, contract, provider, and integration tests. Mocked
automated coverage must validate core behavior; live tests needing real credentials
may remain marked/optional in routine runs. Optional live test automation does not
waive the working-inference and release integration evidence gates.

Critical coverage includes Comic Translate normalization, Chat Completions response
construction and `choices[0].message.content`, namespaced routing, multimodal
preservation, provider error mapping, unsupported capabilities, parameter filtering,
stream collection, OAuth-sensitive paths, and timeout/cancellation. No real
credentials in fixtures.

Required `pytest`, Ruff lint/format checks, and mypy or pyright checks must pass
according to repository configuration. Report skipped/unavailable checks honestly;
do not claim a mock proves real upstream functionality.

## 5. Explicitly Out of Scope

Unless the user explicitly changes scope, the MVP does not include:

- Comic Translate source modifications; hosted/public SaaS deployment; public
  `0.0.0.0` binding by default.
- Multi-user or multi-tenant accounts, organization/team management, billing,
  usage purchasing, or remote provider credential sharing.
- A cloud database requirement, Redis, queues/workers unless later proven
  necessary and approved, distributed architecture, Kubernetes, or horizontal scaling.
- A web administration dashboard, desktop GUI, mobile application, installer polish
  beyond basic local use, or automatic self-update.
- An advanced observability platform, external telemetry requirement, persistent
  request-history product, analytics dashboard, or plugin marketplace.
- Automatic provider failover, load balancing across providers, complex cost/latency
  routing, or premature performance optimization.
- Codex CLI as an inference provider, private/undocumented OpenAI or Google APIs,
  credential scraping, or reverse engineering third-party keyrings.

Ordinary product exclusions can be reconsidered for future versions. Security
prohibitions remain binding; listing them here is not permission to bypass them
in a future release.

## 6. Acceptable MVP Limitations

Local-only operation, a single local user, manual provider configuration, truthful
provider-specific capabilities, marked/optional credential-dependent live tests,
and explicit lack of Google CLI image support are acceptable within the rules above.
No provider failover, persistent request history, analytics dashboard, distributed
workers, or polished packaging is required beyond basic local usability.

Document limitations in setup/runtime guidance and capability reporting. An
acceptable limitation must never violate security, protocol correctness, data
integrity, or capability honesty. Missing credentials in a development environment
may block verification; they do not turn an unmet completion gate into a pass.

## 7. Blocking MVP Failures

Any of these prevents declaring the MVP complete:

- Comic Translate needs source modifications, or canonical architecture is bypassed.
- No real end-to-end inference path works, or a required provider path remains a stub.
- `choices[0].message.content` is incompatible with the confirmed consumer contract.
- Images are silently discarded or unsupported image requests become misleading successes.
- Model routing is ambiguous or depends on hard-coded subscription-plan names.
- Credentials are scraped from Codex or Antigravity, or private/undocumented auth APIs
  are required for the implementation.
- Tokens/secrets appear in logs; upstream errors leak raw secrets or stack traces.
- The service binds publicly by default.
- Required contract tests or security review fail.
- A provider advertises capabilities it cannot perform.
- SIWC or another required path is silently omitted without an explicit user scope decision.

## 8. Security Requirements

All security constraints in `project.md` remain non-negotiable. Bind locally to
`127.0.0.1`; keep administrative endpoints local. Never log access tokens, refresh
tokens, ID tokens, API keys, authorization codes, or the local middleware API key.
Sanitize structured logs and upstream/CLI diagnostics.

Never put OAuth credentials in URLs/query strings. Protect callback state, PKCE
material, token persistence, concurrent refresh, and logout. Use the system
credential store/keyring for middleware-owned credentials as the project defines;
this does not authorize reading another tool's private keyring or auth files.
Prevent subprocess environment/output leakage. Never scrape Google OAuth tokens,
Codex credentials, or Antigravity sessions, and never use undocumented/private
authentication backends. Credentials never enter canonical requests or fixtures.

## 9. MVP Architecture Constraints

The full rules remain in [project.md](project.md). Essential ownership is:

| Area | Responsibility |
| --- | --- |
| `api/` | HTTP only: validation, dependency resolution, service calls, responses. |
| `compatibility/` | Protocol conversion only; provider-agnostic. |
| `providers/` | Upstream integrations behind the common abstraction. |
| `auth/` | Credential lifecycle and secure storage. |
| `models/` | Typed contracts. |
| `config/` | Application settings. |

`CanonicalRequest` and `InferenceResult` are provider-independent. Providers never
require Comic Translate-specific payloads or build its response objects.
Compatibility never imports concrete providers. HTTP endpoints never directly
invoke external SDKs or CLIs. Provider credentials never enter canonical requests.

Prohibited MVP shortcuts include:

```text
FastAPI endpoint → AsyncOpenAI directly
FastAPI endpoint → google-antigravity directly
FastAPI endpoint → agy subprocess directly
compatibility/ → concrete provider
provider → Comic Translate-specific response object
```

## 10. MVP Completion Gates

Record a PASS / FAIL / BLOCKED result and linked evidence for each gate. Architecture
or mock evidence alone cannot satisfy runtime/provider/integration gates. The
Coordinator accepts completion only after required gates pass and blocking findings
are resolved; explicit user-approved scope changes must be recorded.

| Gate | Objective pass condition |
| --- | --- |
| Protocol Gate | Wave 01 has frozen the evidence-backed Comic Translate contract, including endpoint/field requirements and consumer version. |
| Architecture Gate | Canonical contracts, compatibility, provider interfaces, routing, capabilities, errors, and auth ownership are frozen and preserved by implementation. |
| Runtime Gate | The local FastAPI application starts successfully on `127.0.0.1` using documented configuration. |
| Compatibility Gate | Confirmed Comic Translate requests produce consumable responses with zero consumer source changes. |
| OpenAI Gate | Required OpenAI MVP paths work under the approved implementation Wave and have validation evidence. SIWC/plan access remains in scope unless explicitly removed by the user; API-key success alone does not waive it. |
| Google Gate | Approved SDK and personal-plan CLI paths work through supported official surfaces and report accurate capabilities. CLI personal-plan access uses `agy`, never extracted tokens; the explicit CLI image limitation is allowed under section 4.6. |
| Multimodal Gate | At least one intended production provider demonstrates end-to-end image preservation and inference; unsupported paths reject images clearly. |
| Security Gate | Architecture & Security review has no blocking findings and required security corrections are verified. |
| Test Gate | Required automated tests, Ruff lint/format checks, and configured type checks pass, including core mocked contract coverage. |
| Integration Gate | A real or approved integration validation demonstrates Comic Translate → middleware → provider → middleware → Comic Translate-compatible result. Record actual provider execution evidence; an approved harness may supplement consumer validation, but mocks alone cannot establish a working MVP. |
| Documentation Gate | Local setup, configuration, authentication, provider limitations, and run instructions are documented and match delivered behavior. |

## 11. Scope Change Rules

A task may not expand MVP scope merely because an additional feature would be useful.
For an out-of-scope proposal:

1. Record it as deferred/future work.
2. Do not implement it opportunistically.
3. Continue the current bounded task.

If an MVP requirement genuinely cannot be completed without expanding scope:

1. Stop that affected portion.
2. Explain why the existing MVP cannot be completed otherwise, with evidence.
3. Return the decision to the Coordinator.
4. Require explicit user approval for the scope change before dependent work.

Architecture/security fixes required to satisfy existing MVP guarantees are not
feature creep. Scope reduction, including dropping SIWC or a Google path, also
requires an explicit user decision; a Wave cannot silently redefine completion.

## 12. Relationship to Waves and Tasks

- `mvp.md` defines **what** the first usable product must achieve.
- `project.md` defines **how** the project must be architected and developed.
- Wave README/task files define **when and in what order** work is executed.

Wave tasks may refine implementation details but must not silently contradict MVP
scope. If evidence invalidates an assumption, stop affected work and return the
conflict through the Coordinator. Update the relevant instruction only after an
explicit decision; do not silently rewrite architecture or scope. Existing project
conflict-resolution and security rules still apply.

[Wave 01](../tasks/wave-01/README.md) remains the protocol and architecture freeze
Wave. Completing it is necessary but not sufficient for MVP completion. This scope
document does not start Wave 01, modify its task set, or authorize runtime/provider
implementation during architecture-only work.

## 13. Definition of MVP Done

- [ ] Comic Translate requires zero source changes.
- [ ] Evidence-backed public protocol is implemented.
- [ ] Local service runs on `127.0.0.1`.
- [ ] CanonicalRequest / InferenceResult architecture is preserved.
- [ ] Namespaced model routing works without ambiguity.
- [ ] `/v1/models` behavior works as required by the confirmed protocol and approved scope.
- [ ] `/v1/chat/completions` works with compatible `choices[0].message.content`.
- [ ] OpenAI approved MVP API-key path works.
- [ ] ChatGPT/SIWC path works as scoped; any removal has explicit user approval.
- [ ] Google SDK approved MVP path works with supported authentication.
- [ ] Google personal-plan path works through official `agy` behavior.
- [ ] Multimodal inference works through at least one intended production provider.
- [ ] Unsupported capabilities fail explicitly and limitations are documented.
- [ ] Credentials remain protected.
- [ ] Provider errors normalize correctly; timeout/cancellation behave predictably.
- [ ] Required automated tests, including contract tests, pass.
- [ ] Ruff lint/format checks pass.
- [ ] Configured type checker passes.
- [ ] Security review passes.
- [ ] End-to-end integration validation passes with real inference evidence.
- [ ] Setup, authentication, configuration, limitations, and runtime documentation exist.
- [ ] Coordinator accepts the completion evidence with no blocking findings.

## 14. References

These remain authoritative for their respective technical domains. This document
adds product scope and delivery criteria rather than replacing their rules.

- [Project instructions](project.md)
- [Clean code](clean-code-python.md)
- [FastAPI reference](fastapi-reference-python.md)
- [OpenAI SDK](openai-sdk-python.md)
- [Codex CLI boundary](openai-codex-cli.md)
- [Google Antigravity SDK/CLI](google-antigravity-sdk-python.md)
- [Wave 01 index](../tasks/wave-01/README.md)
- [Wave 01 Task 01 — protocol baseline](../tasks/wave-01/01-comic-translate-protocol-baseline.task.md)
