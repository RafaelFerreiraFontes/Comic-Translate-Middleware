# FastAPI Reference — Comic Translate Middleware

This document is the **project-specific FastAPI reference and coding standard** for `comic-translate-middleware`.  
It is based on the official [FastAPI Reference](https://fastapi.tiangolo.com/reference/) and tailored to the architecture, security constraints, and endpoints of this project.

All FastAPI code in the project must follow these conventions.

---

## 1. Core Imports

Prefer the modern, explicit style:

```python
from typing import Annotated

from fastapi import (
    APIRouter,
    Depends,
    FastAPI,
    HTTPException,
    Request,
    Response,
    status,
)
from fastapi.responses import JSONResponse, StreamingResponse
from pydantic import BaseModel, Field
```

Avoid importing rarely used symbols at module level unless they are actually used.

---

## 2. Application Instantiation (`FastAPI`)

Create the application in `app/main.py` (or equivalent entrypoint).

### Recommended configuration for this project

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load credentials, warm providers, etc.
    yield
    # Shutdown: close HTTP clients, clean up resources

app = FastAPI(
    title="Comic Translate Middleware",
    description="OpenAI-compatible local middleware for Comic Translate",
    version="0.1.0",
    docs_url="/docs",          # keep for local development
    redoc_url="/redoc",
    openapi_url="/openapi.json",
    lifespan=lifespan,
    # Security defaults for a local-only service
    # (bind to 127.0.0.1 at the uvicorn level)
)
```

### Important parameters (from official reference)

| Parameter | Project recommendation | Notes |
|-----------|------------------------|-------|
| `title`, `description`, `version` | Always set | Appears in `/docs` |
| `docs_url` / `redoc_url` | Keep enabled in development | Can be disabled in production packaging if desired |
| `openapi_url` | Keep `/openapi.json` | Useful for contract testing |
| `lifespan` | Prefer over `on_startup` / `on_shutdown` | Modern and cleaner |
| `dependencies` | Use sparingly at app level | Prefer router-level or path-level dependencies |
| `default_response_class` | `JSONResponse` (default) | Sufficient for Chat Completions responses |
| `strict_content_type` | Leave default (`True`) | Important for local services |

**Never** bind the server to `0.0.0.0` in the MVP. Always run with `--host 127.0.0.1`.

---

## 3. APIRouter — Organizing Endpoints

Use `APIRouter` to keep the codebase modular, matching the planned structure:

```text
app/api/
├── health.py
├── models.py
├── chat_completions.py
└── auth.py
```

### Recommended pattern

```python
# app/api/chat_completions.py
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/v1",
    tags=["chat"],
    # dependencies=[Depends(verify_local_token)],  # optional global for this router
)

@router.post("/chat/completions")
async def create_chat_completion(...):
    ...
```

Then include in the main app:

```python
from app.api import health, models, chat_completions, auth

app.include_router(health.router)
app.include_router(models.router)
app.include_router(chat_completions.router)
app.include_router(auth.router)          # auth endpoints stay under /auth
```

### Key `APIRouter` parameters

| Parameter | Recommendation |
|-----------|----------------|
| `prefix` | Use for versioning (`/v1`) or grouping (`/auth`) |
| `tags` | Always set — improves `/docs` organization |
| `dependencies` | Use for common auth / validation on a whole group |
| `responses` | Declare common error responses at router level when useful |
| `include_in_schema` | Keep `True` for public endpoints; can be `False` for internal ones |

---

## 4. Path Operation Functions

### Style rules

1. Prefer `async def` for all path operations (the project is I/O-bound).
2. Keep path operation functions thin — they should only:
   - Validate / extract input
   - Call a service / provider / compatibility layer
   - Return a response or raise an exception
3. Never put business logic (provider calls, OAuth, stream collection) directly inside the path operation.

### Example (good)

```python
@router.post(
    "/chat/completions",
    response_model=ChatCompletionResponse,   # or return dict / JSONResponse
    status_code=status.HTTP_200_OK,
)
async def create_chat_completion(
    body: ChatCompletionRequest,
    # request: Request,                      # only if you really need the raw request
    # local_token: Annotated[str, Depends(verify_local_token)],
) -> ChatCompletionResponse:
    canonical = normalize_chat_request(body)
    result = await provider_router.infer(canonical)
    return build_chat_completion_response(result)
```

### Status codes

Always use the `status` module:

```python
from fastapi import status

raise HTTPException(
    status_code=status.HTTP_401_UNAUTHORIZED,
    detail="Invalid or missing local API key",
)
```

Common codes used in this project:

| Code | Meaning |
|------|---------|
| 200 | Successful Chat Completions / models list |
| 400 | Invalid request / unsupported parameter |
| 401 | Missing or invalid local token / OAuth not authenticated |
| 403 | Authenticated but not allowed |
| 404 | Model not found |
| 429 | Rate limit / subscription limit from upstream |
| 500 | Unexpected internal error (should be rare) |
| 502 / 503 | Upstream provider failure |

---

## 5. Dependencies (`Depends` and `Security`)

Use the modern `Annotated` style:

```python
from typing import Annotated
from fastapi import Depends

async def get_provider_router() -> ProviderRouter:
    ...

ProviderRouterDep = Annotated[ProviderRouter, Depends(get_provider_router)]

@router.post("/chat/completions")
async def create_chat_completion(
    body: ChatCompletionRequest,
    router: ProviderRouterDep,
):
    ...
```

### Rules for this project

- Prefer **function-scoped** dependencies.
- Use `use_cache=True` (default) unless you have a clear reason to disable it.
- For local authentication (the optional `ctm_local_...` token), create a dedicated dependency.
- Do **not** put heavy logic (network calls, token refresh) inside a dependency that is declared many times — keep dependencies lightweight or use a proper service layer.
- `Security()` is available if we later expose OAuth2 scopes in OpenAPI, but for the local middleware the simple `Depends` + header check is usually sufficient.

### Dependency with yield (resource management)

```python
async def get_http_client():
    client = httpx.AsyncClient(timeout=60.0)
    try:
        yield client
    finally:
        await client.aclose()
```

---

## 6. Request & Response Handling

### Request body

Always use **Pydantic models** (defined in `app/models/` or `app/compatibility/`):

```python
class ChatCompletionRequest(BaseModel):
    model: str
    messages: list[dict]          # or a more precise Message model
    temperature: float | None = 1.0
    max_completion_tokens: int | None = None
    # ... other fields Comic Translate may send
```

Do **not** read the raw body with `await request.json()` unless you have a very specific reason (e.g. streaming request body or non-JSON content).

### Returning responses

Preferred order:

1. Return a Pydantic model → FastAPI serializes it automatically.
2. Return a plain `dict` that matches the OpenAI Chat Completions shape.
3. Return an explicit `JSONResponse` when you need custom headers/status.
4. Use `StreamingResponse` only if we later decide to support true streaming to Comic Translate (currently the middleware collects the upstream stream).

### Accessing the raw `Request`

Only when necessary (logging client IP, custom headers, etc.):

```python
from fastapi import Request

@router.post("/chat/completions")
async def create_chat_completion(body: ChatCompletionRequest, request: Request):
    client_host = request.client.host if request.client else None
    ...
```

---

## 7. Exception Handling

### Raising client errors

```python
from fastapi import HTTPException, status

raise HTTPException(
    status_code=status.HTTP_400_BAD_REQUEST,
    detail={
        "error": {
            "message": "Model not supported by the current provider",
            "type": "invalid_request_error",
            "code": "model_not_found",
        }
    },
)
```

For OpenAI-compatible error shape, keep the `detail` structure consistent with what Comic Translate (and OpenAI clients) expect.

### Custom exception handlers

Register them on the app when needed:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(ProviderError)
async def provider_error_handler(request: Request, exc: ProviderError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"message": str(exc), "type": "provider_error"}},
    )
```

Never let unexpected exceptions leak stack traces to the client in production packaging.

---

## 8. Models & OpenAPI Schema

- All request/response models live under `app/models/` or `app/compatibility/`.
- Use clear `Field(..., description=...)` and `examples` so that `/docs` stays useful.
- Keep the public models (Chat Completions) stable — they are part of the contract with Comic Translate.
- Internal models (`CanonicalRequest`, `InferenceResult`) should **not** appear in the OpenAPI schema.

Example:

```python
class ModelInfo(BaseModel):
    id: str = Field(..., examples=["openai:gpt-4o", "google:gemini-2.0-flash"])
    object: str = "model"
    owned_by: str | None = None
```

---

## 9. Authentication Endpoints (`/auth/*`)

These endpoints are administrative and local-only.

Recommended structure:

```python
auth_router = APIRouter(
    prefix="/auth",
    tags=["auth"],
)

@auth_router.post("/openai/login")
async def openai_login(...): ...

@auth_router.get("/openai/callback")
async def openai_callback(...): ...

@auth_router.get("/openai/status")
async def openai_status(...): ...

@auth_router.post("/openai/logout")
async def openai_logout(...): ...
```

Rules:

- Never expose these endpoints beyond `127.0.0.1`.
- Prefer short-lived state / PKCE for the OAuth flow.
- Do not log tokens or authorization codes.

---

## 10. Testing with FastAPI

Use `TestClient` (or `httpx.AsyncClient` with `ASGITransport`) for contract and integration tests:

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
```

For async tests prefer `pytest-asyncio` + `httpx.AsyncClient`.

Contract tests must verify that:

- `POST /v1/chat/completions` accepts the payload Comic Translate sends.
- The response shape matches what Comic Translate expects (`choices[0].message.content`).
- Multimodal messages (`image_url` with `data:image/...;base64,...`) are accepted.

---

## 11. Project-Specific FastAPI Rules (Mandatory)

1. **Local-only by default**  
   Server must bind to `127.0.0.1`. Document this clearly in the README and packaging.

2. **Thin path operations**  
   All business logic lives in `compatibility/`, `providers/`, or `auth/`. Path operations only orchestrate.

3. **Consistent error shape**  
   Prefer OpenAI-compatible error objects when responding to `/v1/*` endpoints.

4. **No sensitive data in responses or logs**  
   Never return or log `access_token`, `refresh_token`, or the local `ctm_local_...` key.

5. **Pydantic everywhere for external contracts**  
   Request bodies and (when practical) responses should be typed models.

6. **Lifespan over deprecated events**  
   Use the `lifespan` context manager for startup/shutdown.

7. **Router separation**  
   Keep `/v1/*` (public OpenAI-compatible) and `/auth/*` (administrative) in separate routers.

8. **Type hints and Annotated**  
   Always use `Annotated[..., Depends(...)]` for dependencies.

---

## 12. Quick Reference — Most Used Symbols

| Symbol | Import | Typical use in this project |
|--------|--------|-----------------------------|
| `FastAPI` | `from fastapi import FastAPI` | Application entrypoint |
| `APIRouter` | `from fastapi import APIRouter` | Group endpoints (`/v1`, `/auth`) |
| `Depends` | `from fastapi import Depends` | Dependency injection |
| `HTTPException` | `from fastapi import HTTPException` | Client errors |
| `status` | `from fastapi import status` | HTTP status codes |
| `Request` | `from fastapi import Request` | Rarely needed raw access |
| `Response` / `JSONResponse` | `from fastapi.responses import ...` | Custom responses |
| `Annotated` | `from typing import Annotated` | Modern dependency style |

---

## 13. Code Review Checklist (FastAPI-specific)

- [ ] Path operations are thin and async
- [ ] Routers are used with clear `prefix` and `tags`
- [ ] Dependencies use `Annotated` + `Depends`
- [ ] Request bodies are Pydantic models
- [ ] Errors use `HTTPException` with appropriate status codes
- [ ] No business logic inside path operation functions
- [ ] Lifespan is used instead of deprecated `on_startup`/`on_shutdown`
- [ ] Sensitive data never appears in responses or logs
- [ ] OpenAPI schema remains clean and useful (`/docs`)

---

## 14. Official Documentation Links

- [FastAPI Reference (root)](https://fastapi.tiangolo.com/reference/)
- [FastAPI class](https://fastapi.tiangolo.com/reference/fastapi/)
- [APIRouter](https://fastapi.tiangolo.com/reference/apirouter/)
- [Dependencies – Depends / Security](https://fastapi.tiangolo.com/reference/dependencies/)
- [HTTPException](https://fastapi.tiangolo.com/reference/exceptions/)
- [Request](https://fastapi.tiangolo.com/reference/request/)
- [Response](https://fastapi.tiangolo.com/reference/response/)
- [Bigger Applications – Multiple Files](https://fastapi.tiangolo.com/tutorial/bigger-applications/)
- [Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/)

---

This document is part of the project’s mandatory instruction set.  
Update it when FastAPI practices or the project architecture evolve.