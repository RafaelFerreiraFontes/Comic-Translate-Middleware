# Project Instructions

This document defines durable architecture and development rules.
[MVP instructions](mvp.md) define the mandatory scope and completion criteria of
the first usable product; read both before planning or implementing MVP work.

## Project Mission

`comic-translate-middleware` is a local middleware / Bridge Provider exposing an
OpenAI-compatible API to Comic Translate and adapting requests to supported
upstream providers. Comic Translate requires **zero modifications**: the
middleware adapts to Comic Translate, never the other way around.

## Core Architecture

```text
app/
├── api/
├── compatibility/
├── providers/
├── auth/
├── models/
└── config/
```

| Boundary | Responsibility |
| --- | --- |
| `api/` | HTTP transport only: validate input, resolve dependencies, call application/provider services, return responses. No provider, authentication, or business logic. |
| `compatibility/` | Provider-agnostic OpenAI Chat Completions compatibility: normalize public requests and translate public contracts to/from internal models. Never import concrete providers. |
| `providers/` | Concrete upstream integrations: OpenAI, Antigravity SDK, Antigravity CLI, and future providers. |
| `auth/` | OAuth, PKCE, credential lifecycle, and secure credential storage. |
| `models/` | Pydantic/domain contracts. |
| `config/` | Application configuration. |

`CanonicalRequest` is the internal source of truth. Every provider consumes it
and returns `InferenceResult`, through a common abstraction exposing inference,
model discovery, and capabilities. Providers do not consume Comic Translate
payloads directly. Keep protocol conversion separate from upstream execution;
Responses conversion helpers must not introduce concrete provider dependencies
into `compatibility/`.

```text
Comic Translate
      ↓
OpenAI-compatible HTTP API
      ↓
Compatibility Layer
      ↓
CanonicalRequest
      ↓
Provider Router
      ↓
Provider
      ↓
InferenceResult
      ↓
Compatibility Layer
      ↓
OpenAI-compatible response
      ↓
Comic Translate
```

The router depends on provider abstractions and selects by namespaced public
model IDs, such as `openai:<model>` and `google:<model>`. Never expose bare model
names where the public contract requires a namespace. An upstream model slug
may have its namespace removed after routing. Google routing also distinguishes
SDK and CLI authentication modes.

## OpenAI Provider Rules

- Use the official Python `openai` SDK; prefer `AsyncOpenAI` for I/O.
- ChatGPT plan access uses the official Sign in with ChatGPT (SIWC) OAuth +
  PKCE flow, with a loopback callback. Keep token lifecycle in `auth/`, store
  credentials securely, and protect refresh rotation against concurrent refresh.
- Use the Responses API internally where required; SIWC uses the official
  `https://api.openai.com/v1/responses` endpoint. Comic Translate continues to
  see Chat Completions-compatible requests and responses.
- For the ChatGPT plan / SIWC path, always set:

  ```python
  store=False
  stream=True
  ```

- Filter unsupported Responses parameters instead of blindly forwarding Chat
  Completions fields. In particular, do not forward `temperature` or
  `max_completion_tokens` on the plan path; consult the specialized reference
  for other restrictions. Preserve system instructions through supported mapping.
- Collect internal streams until `response.completed`; treat `response.failed`
  as failure. Keep streaming and non-streaming paths distinct and tested.
  Return the complete response Comic Translate expects, including valid
  `choices[0].message.content`.
- Preserve multimodal image input, including base64 data URIs and conversion to
  Responses `input_image`. Never silently drop images.
- Prefer authenticated model discovery and demonstrated capabilities over
  hard-coded subscription-plan assumptions.
- Never read `~/.codex/auth.json`, other Codex credential files, or private
  Codex/ChatGPT session files. Never call undocumented ChatGPT/Codex backends.

## Codex CLI Boundary

Codex CLI is allowed as a **development tool for contributors and coding agents**.
It is not part of the middleware inference architecture. The middleware must not
implement Codex CLI as its default OpenAI provider, spawn `codex exec` (or other
Codex invocations) for Comic Translate inference requests, extract Codex
credentials, or depend on Codex's local authentication store. Historical
experimental possibilities in the reference do not authorize an inference path.

## Google / Antigravity Rules

### Antigravity SDK provider

`AntigravitySdkProvider` uses the official SDK with officially supported
`GEMINI_API_KEY`, Vertex, or ADC authentication. Map `CanonicalRequest` to SDK
input and SDK output to `InferenceResult`; preserve images through supported
image conversion. Use structured output only where supported.

### Antigravity CLI provider

`AntigravityCliProvider` uses the official `agy` CLI/headless interface for
personal Google/Antigravity plan access. The CLI owns browser login, session,
and keyring; the middleware must never extract Google OAuth credentials from
Antigravity keyring entries or private files, or reuse that session in the SDK.
Never call undocumented/private Google backends, including Cloud Code APIs.

Validate executable discovery, subprocess isolation, stdout/stderr and
JSON/NDJSON parsing, terminal result status, exit codes, timeout, cancellation,
and error mapping. Discover models and report actual capabilities. If headless
multimodal input is unconfirmed, declare that limitation honestly; never claim
support or silently discard images. Document any supported text-only fallback.

## Security Rules

These constraints are **non-negotiable**:

- Bind locally to `127.0.0.1` by default; no public binding in the MVP.
- Never log access tokens, refresh tokens, ID tokens, API keys, or authorization
  codes. Sanitize structured logs, upstream errors, and CLI output before logging.
- Never put OAuth credentials into URLs/query strings.
- Store middleware-managed credentials in the system credential store/keyring
  as defined by the project. This never authorizes reading another tool's keyring.
- Never scrape credentials from other tools or private authentication files.
- Never call undocumented authentication or private backend APIs.
- Treat any local middleware API key as a secret; never expose credentials in
  responses, fixtures, or logs.

## FastAPI Rules

- Use async path operations for I/O-bound routes, thin endpoints, `APIRouter`,
  and `Annotated[..., Depends(...)]`. Keep dependencies explicit and lightweight.
- Use Pydantic for external request contracts and typed responses where practical.
  Keep internal `CanonicalRequest` / `InferenceResult` out of public OpenAPI.
- Manage startup/shutdown and client cleanup with a lifespan context manager,
  not deprecated startup/shutdown handlers.
- Separate `/v1/*` public compatibility routers from local-only `/auth/*`
  administrative routers. Authentication endpoints remain on `127.0.0.1`.
- Return consistent OpenAI-compatible errors for `/v1/*`; map upstream errors
  through an internal error model and include a request ID when available.
  Never leak stack traces or credentials.

## Code Quality

Follow DRY, KISS, Separation of Concerns, and SOLID, especially single
responsibility and dependency inversion. Prefer explicit code over clever code,
descriptive PEP 8 names, small focused functions/modules, explicit isolated side
effects, dependency injection where appropriate, and composition over deep
inheritance. Avoid speculative abstractions, circular imports, flag-driven
multi-purpose functions, commented-out code, and silent exception swallowing.
Comments explain why; keep docstrings useful and consistent.

Use type hints on public interfaces and attributes, modern Python typing
(`list[...]`, `dict[...]`, `|`), and Pydantic data contracts. Replace magic
strings/numbers with named constants where appropriate. Maintain architecture
boundaries even when a shortcut would require fewer lines.

## Tooling and Testing

Expected quality gates are **Ruff** (lint/format), **mypy or pyright** (types),
and **pytest**. Use repository configuration and relevant checks; new public
behavior requires corresponding tests.

Cover unit, provider, contract, and integration tests when relevant, including
multimodal fixtures, OAuth/error paths, malformed upstream data, timeouts and
cancellation, streaming collectors, parameter filtering, and Comic Translate
request/response compatibility. Mock upstreams for focused tests; keep live
integration tests marked and optional as the references specify. No real secrets
in tests or fixtures. Report failures and unavailable checks honestly.

## Git / Change Discipline

Inspect the repository, current branch, working tree, references, and relevant
tests before modifying files. Preserve existing user changes, including unrelated
modified/untracked files. Keep tasks narrowly scoped; avoid unrelated refactors
and never discard or overwrite unrelated work. Inspect the final diff and run
applicable checks before declaring completion. Never commit or push unless the
user explicitly requests it. If Git metadata or tooling is unavailable, report
the limitation rather than claiming a check passed.

## Definition of Done

A task is not complete merely because code was written:

1. Requested behavior implemented.
2. Architecture boundaries preserved.
3. Security constraints preserved.
4. Appropriate tests added/updated.
5. Relevant tests pass.
6. Lint/format checks pass where applicable.
7. Type checks pass where applicable.
8. Final diff reviewed.
9. No secrets introduced.
10. No unrelated modifications.

For documentation-only changes, validate content, links, consistency, and scope;
explain which code checks are not applicable. Do not report required failing or
unavailable checks as passing.

## Collaboration and Conflict Resolution

Use the smallest necessary set of agents: Coordinator plans and integrates;
Architecture & Security reviews contracts/security; Implementation & Testing
implements bounded tasks; Google / Antigravity validates Google-specific work.
Significant changes need independent review before Coordinator final QA.

Within project instructions, priority is:

1. Explicit user instruction for the current task.
2. `.github/instructions/project.md`.
3. Specialized project reference documents.
4. Existing repository architecture/contracts.
5. Agent-specific preferences.

Never interpret a task as permission to leak credentials, scrape private
authentication data, or call prohibited undocumented authentication APIs. If
architecture/reference documents disagree, stop the affected implementation and
report the conflict to the Coordinator; do not silently choose an interpretation.

## Mandatory References

This file is the consolidated operational instruction layer. These original
documents remain authoritative specialized references; read the applicable
documents before changes and resolve conflicts using the rules above. Links are
repository-relative, resolved from this file's directory:

- [`.github/instructions/clean-code-python.md`](clean-code-python.md)
- [`.github/instructions/fastapi-reference-python.md`](fastapi-reference-python.md)
- [`.github/instructions/google-antigravity-sdk-python.md`](google-antigravity-sdk-python.md)
- [`.github/instructions/openai-sdk-python.md`](openai-sdk-python.md)
- [`.github/instructions/openai-codex-cli.md`](openai-codex-cli.md)
- [`.github/instructions/mvp.md`](mvp.md)
