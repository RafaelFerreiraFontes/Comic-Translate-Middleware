# Comic Translate Protocol Contract

## 1. Purpose

This document freezes the observed public protocol between Comic Translate and a
locally configured OpenAI-compatible middleware. It records what Comic Translate
2.8.9 actually sends and consumes at the pinned source revision below. It does not
define provider payloads, canonical implementation types, OAuth internals, or
middleware runtime implementation.

The primary interoperability target is Comic Translate's **Custom** translator.
That translator permits an arbitrary base URL, API key, and model without source
changes. Claims are classified as:

- **Observed**: directly demonstrated by pinned Comic Translate source.
- **Inferred**: a narrow consequence of observed source, identified as inference.
- **Unconfirmed**: not established by the inspected sources.
- **Not used**: absent from the observed Custom translator call path.

## 2. Research Baseline

| Item | Baseline |
| --- | --- |
| Upstream repository | <https://github.com/ogkalu2/comic-translate> |
| Analyzed revision | `8977b91a4f7a40c3917c5a268e9e7d78e1d818da` |
| Branch/reference | Default branch `main`; tag `v2.8.9` points at the analyzed revision |
| Application version | `2.8.9` |
| Analysis date | 2026-10-12 |
| HTTP dependency | `requests>=2.31.0` |
| OpenAI SDK | Not present in `requirements.txt` and not used by the Custom translator |

Research used a temporary read-only clone outside this project working tree. The
revision was checked out by full SHA. Stable source links in section 17 include
that SHA so the analysis remains reproducible if `main` moves.

No credentials were read, generated, or used. No upstream files were modified.

## 3. Executive Contract Summary

Comic Translate's Custom translator uses direct HTTP through `requests`, not an
OpenAI client SDK. A user supplies a base URL, API key, and model. Comic Translate
strips trailing slashes from that URL and appends `/chat/completions`, sends a
non-streaming JSON request with Bearer authentication, and directly reads
`choices[0].message.content` from one complete JSON response.

For the middleware's planned `/v1/chat/completions` route, Comic Translate must be
configured with a base URL ending in `/v1`, for example
`http://127.0.0.1:<port>/v1`. Comic Translate does not append `/v1`, does not call
`GET /v1/models`, and does not validate model names. It sends two ordered messages
(`system`, then `user`) whose `content` values are arrays of content parts. When
image input is enabled, one JPEG data URI image part follows the user's text part.

The model's application-level output is a JSON object keyed as `block_0`,
`block_1`, and so on. Comic Translate greedily matches from the first `{` through
the last `}`, so some prose-wrapped block JSON works but multiple brace-delimited
objects or stray braces can make parsing fail. Pure block JSON is the intended response. Only
`choices[0].message.content` is required from the Chat Completions response by this
consumer revision.

## 4. Configuration and Base URL

### 4.1 Configuration path

**Observed.** The settings UI exposes the Custom translator and stores three
Custom credentials: `api_key`, `api_url`, and `model`. Pipeline validation rejects
the Custom translator unless all three values are nonempty. The translation
factory selects `CustomTranslation`, whose `initialize()` method reads those
values. The values are persisted in Qt `QSettings("ComicLabs",
"ComicTranslate")` only when the user enables **Save Keys**.

The setting labeled **Endpoint URL** is semantically a base URL. The source does:

```text
configured api_url -> rstrip("/") -> append "/chat/completions"
```

Comic Translate does not use an OpenAI SDK, so no SDK inserts `/v1` or otherwise
rewrites the path.

### 4.2 URL composition matrix

| Configured Custom URL | Effective request URL | Result |
| --- | --- | --- |
| `http://127.0.0.1:8000/v1` | `http://127.0.0.1:8000/v1/chat/completions` | Correct for the planned middleware route |
| `http://127.0.0.1:8000/v1/` | `http://127.0.0.1:8000/v1/chat/completions` | Correct; all trailing `/` characters are removed |
| `http://127.0.0.1:8000` | `http://127.0.0.1:8000/chat/completions` | Does not target the planned `/v1` route |
| `http://127.0.0.1:8000/v1/chat/completions` | `http://127.0.0.1:8000/v1/chat/completions/chat/completions` | Incorrect; the UI must not receive a full endpoint URL |

**Observed.** There is no host restriction or API URL validation beyond requiring
a nonempty value. Arbitrary custom URLs are accepted.

**Inferred.** A loopback URL such as `http://127.0.0.1:<port>/v1` is naturally
accepted by the existing path. This is consistent with the project's local-only
binding rule.

### 4.3 Zero-modification proof

**Result: YES.** Current Comic Translate can target the middleware without source
modifications by using its existing Custom translator settings:

1. Select **Custom** as the translator.
2. Enter `http://127.0.0.1:<port>/v1` as the Endpoint URL.
3. Enter the middleware's accepted local Bearer key as the API key.
4. Enter an accepted public model ID, including a namespaced ID when required by
   middleware policy.
5. Optionally enable image input.

This conclusion depends on the middleware accepting the contract frozen here. It
does not claim that middleware runtime behavior has already been implemented or
tested end to end.

## 5. Inference Endpoint

| Property | Confirmed behavior | Classification | Evidence |
| --- | --- | --- | --- |
| HTTP method | `POST` | Observed | CT-002 |
| Path composition | `{api_base_url}/chat/completions` | Observed | CT-001, CT-002 |
| Effective MVP path | `/v1/chat/completions` when base URL ends in `/v1` | Inferred from observed composition | CT-001, CT-002 |
| Client method | `requests.post(...)` | Observed | CT-002 |
| Request encoding | `data=json.dumps(payload)` | Observed | CT-002 |
| Timeout | 120 seconds for Custom translator | Observed | CT-001, CT-002 |
| Response mode | One completed JSON response | Observed | CT-002 |

## 6. Request Contract

### 6.1 Request Fields

| Field | Observed | When used / value | Middleware implication | Evidence |
| --- | --- | --- | --- | --- |
| `model` | Always sent | Exact nonempty Custom model text | Accept the configured public model string | CT-001, CT-002, CT-007 |
| `messages` | Always sent | Two messages in system-then-user order | Accept the content-array shapes below | CT-002 |
| `temperature` | Always sent | `1.0`; initialized internally, not a Custom UI option | Accept this field at the public boundary | CT-002, CT-003 |
| `max_completion_tokens` | Always sent | `5000`; initialized internally | Accept this field at the public boundary | CT-002, CT-003 |
| `stream` | Not observed | Omitted | Not a Comic Translate requirement | CT-002 |
| `response_format` | Not observed | Omitted | Not a Comic Translate requirement | CT-002 |
| `max_tokens` | Not observed | Omitted in translation path | Not a Comic Translate Custom translation requirement | CT-002 |
| `top_p` | Not observed | Omitted | Not a Comic Translate requirement | CT-002 |
| `seed` | Not observed | Omitted | Not a Comic Translate requirement | CT-002 |
| `extra_body` | Not observed | Omitted | Not a Comic Translate requirement | CT-002 |

`temperature` and `max_completion_tokens` are public consumer inputs. Whether a
later provider adapter forwards, maps, or filters them is outside Task 01.

### 6.2 Messages and Roles

Only `system` and `user` roles are emitted by the observed Custom translation
path. `developer` and `assistant` request roles are **not observed**.

| Order | Role | Purpose | Content shape | Required for compatibility |
| --- | --- | --- | --- | --- |
| 1 | `system` | Translation instructions, source/target language, and output rules | Array containing one text part | Yes |
| 2 | `user` | Optional context plus serialized source blocks; optionally one image | Array containing text first, then optional image | Yes |

The current path relies on this order. This task does not decide how canonical or
provider-specific adapters represent system instructions.

### 6.3 Text Content

Both messages use arrays even when the request is text-only:

```json
{
  "model": "openai:example-model",
  "messages": [
    {
      "role": "system",
      "content": [
        {"type": "text", "text": "Translate the supplied comic blocks."}
      ]
    },
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "{\"block_0\":\"Hola\"}"}
      ]
    }
  ],
  "temperature": 1.0,
  "max_completion_tokens": 5000
}
```

The example is synthetic and sanitized. Exact prompt wording is application
content, not a stable transport requirement. Content arrays and part types are
the protocol facts.

### 6.4 Multimodal / Image Content

When image input is enabled, Comic Translate adds exactly one image part after
the user text part:

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "{\"block_0\":\"Hola\"}"},
    {
      "type": "image_url",
      "image_url": {
        "url": "data:image/jpeg;base64,<synthetic-base64>"
      }
    }
  ]
}
```

| Behavior | Finding | Classification | Evidence |
| --- | --- | --- | --- |
| Enablement | Requires `supports_images` and `image_input_enabled` | Observed | CT-002, CT-003 |
| Default UI state | Image input is disabled on initial load/reload | Observed | CT-006 |
| Encoding | Image bytes are base64 encoded | Observed | CT-003 |
| Data URI | `data:{mime_type};base64,{encoded_image}` | Observed | CT-002 |
| MIME in Custom call path | `image/jpeg` because the encoder is invoked with `.jpg` | Observed | CT-002, CT-003 |
| Other encoder mappings | `.jpeg`, `.png`, and `.webp` mappings exist | Observed, but not used by this call path | CT-003 |
| Ordering | User text part first, image part second | Observed | CT-002 |
| Image count | One image part | Observed | CT-002 |
| Multiple images | No multiple-image construction found | Not used | CT-002 |
| Missing image while enabled | No null guard was established before encoding | Unconfirmed runtime outcome | CT-002, CT-003 |

Middleware normalization must preserve the image and MIME/data URI information.
Provider-specific conversion remains deferred. Images must not be silently dropped.

### 6.5 Translation Payload Semantics

Comic Translate serializes source text blocks into a JSON object with sequential
keys such as `block_0`, `block_1`, and `block_2`. The prompt asks the model to
return translated values under the corresponding keys.

The returned `message.content` is passed to a helper that greedily matches from
the first `{` through the last `}` with `r"\{[\s\S]*\}"`, parses that entire match
with `json.loads`, and assigns values back to blocks by their `block_N` keys.
Therefore:

- Valid block-keyed JSON is the application-level response contract.
- Pure JSON is preferred and should be requested from providers.
- Prose outside a single brace-delimited JSON object can be tolerated, but
  multiple objects or stray braces can make the greedy match invalid JSON.
- Missing or unknown block keys produce warnings and leave blocks unmapped rather
  than creating a transport-level failure.
- Content without a brace-delimited object logs a warning and leaves blocks
  unmapped. A brace-delimited but invalid JSON match raises `JSONDecodeError`.

The Custom, GPT, and Deepseek translators share this GPT-style request/response
implementation. Custom is the configurable middleware path. GPT has a fixed
OpenAI base URL. Deepseek has a fixed Deepseek base URL and disables images.
Claude and Gemini use provider-native contracts and do not define this public
middleware boundary. GPT OCR is a separate fixed-OpenAI OCR call and is not
configurable to target the Custom middleware.

## 7. Response Contract

### 7.1 Required Response Fields

| Response path | Read by Comic Translate? | Middleware requirement | Evidence |
| --- | --- | --- | --- |
| `choices` | Yes, directly indexed | Must be an array with at least one item | CT-002 |
| `choices[0]` | Yes | Must exist | CT-002 |
| `choices[0].message` | Yes | Must be an object | CT-002 |
| `choices[0].message.content` | Yes | Must be a string containing parseable block JSON | CT-002, CT-004 |

A minimal consumer-compatible success body is:

```json
{
  "choices": [
    {
      "message": {
        "content": "{\"block_0\":\"Hello\"}"
      }
    }
  ]
}
```

### 7.2 Optional Compatibility Fields

| Response field | Read by Comic Translate? | Consumer requirement |
| --- | --- | --- |
| `id` | No | Optional OpenAI compatibility metadata |
| `object` | No | Optional OpenAI compatibility metadata |
| `created` | No | Optional OpenAI compatibility metadata |
| `model` | No | Optional OpenAI compatibility metadata |
| `choices[0].index` | No | Optional OpenAI compatibility metadata |
| `choices[0].message.role` | No | Optional OpenAI compatibility metadata |
| `choices[0].finish_reason` | No | Optional OpenAI compatibility metadata |
| `usage` | No | Optional OpenAI compatibility metadata |

The middleware MVP may require a fuller OpenAI-compatible response for product
reasons. Those fields must not be mislabeled as requirements imposed by Comic
Translate 2.8.9.

### 7.3 Application-Level Content Expectations

HTTP success and a syntactically valid Chat Completions envelope are insufficient
if `message.content` lacks the expected block JSON. A usable content string has a
JSON object whose keys correspond to submitted blocks:

```json
{"block_0":"Hello","block_1":"Goodbye"}
```

The greedy extraction may accept prose around one JSON object, but multiple
objects or stray braces can fail parsing. The middleware should preserve provider
text accurately; later provider contracts own prompting and malformed-output policy.

## 8. Model Selection and Discovery

**Observed.** The Custom model is entered manually in an editable text field and
is sent unchanged. Pipeline validation checks only that it is nonempty. Comic
Translate does not parse namespaces or validate availability.

**Not used.** No `GET /v1/models`, OpenAI model-list client call, automatic model
discovery, model cache, or Custom model-catalog validation occurs in the inspected
path.

Namespaced public IDs such as `openai:gpt-4.1` or `google:example-model` can be
entered as ordinary text. Their required format and routing meaning are middleware
architecture decisions, not Comic Translate behavior.

The middleware MVP still plans `GET /v1/models`; that is a **middleware product
feature**, not a confirmed Comic Translate protocol requirement.

## 9. Public Authentication Expectations

Comic Translate sends these headers on every Custom translation request:

```text
Content-Type: application/json
Authorization: Bearer <configured Custom API key>
```

The Custom configuration requires a nonempty API key. The local public boundary
must therefore accept a user-configured Bearer value. Whether the middleware
validates that value and how it is configured are later security/configuration
decisions.

The Bearer value is a public-boundary credential. It must not enter canonical
provider requests, logs, error details, fixtures, model metadata, or architecture
examples. Comic Translate can persist it in QSettings when **Save Keys** is
enabled; that consumer-side behavior does not authorize middleware access to any
other credentials.

No Comic Translate OAuth behavior is part of this contract. Upstream provider
OAuth and API-key handling remain private middleware concerns.

## 10. Streaming Semantics

### Comic Translate to middleware

- `stream` is omitted from the request.
- Comic Translate calls `response.json()` after the POST completes.
- It expects one completed Chat Completions JSON object.
- It does not consume SSE, chunks, deltas, or `[DONE]` markers.

Therefore public streaming is **not required** for this consumer revision, and a
streaming-only response would be incompatible.

### Middleware to upstream provider

Future providers may be required to stream internally and collect the result, such
as the project's planned OpenAI SIWC path. That private provider behavior is
separate from the complete public response required here and is not frozen by
Task 01.

## 11. Error and Failure Behavior

The Custom translator calls `raise_for_status()`. Its broad
`requests.exceptions.RequestException` handler converts HTTP and transport errors
to `RuntimeError`, including parsed response JSON or status details when available.
The immediate path does not preserve a typed HTTP status. Later batch reporting
uses error-message string heuristics, so this classification is fragile.

| Failure | Observed consumer behavior | Classification |
| --- | --- | --- |
| HTTP 400 | `HTTPError` -> generic `RuntimeError` | Generic exception path |
| HTTP 401 | `HTTPError` -> generic `RuntimeError`; later UI may classify string | Generic exception path |
| HTTP 403 | Same; later UI may classify string | Generic exception path |
| HTTP 404 | `HTTPError` -> generic `RuntimeError` | Generic exception path |
| HTTP 429 | Same; later UI may classify string | Generic exception path |
| HTTP 500 | `HTTPError` -> generic `RuntimeError` | Generic exception path |
| HTTP 502 | Same; later UI may classify string | Generic exception path |
| HTTP 503 | Same; later UI may classify string | Generic exception path |
| HTTP 504 | Same; later UI may classify string | Generic exception path |
| Timeout | `RequestException` -> generic `RuntimeError` | Generic exception path |
| Connection failure | `RequestException` -> generic `RuntimeError` | Generic exception path |
| Malformed HTTP-success JSON | With the declared Requests dependency, `JSONDecodeError` is a `RequestException` and is converted to `RuntimeError`; the exact installed Requests version is not pinned | Generic exception path |
| Missing `choices`/`message`/`content` | Key/type failure propagates | Generic exception path |
| Empty content string | JSON extraction warns; blocks remain unmapped | Explicit downstream tolerance |
| Brace-delimited invalid block JSON | Uncaught `json.loads` `JSONDecodeError` propagates | Generic exception path |

The middleware should return an HTTP error instead of an invalid success envelope.
The exact OpenAI-compatible error schema and complete middleware status mapping are
Task 06 concerns. Safe actionable text is useful to the current client, but raw
upstream errors and secrets must never be exposed.

## 12. Compatibility Matrix

`MUST` means required for interoperability with the pinned Comic Translate Custom
translator. `SHOULD` improves reliable application behavior. `OPTIONAL` is useful
OpenAI compatibility or product behavior not consumed here. `NOT REQUIRED` means
the consumer does not require it; it may still be an MVP feature.

| ID | Comic Translate behavior | Evidence | Middleware responsibility | Classification |
| --- | --- | --- | --- | --- |
| CT-REQ-001 | Appends `/chat/completions` to configured base URL | CT-001, CT-002 | Expose `POST /v1/chat/completions` and document `/v1` as the configured base | MUST |
| CT-REQ-002 | Sends JSON request body | CT-002 | Accept JSON at the inference route | MUST |
| CT-REQ-003 | Sends `Content-Type: application/json` | CT-002 | Accept the declared media type | MUST |
| CT-REQ-004 | Sends `Authorization: Bearer <key>` | CT-002, CT-007 | Accept the configured Bearer credential without leaking it | MUST |
| CT-REQ-005 | Does not call model listing | CT-001, CT-002 | Do not require a prior `GET /v1/models` call | MUST |
| CT-REQ-006 | Omits `stream` and parses one JSON response | CT-002 | Return one complete JSON response | MUST |
| CT-REQ-007 | Sends manually entered nonempty `model` | CT-001, CT-002, CT-007 | Accept and route the configured public model ID | MUST |
| CT-REQ-008 | Sends ordered system and user messages | CT-002 | Accept both messages in order | MUST |
| CT-REQ-009 | Always sends `temperature: 1.0` | CT-002, CT-003 | Accept the field at the compatibility boundary | MUST |
| CT-REQ-010 | Always sends `max_completion_tokens: 5000` | CT-002, CT-003 | Accept the field at the compatibility boundary | MUST |
| CT-REQ-011 | Uses system content as a text-part array | CT-002 | Preserve/normalize this content-array shape | MUST |
| CT-REQ-012 | Uses user content as a text-part array | CT-002 | Preserve/normalize this content-array shape | MUST |
| CT-REQ-013 | Adds one JPEG data URI image after user text when enabled | CT-002, CT-003 | Preserve the image, MIME type, base64 data, and ordering | MUST |
| CT-REQ-014 | Directly reads `choices[0].message.content` | CT-002 | Return non-null string content in that path | MUST |
| CT-REQ-015 | Parses `block_N` JSON from returned content | CT-004 | Produce content usable as block-keyed JSON | MUST |
| CT-REQ-016 | Greedily parses from the first `{` through the last `}`, allowing some prose but risking failure with extra braces/objects | CT-004 | Prefer one pure block JSON object; exact preservation remains provider policy | SHOULD |
| CT-REQ-017 | Does not read standard response metadata | CT-002 | Metadata may be added for broader compatibility | OPTIONAL |
| CT-REQ-018 | Accepts arbitrary manual model text | CT-001, CT-007 | Permit valid middleware namespaced IDs through manual entry | MUST |
| CT-REQ-019 | Does not discover models | CT-001, CT-002 | `GET /v1/models` may exist as an MVP feature | NOT REQUIRED |
| CT-REQ-020 | Converts HTTP/transport failures to generic errors | CT-002 | Return real HTTP failures with safe actionable details | MUST |
| CT-REQ-021 | Never consumes response streams | CT-002 | Public SSE/chunk support is unnecessary for this client | NOT REQUIRED |
| CT-REQ-022 | Never reads `id`, `object`, `created`, `model`, `finish_reason`, or `usage` | CT-002 | Include only if required by broader OpenAI compatibility | OPTIONAL |

## 13. Confirmed Middleware Requirements

The confirmed consumer-facing minimum is:

1. A `POST /v1/chat/completions` route reachable when Comic Translate is
   configured with a base URL ending in `/v1`.
2. JSON parsing for `model`, ordered `messages`, `temperature`, and
   `max_completion_tokens` in the observed shapes.
3. Support for content-part arrays on both system and user messages.
4. Preservation of the optional one-image JPEG data URI part.
5. Acceptance of Bearer authentication supplied by the Custom API-key setting.
6. A complete non-streaming JSON response with a string at
   `choices[0].message.content`.
7. Application-level content containing parseable `block_N` translation JSON.
8. Failure responses that use HTTP errors rather than malformed success objects.
9. No dependency on consumer-side model discovery or model-name validation.

These requirements do not authorize implementation in Task 01.

## 14. Non-Requirements / Not Observed

The following are not requirements imposed by Comic Translate 2.8.9's Custom
translation path:

- The OpenAI Python SDK or any SDK-specific URL behavior.
- `GET /v1/models`, model discovery, model caching, or automatic validation.
- A `stream` request field, public response streaming, SSE, or chunk handling.
- `response_format`, `max_tokens`, `top_p`, `seed`, `extra_body`, tools, or tool
  choice.
- Request roles `developer` or `assistant`.
- Multiple images in one request.
- Response metadata fields `id`, `object`, `created`, `model`, `finish_reason`,
  or `usage`.
- Namespaced model syntax as a consumer rule. Namespacing remains a middleware
  architecture requirement and works through manual entry.
- Any public-boundary OAuth flow.
- `/health` or administrative routes.

## 15. Unconfirmed Questions

| Question | Status | Owner / revisit trigger |
| --- | --- | --- |
| What occurs if image input is enabled but no image object reaches the encoder? | Unconfirmed; no null guard established | Compatibility tests when image request normalization is implemented |
| How should malformed provider block JSON map to middleware errors versus successful text? | Intentionally deferred | Task 03/06 contract and error design |
| Which providers accept system content arrays and data URIs without conversion? | Provider-specific and unconfirmed here | Provider implementation tasks |
| What exact local Bearer-key validation policy should be used? | Not a consumer fact | Task 07 security contract |
| What full OpenAI-compatible success/error metadata should the MVP return? | Not consumed by Comic Translate | Task 03 and Task 06 |

No unconfirmed question blocks the external Custom translator protocol baseline.

## 16. Architecture Implications

- The compatibility layer must support content arrays; assuming plain string
  message content would reject the actual consumer.
- Image data must survive normalization unchanged in meaning. Provider adapters,
  not the public compatibility boundary, own later image conversion.
- The configured model text can carry the project's required provider namespace.
  Comic Translate does not remove or validate it.
- `/v1/models` remains an approved MVP product feature, but implementation must
  not require Comic Translate to call it before inference.
- Public behavior is complete-response/non-streaming. Internal provider streaming
  must be collected before the public response when a provider requires it.
- Public Bearer credentials and private provider credentials are separate security
  boundaries. Neither belongs in canonical inference content.
- Error architecture should preserve an actionable sanitized message because the
  consumer exposes generic request failures and later applies string heuristics.
- Direct source evidence supersedes prior assumptions that Comic Translate uses an
  OpenAI SDK or sends additional OpenAI fields.

### Deferred decision boundaries

Task 01 does not freeze `CanonicalRequest`, `InferenceResult`, `BaseProvider`, the
provider registry, provider payload mappings, OAuth internals, retry policy,
Responses API conversion, or Antigravity process behavior. Those remain owned by
their later Wave tasks and must consume, not redefine, this external contract.

## 17. Evidence Index

All links are pinned to
`8977b91a4f7a40c3917c5a268e9e7d78e1d818da`.

| ID | Source / symbol | Finding |
| --- | --- | --- |
| CT-001 | [`modules/translation/llm/custom.py`, `CustomTranslation.initialize`, L11-L31](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/llm/custom.py#L11-L31) | Reads Custom key/model/URL, strips trailing slashes, sets 120-second timeout |
| CT-002 | [`modules/translation/llm/gpt.py`, `_perform_translation` / `_make_api_request`, L37-L116](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/llm/gpt.py#L37-L116) | Headers, messages, image part, request fields, endpoint append, POST, complete response access, request errors |
| CT-003 | [`modules/translation/llm/base.py`, `BaseLLMTranslation`, L14-L105](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/llm/base.py#L14-L105) | Image flag, fixed temperature/token values, translation flow, base64/MIME encoding |
| CT-004 | [`modules/utils/translator_utils.py`, `get_raw_text` / `set_texts_from_json`, L25-L59](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/utils/translator_utils.py#L25-L59) | `block_N` input/output semantics, JSON extraction, missing-key behavior |
| CT-005 | [`modules/translation/factory.py`, `TranslationFactory`, L29-L94](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/factory.py#L29-L94) | Custom translator selection and engine creation/cache |
| CT-006 | [`app/ui/settings/settings_page.py`, settings access/persistence, L141-L185](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/app/ui/settings/settings_page.py#L141-L185), [L313-L328](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/app/ui/settings/settings_page.py#L313-L328), [L403-L452](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/app/ui/settings/settings_page.py#L403-L452) | Image setting, QSettings persistence/reload, Custom credentials UI behavior |
| CT-007 | [`modules/utils/pipeline_config.py`, `validate_translator`, L57-L84](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/utils/pipeline_config.py#L57-L84) | Requires nonempty Custom `api_key`, `api_url`, and `model` |
| CT-008 | [`modules/translation/processor.py`, `Translator.translate`, L86-L114](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/processor.py#L86-L114) | Actual translation invocation and image forwarding |
| CT-009 | [`modules/translation/llm/deepseek.py`, `DeepseekTranslation`, L7-L31](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/modules/translation/llm/deepseek.py#L7-L31) | Shared GPT-style implementation with fixed Deepseek base and images disabled |
| CT-010 | [`requirements.txt`, L1-L21](https://github.com/ogkalu2/comic-translate/blob/8977b91a4f7a40c3917c5a268e9e7d78e1d818da/requirements.txt#L1-L21) | `requests` dependency; no OpenAI Python SDK dependency |

### Evidence quality

The central contract is source-derived from the exact default-branch revision.
The decisive configuration-to-response path was independently followed in the
pinned tree. No dedicated upstream tests for this Custom protocol path were found,
and no runtime request was sent; runtime integration remains a later validation
gate.

## 18. Task 01 Freeze

### Frozen consumer facts

- Custom base URL composition and `/v1` configuration requirement.
- Direct `POST .../chat/completions` transport and Bearer header.
- Observed request fields, role order, and content-part array shapes.
- Optional single JPEG data URI image behavior.
- Complete non-streaming response consumption.
- Direct dependency on `choices[0].message.content` and block-keyed JSON semantics.
- Manual model entry and absence of model discovery.
- Generic consumer error handling and known evidence gaps.
- Zero Comic Translate source modifications are required.

### Review state

Primary Architecture & Security analysis found no blocking protocol issue and
identified the direct-HTTP/custom-base behavior as an important correction to
earlier assumptions. A fresh Architecture & Security review independently checked
the pinned source and returned **APPROVE** after two required parser-semantics
corrections were applied and verified. The final review reported no blocking,
important, optional, architecture, or security findings. The user explicitly
approved Task 01 and the two-commit integration strategy on 2026-10-12. Task 01
is complete; Task 02 remains unstarted.

### Future contract-test candidates

No tests are created by Task 01. Later contract work should cover:

- Base URLs ending in `/v1` and `/v1/`, plus rejection/documentation of a full
  `/v1/chat/completions` value.
- Text-only system/user content arrays.
- One `data:image/jpeg;base64,...` part after user text.
- Required request fields and Bearer handling without secret logging.
- Complete response with `choices[0].message.content` containing valid block JSON.
- Single-object prose-wrapped, multi-object/stray-brace, invalid JSON, empty, and
  missing-choice content behavior.
- Manual namespaced model entry without a prior model-list call.
- Public complete-response behavior separated from private provider streaming.