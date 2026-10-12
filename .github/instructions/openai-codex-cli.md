# OpenAI Codex CLI — Comic Translate Middleware

This document is the **project-specific instruction set** for the OpenAI Codex CLI (`codex`) in the context of `comic-translate-middleware`.

It is based on the official documentation and repositories:

- [Codex CLI](https://developers.openai.com/codex/cli/)
- [Non-interactive mode (`codex exec`)](https://developers.openai.com/codex/noninteractive)
- [GitHub – openai/codex](https://github.com/openai/codex)
- Related Sign in with ChatGPT / plan-usage docs

and is aligned with the architecture decisions of the project plan.

---

## 1. Project Decision (important)

**The Comic Translate Middleware does NOT use the Codex CLI (or the Codex SDK) as the primary OpenAI integration path.**

Preferred path (see `openai-sdk-python.md`):

- Official `openai` Python package
- Sign in with ChatGPT (open-source OAuth + PKCE)
- Responses API (`client.responses.create` with `store=False`, `stream=True`)

Reasons:

1. Codex CLI is a **coding agent** (reads/writes files, runs shell commands, multi-step agentic loop).  
   Comic Translate needs a simple, multimodal **chat/completions-style** translation service.
2. Using Codex would add an unnecessary agentic layer, sandbox/approval complexity, and process-management overhead.
3. The official Responses API + SIWC flow already gives access to ChatGPT plan usage without scraping `~/.codex/` or reverse-engineering the CLI.
4. Multimodal image handling and the exact Chat Completions contract expected by Comic Translate are cleaner with the Python SDK.

**Therefore:**

- Do **not** implement an `OpenAICodexCliProvider` for the MVP.
- Do **not** read `~/.codex/auth.json` or any Codex credential files.
- Do **not** spawn `codex` / `codex exec` for translation requests.

This document exists so contributors understand what the Codex CLI is, how it authenticates, and why we deliberately stay away from it for the middleware.

---

## 2. What is Codex CLI?

Codex CLI is OpenAI’s local coding agent that runs in the terminal.

- Open-source (Rust implementation is the current default).
- Can inspect repositories, edit files, run commands, and use tools.
- Included with ChatGPT Plus, Pro, Business, Edu, and Enterprise plans.
- Also supports API-key authentication.

It is **not** a general-purpose chat completions client. It is an agentic coding assistant.

### Installation (reference only)

```bash
# Standalone installer (macOS / Linux)
curl -fsSL https://chatgpt.com/codex/install.sh | sh

# npm
npm install -g @openai/codex

# Homebrew
brew install --cask codex
```

Windows: native support exists; WSL2 is often recommended for the best experience.

---

## 3. Authentication

Codex supports two main ways to authenticate:

1. **Sign in with ChatGPT** (recommended for plan users)  
   - Interactive browser flow on first launch.
   - Credentials stored locally by the CLI (typically under `~/.codex/`).

2. **API key**  
   - `OPENAI_API_KEY` or equivalent configuration.

The middleware **must never**:

- Read or parse `~/.codex/auth.json` (or any other Codex auth file).
- Attempt to reuse Codex session tokens.
- Call private ChatGPT / Codex backend endpoints.

All plan-based access in the middleware goes through the official Sign in with ChatGPT open-source OAuth flow documented in `openai-sdk-python.md`.

---

## 4. Non-interactive / Headless Usage (`codex exec`)

Codex provides a non-interactive mode intended for scripts and CI:

```bash
codex exec "your task prompt here"
```

Characteristics (official behavior):

- Progress / logs go to **stderr**.
- Final agent message goes to **stdout** (easy to pipe).
- `--json` produces JSONL event streams.
- Supports sandbox modes, model selection, image attachments, output schemas, etc.
- Exit codes are meaningful (0 = success, non-zero = various error classes).

Example patterns:

```bash
codex exec "summarize the repository structure"
codex exec --json "run the tests and fix failures" > events.jsonl
codex exec --ephemeral "one-shot task that should not persist session"
```

Even though `codex exec` is powerful for coding automation, it is **the wrong abstraction** for a translation middleware that must:

- Speak a strict OpenAI Chat Completions protocol to Comic Translate.
- Preserve multimodal images.
- Return a deterministic JSON translation of page blocks.
- Avoid file-system side effects and agentic tool loops.

---

## 5. Relationship to the Middleware Architecture

```text
Comic Translate
      │
      │  POST /v1/chat/completions
      ▼
Middleware (FastAPI)
      │
      ├── OpenAI path  ──►  openai Python SDK
      │                      + SIWC OAuth
      │                      + Responses API
      │
      ├── Google path  ──►  Antigravity SDK  or  agy CLI
      │
      └── (future providers)
```

There is **no box** for “spawn `codex exec`” in the OpenAI path.

If in the future a contributor believes Codex CLI should be used (for example as an optional experimental backend), it would require:

- A new provider that still implements the common `BaseProvider` interface.
- Careful handling of stdout/stderr, JSONL events, timeouts, and cancellation.
- Explicit declaration that it is agentic and may have side effects.
- Clear documentation that it is **not** the recommended or default path.

Until such a decision is made and documented in the architecture, treat Codex CLI as out-of-scope for inference.

---

## 6. Security Rules (reinforced)

These rules apply specifically to anything related to Codex:

- **Never** read `~/.codex/auth.json` or any file under `~/.codex/`.
- **Never** extract or log Codex / ChatGPT session tokens.
- **Never** call undocumented `chatgpt.com/backend-api` or Codex-internal endpoints.
- Prefer the official open-source Sign in with ChatGPT flow + Responses API for plan usage.
- Keep the middleware bound to `127.0.0.1`.

Violating these rules would break the security model defined in the project plan.

---

## 7. When (if ever) Codex CLI might be relevant

Possible future, non-MVP scenarios (not currently planned):

- Using Codex as a **development aid** while building the middleware itself (human developers running `codex` interactively).
- An optional advanced “agentic translation” mode that is clearly separated from the standard Chat Completions path.
- Integration tests that verify the middleware can coexist with a local Codex installation without credential collisions.

None of these scenarios change the core rule: **translation traffic from Comic Translate goes through the official SDK + Responses API path**.

---

## 8. Comparison with the Chosen Approach

| Aspect                    | Codex CLI                          | Chosen path (SDK + SIWC + Responses) |
|---------------------------|------------------------------------|--------------------------------------|
| Primary purpose           | Coding agent                       | General inference / chat             |
| Protocol to Comic Translate | Would need heavy adaptation       | Native Chat Completions compatibility |
| Multimodal images         | Supported in interactive / exec    | First-class via Responses `input_image` |
| Plan usage                | Via CLI sign-in                    | Official SIWC OAuth + Responses      |
| Side effects              | File edits, shell, tools           | None (pure request/response)         |
| Process model             | Long-running agent / subprocess    | In-process async HTTP client         |
| Project recommendation    | Out of scope for MVP               | **Required** for OpenAI path         |

---

## 9. Official Links

| Resource | URL |
|----------|-----|
| Codex CLI overview | https://developers.openai.com/codex/cli/ |
| Non-interactive mode | https://developers.openai.com/codex/noninteractive |
| Command reference | https://developers.openai.com/codex/cli/reference |
| GitHub repository | https://github.com/openai/codex |
| Install script (macOS/Linux) | https://chatgpt.com/codex/install.sh |
| Sign in with ChatGPT (OSS) | https://developers.openai.com/siwc/token-sharing-open-source |

---

## 10. Summary for Contributors

1. Codex CLI is a powerful local coding agent — not a Chat Completions proxy.
2. The middleware **must not** depend on it for translation requests.
3. Do not read Codex credential files or reverse-engineer its auth.
4. All ChatGPT-plan usage goes through the official Sign in with ChatGPT flow and the Responses API (see `openai-sdk-python.md`).
5. Keep this separation clean so the architecture remains simple, secure, and aligned with OpenAI’s supported open-source contracts.

This document is part of the project’s mandatory instruction set.  
Update it only if the architecture explicitly decides to add an optional Codex-based provider in the future.