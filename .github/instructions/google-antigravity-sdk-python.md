# Google Antigravity SDK & CLI — Comic Translate Middleware

This document is the **project-specific instruction set** for integrating Google Antigravity into `comic-translate-middleware`.  
It is based on the official documentation:

- [Home](https://antigravity.google/docs/home/)
- [SDK Overview](https://antigravity.google/docs/sdk/overview/)
- [CLI Overview](https://antigravity.google/docs/cli/overview/)
- [CLI Installation & Auth](https://antigravity.google/docs/cli/install/)
- [CLI Headless](https://antigravity.google/docs/cli/headless)
- [Structured Output / Multimodal](https://antigravity.google/docs/sdk/structured-output/)

and is strictly aligned with the architecture defined in the project plan.

---

## 1. High-Level Decision (from project plan)

The middleware **must support two distinct Google adapters**:

| Provider class              | Auth mode                          | Implementation surface      | Use case |
|-----------------------------|------------------------------------|-----------------------------|----------|
| `AntigravitySdkProvider`   | `GEMINI_API_KEY` or Vertex / ADC  | Official Python SDK         | API-key users, Enterprise / Vertex |
| `AntigravityCliProvider`   | Personal Google Account / plan    | Official `agy` CLI (headless) | Users who want to use their Antigravity / Google AI subscription |

**Critical constraint (confirmed by upstream issue #20):**  
As of the current SDK versions, `google-antigravity` **does not** support reusing the personal Google Account OAuth / CLI keyring session.  
Therefore we **must not** attempt to extract tokens from the system keyring or call private Cloud Code endpoints.  
The official and supported way to use a personal plan is through the `agy` CLI.

---

## 2. Official Surfaces Summary

### 2.1 Antigravity SDK (`google-antigravity`)

- Python package: `pip install google-antigravity`
- Primary entry points: `Agent` + `LocalAgentConfig`
- Designed for building autonomous agents with tools, policies, subagents, multimodal input, and structured output.
- Authentication:
  - `GEMINI_API_KEY` environment variable (consumer API key)
  - Vertex AI / Gemini Enterprise via `vertex=True` + `project` + `location` (or ADC)
- **Does not** currently expose a first-class consumer OAuth / plan-sharing flow.

### 2.2 Antigravity CLI (`agy`)

- Lightweight TUI + powerful headless mode.
- Installation:
  - macOS / Linux: `curl -fsSL https://antigravity.google/cli/install.sh | bash`
  - Windows: PowerShell `irm https://antigravity.google/cli/install.ps1 | iex`
- Authentication:
  - Silent keyring reuse when a session already exists
  - Browser-based Google Account sign-in (personal or Business / Gemini Enterprise)
  - Optional pure `GEMINI_API_KEY` mode (requires `modelProvider: "gemini"` in settings)
- Headless / non-interactive mode is first-class:
  ```bash
  agy -p "prompt" --output-format json
  agy -p "prompt" --output-format stream-json
  ```

---

## 3. Mapping to Middleware Architecture

```text
Comic Translate
      │
      │  POST /v1/chat/completions
      ▼
Compatibility Layer  →  CanonicalRequest
      │
      ▼
Provider Router
      │
      ├── model starts with "google:" and auth_mode == "sdk"
      │         └── AntigravitySdkProvider  (google-antigravity)
      │
      └── model starts with "google:" and auth_mode == "cli" / plan
                └── AntigravityCliProvider  (subprocess agy -p)
```

Model namespace examples:

```text
google:gemini-2.0-flash
google:gemini-2.5-pro
google:gemini-3-flash          # names will evolve; always discover
```

---

## 4. AntigravitySdkProvider (SDK path)

### 4.1 Installation & configuration

```bash
pip install google-antigravity
export GEMINI_API_KEY="..."
# or
export GOOGLE_GENAI_USE_VERTEXAI=true
export GOOGLE_CLOUD_PROJECT="..."
export GOOGLE_CLOUD_LOCATION="us-central1"
gcloud auth application-default login
```

### 4.2 Basic usage pattern (recommended for the provider)

```python
from google.antigravity import Agent, LocalAgentConfig
from google.antigravity.types import Image, from_file  # for multimodal

config = LocalAgentConfig(
    # system_instructions=...,          # optional
    # response_schema=SomePydanticModel, # optional structured output
    # vertex=True, project=..., location=...  # for Enterprise
)

async with Agent(config) as agent:
    # Text-only
    response = await agent.chat("Translate the following JSON...")
    text = await response.text()

    # Multimodal (image from base64 or file)
    image = Image(data=image_bytes, mime_type="image/jpeg")
    # or Image.from_file("page.jpg") / from_file(...)
    response = await agent.chat(["Translate the text in this comic page", image])
    text = await response.text()
```

### 4.3 Multimodal support (critical for Comic Translate)

Comic Translate sends:

```json
{
  "type": "image_url",
  "image_url": {
    "url": "data:image/jpeg;base64,..."
  }
}
```

The SDK provider **must**:

1. Detect `image_url` / `data:image/...;base64,...` in the CanonicalRequest.
2. Decode the base64 payload.
3. Construct an `Image` (or equivalent) object.
4. Pass a mixed list `[text_prompt, image, ...]` to `agent.chat()`.

Never drop images when `image_input_enabled` is active.

### 4.4 Structured output

When useful (e.g. forcing the model to return pure translated JSON), the provider may use:

```python
config = LocalAgentConfig(response_schema=TranslatedPageSchema)
...
data = await response.structured_output()
```

For the MVP it is acceptable to rely on careful prompting + post-processing of `response.text()`.

### 4.5 Streaming

The SDK supports token streaming (`async for token in response`).  
For the middleware MVP we usually collect the full response (Comic Translate expects a complete Chat Completions object).  
Streaming can be added later for observability or progressive UI.

### 4.6 Capabilities declaration

```python
ProviderCapabilities(
    supports_images=True,
    supports_system_prompt=True,
    supports_streaming=True,
    supports_structured_output=True,
    auth_mode="api_key_or_vertex",
    available_models=...,  # discovered or configured
)
```

---

## 5. AntigravityCliProvider (CLI / personal plan path)

This is the **only supported way** to use a personal Google Account / Antigravity subscription inside the middleware.

### 5.1 Responsibilities of the provider

- Detect whether `agy` is installed and on `PATH`.
- Verify that a usable session exists (or guide the user to run `agy` once interactively).
- Invoke headless mode:
  ```bash
  agy -p "<prompt>" --output-format stream-json
  # or --output-format json
  ```
- Parse NDJSON (`stream-json`) or the final JSON object.
- Extract the final response text, usage, conversation_id, and errors.
- Handle timeouts, cancellation, and non-zero exit codes.
- **Never** read the system keyring or `~/.gemini/...` auth files directly.

### 5.2 Recommended invocation flags

| Flag / option              | Purpose |
|---------------------------|---------|
| `-p` / `--print` / `--prompt` | Headless one-shot mode |
| `--output-format json`    | Single JSON result (simpler) |
| `--output-format stream-json` | NDJSON events (richer, preferred for progress) |
| `--print-timeout <duration>` | Guard against hung runs |
| `--model <name>`          | Force a specific model when supported |
| `--dangerously-skip-permissions` | Only if the middleware runs in a controlled sandbox and tools are not needed |

For pure translation we usually do **not** need tools; keep the prompt focused on text (+ optional image description or base64 if the CLI supports multimodal input in headless mode).

### 5.3 Output parsing (`stream-json`)

Typical event sequence:

```json
{"event":"init", ...}
{"event":"step_update", "step_update":{"step_type":"agent_response", "text_delta":"...", "state":"ACTIVE"}}
...
{"event":"result", "result":{"status":"SUCCESS", "response":"...", "usage":{...}}}
```

The provider must:

- Accumulate `text_delta` fragments.
- Wait for the final `result` event.
- Map `status`, `response`, and `usage` into `InferenceResult`.
- Treat non-`SUCCESS` status or non-zero process exit as provider errors.

### 5.4 Multimodal with the CLI

As of current documentation, headless multimodal support (passing images) should be verified experimentally.  
If the CLI does not accept image attachments in `-p` mode, the CliProvider should:

- Clearly declare `supports_images=False` (or limited), **or**
- Fall back to a pure-text prompt that includes OCR / description if available,
- and document the limitation.

The SdkProvider remains the preferred path when images are required and an API key is available.

### 5.5 Session / auth management

- The CLI owns the keyring and browser OAuth flow.
- Middleware only exposes high-level endpoints:
  - `GET  /auth/google/status`
  - `POST /auth/google/login`  → may simply instruct the user to run `agy` once, or open a helper
  - `POST /auth/google/logout` → can call `agy` logout if a stable non-interactive way exists; otherwise document manual `/logout`
- Never implement token extraction.

---

## 6. Provider Contract Alignment

Both providers must implement the common interface defined in the project:

```python
class BaseProvider(ABC):
    @abstractmethod
    async def list_models(self) -> list[ModelInfo]: ...

    @abstractmethod
    async def infer(self, request: CanonicalRequest) -> InferenceResult: ...

    @property
    @abstractmethod
    def capabilities(self) -> ProviderCapabilities: ...
```

`CanonicalRequest` already contains:

- system prompt
- messages (text + optional images)
- model id (without the `google:` prefix after routing)
- timeout / output format hints

The providers translate this into either:

- SDK `agent.chat(...)` calls, or
- a carefully constructed `agy -p` prompt + flags.

---

## 7. Security Rules (mandatory)

These rules are non-negotiable and match the project security model:

- **Never** read `~/.codex/auth.json`, Antigravity keyring entries, or any private auth files.
- **Never** extract or log OAuth access / refresh tokens.
- **Never** call undocumented Cloud Code / backend-api endpoints.
- Prefer letting the official `agy` binary manage personal authentication.
- When using the SDK, only use the documented `GEMINI_API_KEY` or Vertex / ADC paths.
- Bind the middleware itself to `127.0.0.1`.
- Treat any local API key (`ctm_local_...`) as a secret.

---

## 8. Error Handling & Reliability

Map upstream failures into the middleware’s structured error model:

| Situation                        | Suggested handling |
|----------------------------------|--------------------|
| `agy` not found on PATH          | Clear 503 / configuration error with install instructions |
| No active CLI session            | 401-style error + guidance to run `agy` once |
| Headless timeout                 | 504 / timeout error, cancel process |
| Non-zero exit / `status != SUCCESS` | Provider error with stderr / AGY_ERROR payload |
| SDK authentication failure       | 401 / 403 with actionable message |
| Rate / quota limits              | 429 with retry-after if available |
| Multimodal not supported in path | 400 with clear capability message |

Always include a `request_id` when possible.

---

## 9. Testing Strategy

### SdkProvider

- Unit tests with mocked `Agent` / responses.
- Integration tests (optional, marked) using a real `GEMINI_API_KEY`.
- Multimodal fixture: base64 JPEG + text prompt → verify image is passed.

### CliProvider

- Detect `agy` presence.
- Contract tests that spawn `agy -p "ping" --output-format json` (or stream-json) and parse the result.
- Timeout and cancellation tests.
- Error-path tests (missing binary, invalid flags, non-SUCCESS status).

Both providers must pass the same Comic Translate contract tests (page JSON in → translated JSON out).

---

## 10. Implementation Checklist (Wave 05 + Wave 06)

**Wave 05 – AntigravitySdkProvider**

- [ ] Add `google-antigravity` dependency
- [ ] Implement `AntigravitySdkProvider` against `BaseProvider`
- [ ] Support `GEMINI_API_KEY` and Vertex / ADC
- [ ] Multimodal path (base64 → `Image`)
- [ ] Map `CanonicalRequest` → `agent.chat` → `InferenceResult`
- [ ] Capability flags and basic model listing
- [ ] Unit + optional integration tests

**Wave 06 – AntigravityCliProvider**

- [ ] Detect `agy` binary
- [ ] Headless invocation (`-p`, `--output-format json|stream-json`)
- [ ] Robust NDJSON / JSON parser
- [ ] Timeout, cancellation, exit-code handling
- [ ] Status / login / logout endpoints (high-level only)
- [ ] Capability declaration (especially `supports_images`)
- [ ] Contract tests with real or mocked CLI

---

## 11. Official Links (keep updated)

| Resource | URL |
|----------|-----|
| Docs Home | https://antigravity.google/docs/home/ |
| SDK Overview | https://antigravity.google/docs/sdk/overview/ |
| SDK Structured / Multimodal | https://antigravity.google/docs/sdk/structured-output/ |
| CLI Overview | https://antigravity.google/docs/cli/overview/ |
| CLI Install & Auth | https://antigravity.google/docs/cli/install/ |
| CLI Headless | https://antigravity.google/docs/cli/headless |
| SDK GitHub | https://github.com/google-antigravity/antigravity-sdk-python |
| CLI GitHub | https://github.com/google-antigravity/antigravity-cli |
| OAuth reuse issue | https://github.com/google-antigravity/antigravity-sdk-python/issues/20 |

---

## 12. Summary for Contributors

1. Prefer the **SDK** when the user has a Gemini API key or Vertex credentials — full multimodal and structured-output support.
2. Use the **CLI headless mode** when the user wants to consume a personal Antigravity / Google AI plan — the CLI owns authentication.
3. Never scrape tokens or call private endpoints.
4. Keep both providers behind the same `BaseProvider` interface and CanonicalRequest contract.
5. Multimodality is a first-class requirement for Comic Translate quality; document any limitations of the CLI path clearly.

This document is part of the project’s mandatory instruction set.  
Update it when the upstream SDK or CLI gains native personal-plan OAuth support or changes headless contracts.