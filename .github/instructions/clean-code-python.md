# Clean Code in Python — Comic Translate Middleware

This document defines the **coding standards, naming conventions, architectural principles, and quality rules** for the `comic-translate-middleware` project.  
It is based on the principles from [Clean Code in Python (TestDriven.io)](https://testdriven.io/blog/clean-code-python/) and adapted to the specific needs of this FastAPI-based OpenAI-compatible middleware.

All contributors must follow these guidelines. Code that violates them should be refactored before merge.

---

## 1. Core Principles

We follow these foundational principles:

| Principle | Meaning in this project |
|-----------|-------------------------|
| **DRY**   | Don't Repeat Yourself. Extract shared request/response normalization, OAuth token handling, and error formatting into reusable modules. Avoid premature abstraction. |
| **KISS**  | Keep It Simple, Stupid. Prefer clear, straightforward code over clever solutions. The middleware must remain easy to reason about. |
| **SoC**   | Separation of Concerns. Keep the OpenAI compatibility layer, provider adapters, auth, and configuration strictly separated. |
| **SOLID** | Especially Single Responsibility and Dependency Inversion. Each provider implements a clear interface; the router depends on abstractions, not concrete providers. |

Additional guiding values from the Zen of Python that we prioritize:

- Explicit is better than implicit.
- Simple is better than complex.
- Readability counts.
- Errors should never pass silently.
- There should be one — and preferably only one — obvious way to do it.

---

## 2. Project Structure & Modularity

The project structure must remain close to the architecture defined in the plan:

```text
comic-translate-middleware/
├── app/
│   ├── main.py
│   ├── api/                  # HTTP endpoints only
│   ├── compatibility/        # Chat Completions ↔ Canonical ↔ Responses adapters
│   ├── providers/            # Provider implementations (OpenAI, Antigravity, etc.)
│   ├── auth/                 # OAuth, PKCE, credential store
│   ├── models/               # Pydantic models and domain objects
│   └── config/               # Settings and configuration
├── tests/
│   ├── contract/
│   ├── integration/
│   ├── providers/
│   └── fixtures/
├── pyproject.toml
├── README.md
└── .env.example
```

### Rules

- Each package has a single, clear responsibility.
- Do **not** put business logic inside `api/` endpoints. Endpoints should only validate input, call services/providers, and return responses.
- Provider-specific code lives only inside `providers/`.
- Compatibility/normalization logic lives only inside `compatibility/`.
- Prefer small modules (< 250–300 lines). Split when a file starts mixing concerns.
- Avoid circular imports. Use dependency injection or late imports when necessary.

---

## 3. Naming Conventions (PEP 8 + Project Rules)

Follow PEP 8 strictly, with these project-specific conventions:

| Element              | Convention                  | Example |
|----------------------|-----------------------------|---------|
| Modules / packages   | `snake_case`                | `openai_siwc.py`, `antigravity_cli.py` |
| Classes              | `PascalCase`                | `OpenAISiwcProvider`, `CanonicalRequest` |
| Functions / methods  | `snake_case` + verb         | `normalize_chat_request()`, `refresh_token()` |
| Variables            | `snake_case` + noun         | `access_token`, `model_catalog` |
| Constants            | `UPPER_SNAKE_CASE`          | `DEFAULT_TIMEOUT_SECONDS`, `MAX_COMPLETION_TOKENS` |
| Private              | leading underscore          | `_build_pkce_challenge()` |
| Type aliases         | `PascalCase`                | `ProviderName`, `ModelId` |

### Naming Best Practices

1. **Use nouns for variables, verbs for functions.**
2. **Use descriptive, intention-revealing names.**  
   Prefer `openai_access_token` over `token` or `t`.
3. **Use pronounceable names.**  
   Avoid abbreviations like `genyyyymmdd` or `cre_tmstp`.
4. **Avoid ambiguous abbreviations.**  
   Prefer `endpoint_url` over `ep_url`, `max_completion_tokens` over `max_tok`.
5. **Use consistent vocabulary.**  
   Always use `provider`, `model`, `request`, `response`, `token`. Do not mix `client` / `customer`, `fetch` / `get`, `auth` / `login` inconsistently.
6. **No magic numbers or strings.**  
   Extract them into named constants:
   ```python
   DEFAULT_PORT = 8420
   LOCAL_BIND_HOST = "127.0.0.1"
   OPENAI_RESPONSES_ENDPOINT = "https://api.openai.com/v1/responses"
   ```
7. **Do not add redundant context.**  
   Inside class `OpenAISiwcProvider`, prefer `self.access_token` over `self.openai_access_token`.
8. **Model IDs use namespaces.**  
   Always use the form `provider:model` (e.g. `openai:gpt-4o`, `google:gemini-2.0-flash`). Never use bare model names in public contracts.

---

## 4. Functions & Methods

### Rules

1. **Functions should do one thing only.**  
   If the name needs “and”, split it.
2. **Keep functions short and focused.**  
   Prefer < 30–40 lines. A function should be understandable in under a minute.
3. **Minimize number of arguments.**  
   Ideal: 0–2. More than 3 → consider a Pydantic model or dataclass.
4. **Never use flag arguments.**  
   Instead of `def process(data, is_stream: bool)`, create two clear functions or use polymorphism.
5. **Avoid side effects when possible.**  
   Pure functions are preferred for normalization and conversion logic. Side effects (network, disk, keyring) must be explicit and isolated.
6. **Use verbs that accurately describe the action:**  
   `normalize_`, `convert_`, `validate_`, `refresh_`, `collect_`, `build_`, `parse_`.

### Good Example

```python
def normalize_chat_completion_request(
    raw: ChatCompletionRequest,
) -> CanonicalRequest:
    """Convert an OpenAI Chat Completions request into the internal CanonicalRequest."""
    ...
```

### Bad Example

```python
def handle(data, stream=False, with_images=True, use_oauth=False):
    # does too many things, flag parameters, unclear name
    ...
```

---

## 5. Classes & Interfaces

### Provider Interface

Every provider must implement a clear abstract base (or Protocol):

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

### Rules

- Follow **Single Responsibility Principle**: one class = one reason to change.
- Prefer composition over deep inheritance.
- Use Pydantic models for all external and internal data contracts (`ChatCompletionRequest`, `CanonicalRequest`, `InferenceResult`, etc.).
- Keep classes under ~200–250 lines when possible.
- Make dependencies explicit via constructor injection.

---

## 6. Comments & Docstrings

1. **Prefer readable code over comments.**  
   If you need a comment to explain *what* the code does, rewrite the code.
2. **Comments should explain *why*, not *what*.**
3. **No noise comments.**  
   Never write comments like `# increment counter` or `# return result`.
4. **Never leave commented-out code.**  
   Delete it. Git history exists for a reason.
5. **Use proper docstrings** (Google or NumPy style is acceptable; be consistent):

```python
async def refresh_access_token(self) -> str:
    """Refresh the OAuth access token using the stored refresh token.

    Raises:
        AuthenticationError: If the refresh token is invalid or expired.
        NetworkError: If the token endpoint is unreachable.
    """
```

6. **Do not comment bad code — rewrite it.**

---

## 7. Error Handling

- Never silence exceptions unless explicitly intended and documented.
- Use custom exception hierarchy rooted in a project base exception.
- Translate provider-specific errors into a consistent internal error model before returning OpenAI-compatible responses.
- Always include a `request_id` in error responses when available.
- Log errors with structured context, but **never log tokens, refresh tokens, or API keys**.

---

## 8. Type Hints & Static Analysis

- All public functions, methods, and class attributes must have type hints.
- Use `from __future__ import annotations` when helpful.
- Prefer modern typing (`list[...]`, `dict[...]`, `|` union syntax).
- Run `mypy` (or `pyright`) in CI. Aim for strict mode on new code.
- Use Pydantic for runtime validation of external payloads.

---

## 9. Formatting & Tooling

We enforce consistency via automated tools:

| Tool          | Purpose                          |
|---------------|----------------------------------|
| **Ruff**      | Linting + import sorting + formatting (preferred over Black + isort + flake8) |
| **mypy / pyright** | Static type checking            |
| **pytest**    | Testing                          |

- Line length: 88 or 100 (project decision; be consistent).
- Indentation: 4 spaces.
- Always run the formatter and linter before committing.

---

## 10. Testing Guidelines

- Write tests for every public behavior.
- Prefer **contract tests** that verify the OpenAI Chat Completions interface remains stable.
- Provider adapters should have both unit tests (with mocks) and integration tests (marked and optional).
- Use fixtures for common payloads (multimodal messages, large page JSON, etc.).
- Test names should describe the behavior:  
  `test_normalize_request_with_image_url_preserves_base64_data`

---

## 11. Security & Sensitive Data (Project-Specific)

These rules are **non-negotiable**:

- Never read `~/.codex/auth.json` or any other private auth files of third-party tools.
- Never extract tokens from the Antigravity keyring manually.
- Never call private/undocumented backend APIs.
- Never log `access_token`, `refresh_token`, or `id_token`.
- Never put OAuth tokens in query strings or URLs.
- Bind the server to `127.0.0.1` by default.
- Store credentials using the system keyring (`keyring` library) on Windows/macOS/Linux.
- Optional local API key (`ctm_local_...`) must be treated as a secret.

---

## 12. Architectural Rules Specific to This Project

1. **Zero modification to Comic Translate.**  
   The middleware must adapt to Comic Translate, never the other way around.

2. **CanonicalRequest is the single source of truth** inside the system.  
   All providers consume `CanonicalRequest` and produce `InferenceResult`.

3. **Compatibility layer is provider-agnostic.**  
   `compatibility/` must not import any concrete provider.

4. **Provider router decides the concrete implementation** based on the model namespace (`openai:...`, `google:...`).

5. **Streaming and non-streaming paths must be clearly separated** and well-tested.  
   Comic Translate currently expects a full Chat Completions response; the adapter must collect the stream when necessary.

6. **Model discovery is the source of truth**, not hard-coded plan names (`Plus`, `Pro`, `Ultra`).

---

## 13. Code Review Checklist

Before approving a PR, verify:

- [ ] Names are clear, consistent, and intention-revealing
- [ ] Functions are short and do one thing
- [ ] No magic numbers / strings
- [ ] No commented-out code
- [ ] Type hints present on public APIs
- [ ] Errors are handled explicitly
- [ ] No sensitive data in logs
- [ ] New code has corresponding tests
- [ ] Architecture boundaries (api / compatibility / providers / auth) are respected
- [ ] Ruff and type checker pass

---

## 14. Summary

Write code that is:

- **Easy to read** (especially months later)
- **Easy to change** (new providers, new OpenAI API shapes)
- **Hard to misuse** (clear interfaces, good validation)
- **Safe by default** (no token leakage, localhost binding)

When in doubt, prefer the simpler, more explicit solution.

> "Writing code is easy, but writing good, clean code is hard."  
> — Clean Code in Python (TestDriven.io)

This document is part of the project’s mandatory instruction set. Update it when the architecture evolves.