# OpenAI Python SDK — Comic Translate Middleware

This document is the **project-specific instruction set** for using the official OpenAI Python SDK inside `comic-translate-middleware`.  
It is based on:

- [OpenAI Python API library](https://developers.openai.com/api/reference/python)
- [Responses API](https://platform.openai.com/docs/api-reference/responses)
- [Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
- [Sign in with ChatGPT – open-source / token sharing](https://developers.openai.com/siwc/token-sharing-open-source)
- [Models and inference (SIWC)](https://developers.openai.com/siwc/token-sharing-open-source/models-and-inference)
- [Preview limitations (SIWC)](https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations)

and is strictly aligned with the architecture defined in the project plan.

---

## 1. High-Level Decision (from project plan)

The middleware exposes an **OpenAI-compatible** surface to Comic Translate:

```text
POST /v1/chat/completions
```

Internally, when the selected model belongs to the OpenAI / ChatGPT plan path, the provider must:

1. Accept a Chat Completions-style request from Comic Translate.
2. Normalize it into the internal `CanonicalRequest`.
3. Call the **Responses API** (`client.responses.create`) using either:
   - a classic `OPENAI_API_KEY`, or
   - an OAuth access token obtained via **Sign in with ChatGPT** (open-source flow).
4. Collect the streamed response and convert it back into a Chat Completions-shaped response that Comic Translate understands.

**Do not** use the Codex SDK or scrape `~/.codex/auth.json`.  
Use the official `openai` Python package + the official Sign in with ChatGPT OAuth flow.

---

## 2. Installation & Client Instantiation

```bash
pip install openai
```

### Classic API key

```python
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key=os.environ["OPENAI_API_KEY"],
    # base_url can be left default
)
```

### OAuth access token (Sign in with ChatGPT – plan usage)

```python
client = AsyncOpenAI(
    api_key=oauth_access_token,   # the token is sent as Bearer
    base_url="https://api.openai.com/v1",
    max_retries=0,                # recommended for plan-usage flow
)
```

Prefer the **async** client (`AsyncOpenAI`) throughout the middleware.

---

## 3. Two Authentication Modes

| Mode | Credential | Endpoint used | Notes |
|------|------------|---------------|-------|
| Classic API key | `OPENAI_API_KEY` | Responses or Chat Completions | Full parameter support |
| ChatGPT plan (SIWC) | OAuth `access_token` | **Responses API only** | Strict limitations (see §5) |

Model discovery for the plan path:

```python
models = await client.models.list()
# Filter / present models that the current account can actually use
```

Never hard-code plan names (`Plus`, `Pro`).  
Source of truth = successful OAuth + `chatgpt.tokens.use.direct` scope + `GET /v1/models` + successful inference.

---

## 4. Core Conversion Pipeline

```text
Comic Translate
      │  POST /v1/chat/completions
      ▼
ChatCompletionRequest (Pydantic)
      │
      ▼  normalize
CanonicalRequest
      │
      ▼  OpenAI Responses Adapter
client.responses.create( ... store=False, stream=True ... )
      │
      ▼  stream collector
accumulate response.output_text.delta
wait for response.completed
      │
      ▼  build
ChatCompletionResponse  →  Comic Translate
```

The adapter lives in `app/compatibility/responses_adapter.py` (or equivalent).  
Providers should not talk to Comic Translate formats directly.

---

## 5. Responses API Requirements for ChatGPT Plan Usage

When using an OAuth access token obtained via Sign in with ChatGPT, the following rules are **mandatory**:

| Requirement | Value / rule |
|-------------|--------------|
| Endpoint | `POST https://api.openai.com/v1/responses` only |
| `store` | **must be `false`** |
| `stream` | **must be `true`** |
| Unsupported fields | Do **not** send: `temperature`, `top_p`, `max_output_tokens`, `metadata`, `user`, `background`, `conversation`, `moderation`, … (see official preview limitations) |
| System messages | Prefer `instructions` or developer-role messages. Explicit `{type: "message", role: "system"}` items may be rejected. |
| Success condition | Only treat the call as successful after receiving the `response.completed` event |
| Error handling | Watch for `response.failed` with codes such as `subscription_sharing_usage_limit_exceeded` |

### Minimal correct call (plan path)

```python
async with client.responses.create(
    model=model_slug,                    # from /v1/models
    input=normalized_input,              # list of messages / content parts
    store=False,
    stream=True,
    # do NOT pass temperature, max_tokens, etc.
) as stream:
    text_parts: list[str] = []
    async for event in stream:
        if event.type == "response.output_text.delta":
            text_parts.append(event.delta)
        elif event.type == "response.failed":
            # map to structured provider error
            ...
        elif event.type == "response.completed":
            break
    full_text = "".join(text_parts)
```

Comic Translate currently sends `temperature` and `max_completion_tokens`.  
The adapter **must ignore or map** these fields; it must never forward unsupported parameters on the plan path.

---

## 6. Multimodal Support

Comic Translate (when `image_input_enabled` is on) sends:

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "..."},
    {
      "type": "image_url",
      "image_url": {
        "url": "data:image/jpeg;base64,..."
      }
    }
  ]
}
```

### Responses API shape

```python
input=[
    {
        "role": "user",
        "content": [
            {"type": "input_text", "text": prompt_text},
            {
                "type": "input_image",
                "image_url": "data:image/jpeg;base64,...",
            },
        ],
    }
]
```

The compatibility layer must convert `image_url` / data-URI content into the Responses `input_image` format.  
Never drop images when the original request contained them.

---

## 7. Chat Completions Compatibility Layer

Even though the upstream call uses the Responses API, the **public contract** of the middleware remains Chat Completions:

**Request (from Comic Translate)**

```json
{
  "model": "openai:gpt-...",
  "messages": [...],
  "temperature": 1.0,
  "max_completion_tokens": 5000
}
```

**Response (to Comic Translate)**

```json
{
  "id": "...",
  "object": "chat.completion",
  "created": 1710000000,
  "model": "...",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "<translated JSON or text>"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": { ... }   // optional but nice to have
}
```

The response builder must always produce a valid `choices[0].message.content`.

---

## 8. Provider Implementation Notes (`OpenAISiwcProvider`)

Recommended location: `app/providers/openai_siwc.py`

Responsibilities:

- Accept `CanonicalRequest`
- Select credentials (API key vs OAuth access token)
- Build the correct Responses payload (respecting plan limitations when applicable)
- Drive the stream to completion
- Map usage, finish reason, and errors into `InferenceResult`
- Expose capabilities:

```python
ProviderCapabilities(
    supports_images=True,
    supports_system_prompt=True,
    supports_streaming=True,          # internal streaming
    supports_structured_output=False, # or True if we add schema support later
    auth_mode="api_key_or_siwc",
    available_models=...,
)
```

Model IDs exposed to Comic Translate should be namespaced:

```text
openai:gpt-4o
openai:gpt-4.1
openai:gpt-6.1-sol          # example plan model slug
```

---

## 9. OAuth / Sign in with ChatGPT Integration

Handled in `app/auth/openai_oauth.py` (and related modules).

Key points:

- Use the **official open-source registration + PKCE** flow.
- Required scopes include at least:
  ```text
  openid profile email offline_access
  resource.invoke
  chatgpt.tokens.use.direct
  ```
- Callback must be a loopback address (`http://127.0.0.1:<port>/auth/openai/callback`).
- Store `access_token`, `refresh_token`, `id_token`, `client_id`, scopes and expiry securely (system keyring via `keyring`).
- Never log tokens.
- Refresh tokens must be rotated safely; protect concurrent refresh with a lock.
- Model catalog must be fetched with the current access token (`GET /v1/models`).

Administrative endpoints (local only):

```text
POST /auth/openai/login
GET  /auth/openai/callback
GET  /auth/openai/status
POST /auth/openai/logout
```

---

## 10. Error Handling

Map upstream errors into the middleware’s structured error model.

Important SIWC error codes:

| Code | Meaning | Suggested HTTP status |
|------|---------|-----------------------|
| `subscription_sharing_usage_limit_exceeded` | Plan usage limit reached | 429 |
| `subscription_sharing_usage_unavailable` | Plan usage temporarily unavailable | 503 / 429 |
| authentication / invalid token | Token expired or revoked | 401 |
| model not available for account | Wrong model slug | 400 / 404 |

Always prefer returning an OpenAI-compatible error shape when the request came through `/v1/chat/completions`.

---

## 11. Security Rules (mandatory)

- Never read `~/.codex/auth.json` or any private ChatGPT session files.
- Never call `chatgpt.com/backend-api` or other undocumented endpoints.
- Never put OAuth tokens in query strings or logs.
- Store tokens only in the system credential store (`keyring`).
- Bind the middleware to `127.0.0.1`.
- Treat the optional local API key (`ctm_local_...`) as a secret.

---

## 12. Testing Strategy

- **Unit tests** for the Responses ↔ Chat Completions adapter (with and without images).
- **Contract tests** that feed real Comic Translate payloads and assert the returned `choices[0].message.content` shape.
- **Integration tests** (optional, marked) using a real API key and, separately, a mocked or live SIWC token.
- Stream collector tests: ensure we only finish after `response.completed` and correctly handle `response.failed`.
- Parameter filtering tests: verify that `temperature` / `max_completion_tokens` are **not** forwarded on the plan path.

---

## 13. Implementation Checklist (Wave 04)

- [ ] Add `openai` dependency
- [ ] Implement OAuth 2.0 + PKCE + loopback callback
- [ ] Secure credential store (keyring)
- [ ] Token refresh with locking
- [ ] `OpenAISiwcProvider` implementing `BaseProvider`
- [ ] Responses adapter (`store=False`, `stream=True`, event collector)
- [ ] Multimodal conversion (`image_url` → `input_image`)
- [ ] Chat Completions response builder
- [ ] Model listing via `client.models.list()`
- [ ] Error mapping (including subscription limit codes)
- [ ] Contract tests with Comic Translate payloads
- [ ] Local-only auth endpoints under `/auth/openai/*`

---

## 14. Official Links

| Resource | URL |
|----------|-----|
| Python SDK reference | https://developers.openai.com/api/reference/python |
| Responses API | https://platform.openai.com/docs/api-reference/responses |
| Chat Completions API | https://platform.openai.com/docs/api-reference/chat |
| Sign in with ChatGPT (OSS) | https://developers.openai.com/siwc/token-sharing-open-source |
| Registration & sign-in | https://developers.openai.com/siwc/token-sharing-open-source/sign-in |
| Models & inference | https://developers.openai.com/siwc/token-sharing-open-source/models-and-inference |
| Preview limitations | https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations |
| Errors & recovery | https://developers.openai.com/siwc/token-sharing-open-source/errors-and-recovery |
| GitHub – openai-python | https://github.com/openai/openai-python |

---

## 15. Summary for Contributors

1. Comic Translate talks **Chat Completions**; the middleware talks **Responses** upstream when using ChatGPT plan tokens.
2. Always set `store=False` and `stream=True` on the plan path and collect until `response.completed`.
3. Never forward unsupported parameters (`temperature`, etc.) on the SIWC path.
4. Multimodality is required for full Comic Translate quality — preserve images.
5. Use only the official OAuth flow and the official `openai` package.
6. Keep the public contract of the middleware stable so that **zero changes** are needed in Comic Translate.

This document is part of the project’s mandatory instruction set.  
Update it when OpenAI changes the Responses API contract or the Sign in with ChatGPT preview limitations.