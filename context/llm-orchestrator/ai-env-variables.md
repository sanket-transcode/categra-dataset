# AI Feature — Environment Variables Reference

> Every env variable the AI feature actually reads, with required/optional status. Derived from the live code: `llm-config.service.ts`, `llm-orchestrator.service.ts`, `gemini.service.ts`, `ollama.service.ts`, and `ai.service.ts`. Reflects the strict-config-validation behavior (validation runs eagerly at boot when `AI_FEATURE=true`).

`<OP>` expands to one of **`GENERATION`**, **`LOCALIZATION`**, **`SUGGESTION`** — each operation is configured independently, so every `LLM_<OP>_*` row below exists three times (once per operation).

Operation → business method mapping:
| Operation | Business methods |
|---|---|
| `GENERATION` | `suggestProductContent`, `suggestFamilyAttributes`, `suggestProductDescription` |
| `LOCALIZATION` | `localizeText`, `localizeAttributeValues` |
| `SUGGESTION` | `suggestAmazonAttributeValues` |

---

## 1. Global gate

| Variable | Required? | Default | Notes |
|---|---|---|---|
| `AI_FEATURE` | **Required to use AI** | unset → treated as off | Must be the string `true` to enable any LLM operation. When off: no boot validation, no Ollama warmup, and every orchestrator call throws the feature-disabled error. |

---

## 2. Global Gemini credential

| Variable | Required? | Default | Notes |
|---|---|---|---|
| `GEMINI_API_KEY` | **Required if any operation's provider is `GEMINI`** (when `AI_FEATURE=true`) | none | Single global key, shared by every Gemini-provider operation. Read into `OperationConfig.apiKey`; `GeminiService` uses it. Boot fails if a Gemini operation has no key. |

---

## 3. Per-operation — REQUIRED (no fallback, strictly per-operation)

Validated at boot for every operation when `AI_FEATURE=true`. Any missing/invalid one aborts startup with `LlmConfigurationError`.

| Variable | Required when | Allowed values | Notes |
|---|---|---|---|
| `LLM_<OP>_PROVIDER` | **Always** | `OLLAMA` or `GEMINI` (case-insensitive, normalized to uppercase) | No default. Any other value → `provider:invalid(...)` and boot fails. |
| `LLM_<OP>_MODEL` | **Always** | non-empty string | No alias, no default. e.g. `llama3.2`, `gemini-2.5-flash`. |
| `LLM_<OP>_HOST` | **When provider = `OLLAMA`** | URL | No `http://localhost:11434` fallback. Ignored for Gemini. |
| `LLM_<OP>_GEMINI_PLAN` | **When provider = `GEMINI`** | `TIER_0` / `TIER_1` / `TIER_2` / `TIER_3` | Strictly per-operation — **no** global `GEMINI_PLAN` fallback, no `TIER_1` default. Must be a key of `GEMINI_RATE_LIMIT`; unknown value → `geminiPlan:invalid(...)`. Drives Redis quota gating. Ignored for Ollama. |

---

## 4. Per-operation — OPTIONAL (defaults applied, not validated)

These never block boot. Defaults come from `operationDefaults()` / `buildConfig()`.

### Generation tuning
| Variable | Default | Applies to | Notes |
|---|---|---|---|
| `LLM_<OP>_TEMPERATURE` | `0` (generation/suggestion), `0.5` (localization) | both | |
| `LLM_<OP>_TOP_P` | `0.9` (gen/sugg), `0.95` (localization) | both | |
| `LLM_<OP>_TOP_K` | `0` (gen/sugg), `64` (localization) | both | Ollama applies only when `> 0`. |
| `LLM_<OP>_NUM_CTX` | `8192` | Ollama | Context window; applied only when `> 0`. |
| `LLM_<OP>_REPEAT_PENALTY` | `1.05` | Ollama | |
| `LLM_<OP>_MAX_OUTPUT_TOKENS` | `8192` | Gemini | Passed as `maxOutputTokens`. |
| `LLM_<OP>_THINK` | `false` | Ollama only | `true`/`false`. Extended-thinking flag; ignored by Gemini. Effective only when the prompt also enables think. |

### Transport / retry
| Variable | Default | Applies to | Notes |
|---|---|---|---|
| `LLM_<OP>_REQUEST_TIMEOUT_MS` | `300000` | Ollama | Falls back to `OLLAMA_REQUEST_TIMEOUT_MS` (see §5). |
| `LLM_<OP>_WARMUP_TIMEOUT_MS` | `60000` | Ollama | Warmup deadline at boot. Falls back to `OLLAMA_WARMUP_TIMEOUT_MS` (see §5). |
| `LLM_<OP>_MAX_ATTEMPTS` | `3` (clamped to 1–10) | both | Provider-level transport retry budget; also used by orchestrator common-retry. |
| `LLM_<OP>_RETRY_BACKOFF_MS` | `1000` | both | Base backoff; capped at 5000ms in orchestrator/Ollama. |

---

## 5. Optional transport aliases (still consulted)

Only these two legacy aliases remain wired, as **optional** fallbacks for the transport timeouts when the per-operation var is absent:

| Alias | Falls back into | Required? |
|---|---|---|
| `OLLAMA_REQUEST_TIMEOUT_MS` | `LLM_<OP>_REQUEST_TIMEOUT_MS` | Optional |
| `OLLAMA_WARMUP_TIMEOUT_MS` | `LLM_<OP>_WARMUP_TIMEOUT_MS` | Optional |

---

## 6. Deprecated / no longer used for config resolution

Strict validation removed these as config sources. They are **not** read when resolving provider/model/host/plan anymore:

| Variable | Status |
|---|---|
| `GEMINI_PLAN` (global) | **No longer a source** — plan is strictly per-operation (`LLM_<OP>_GEMINI_PLAN`). |
| `OLLAMA_HOST` | **No longer a source** for host — use `LLM_<OP>_HOST`. |
| `OLLAMA_MODEL` | **No longer a source** for model — use `LLM_<OP>_MODEL`. |
| `AI_PRODUCT_MODEL` / `AI_PRODUCT_HOST` | **No longer a source** for generation — use `LLM_GENERATION_MODEL` / `LLM_GENERATION_HOST`. |
| `AI_TRANSLATE_MODEL` / `AI_TRANSLATE_HOST` | **No longer a source** for localization — use `LLM_LOCALIZATION_MODEL` / `LLM_LOCALIZATION_HOST`. |
| `GEMINI_MODEL`, `AI_TRANSLATE_MODEL`, `LLM_LOCALIZATION_MODEL` (in `ai.service.ts:154`) | Read **only** to build a human-readable model label inside a localization error object — not used to configure any call. |

---

## 7. Adjacent AI-feature variables (image lookup & signing)

Used by `AIService.imagesAI` (Google Custom Search image lookup, exposed under the AI controller) and signed-URL generation. Independent of the LLM orchestrator and **not** part of boot validation.

| Variable | Required? | Notes |
|---|---|---|
| `GOOGLE_API_KEY` | Required for the AI **image lookup** endpoint | Google Custom Search API key. |
| `GOOGLE_CUSTOM_SEARCH_ENGINE_ID` | Required for the AI **image lookup** endpoint | Custom Search engine (cx) id. |
| `JWT_SECRET` | Required (app-wide) | Used by `generateSignedUrl` for secure image URLs; also a global auth secret outside the AI feature. |
| `NODE_ENV` | Optional (app-wide) | Only gates debug logging in `isValidImage` (`!== 'production'`). |

---

## 8. Minimal working examples

### All operations on Ollama
```bash
AI_FEATURE=true

LLM_GENERATION_PROVIDER=OLLAMA
LLM_GENERATION_MODEL=llama3.2
LLM_GENERATION_HOST=http://localhost:11434

LLM_LOCALIZATION_PROVIDER=OLLAMA
LLM_LOCALIZATION_MODEL=llama3.2
LLM_LOCALIZATION_HOST=http://localhost:11434

LLM_SUGGESTION_PROVIDER=OLLAMA
LLM_SUGGESTION_MODEL=llama3.2
LLM_SUGGESTION_HOST=http://localhost:11434
```

### Mixed (generation+suggestion on Ollama, localization on Gemini)
```bash
AI_FEATURE=true
GEMINI_API_KEY=AIza...

LLM_GENERATION_PROVIDER=OLLAMA
LLM_GENERATION_MODEL=llama3.2
LLM_GENERATION_HOST=http://localhost:11434

LLM_LOCALIZATION_PROVIDER=GEMINI
LLM_LOCALIZATION_MODEL=gemini-2.5-flash
LLM_LOCALIZATION_GEMINI_PLAN=TIER_1

LLM_SUGGESTION_PROVIDER=OLLAMA
LLM_SUGGESTION_MODEL=llama3.2
LLM_SUGGESTION_HOST=http://localhost:11434
```

> With `AI_FEATURE=true`, all three operations must be fully configured — even ones you don't actively call — or the server refuses to start.
