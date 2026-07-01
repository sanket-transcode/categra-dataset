# LLM Orchestrator — Current Working (as-built)

> Documented directly from the live code under `api/src/core/llm/` and its consumer `api/src/modules/app/ai/ai.service.ts`. Describes what the system actually does today, not the design intent.

---

## 1. Directory layout (actual)

```
api/src/core/llm/
├── llm.module.ts                          # @Global module wiring the layer
├── config/
│   └── llm-config.service.ts              # env → OperationConfig, validation, feature gate
├── orchestrator/
│   ├── llm-orchestrator.service.ts        # single public entry point + bootstrap
│   └── llm-provider-factory.service.ts    # provider name → provider instance
├── providers/
│   ├── llm-provider.interface.ts          # ILlmProvider contract
│   ├── gemini/
│   │   ├── gemini.module.ts               # @Global
│   │   └── gemini.service.ts              # @google/genai + Redis quota gating
│   └── ollama/
│       ├── ollama.module.ts               # @Global
│       └── ollama.service.ts              # ollama SDK + transport retry
├── prompts/
│   ├── prompt-builder.service.ts          # thin dispatcher
│   ├── generation.prompt.ts               # product / family-attributes / description
│   ├── localization.prompt.ts             # text / attribute localization
│   └── suggestion.prompt.ts               # Amazon attribute suggestion
├── errors/
│   └── llm-errors.ts                      # error classes + classifier
└── types/
    ├── llm-prompt.types.ts
    ├── llm-response.types.ts
    ├── llm-operation.types.ts
    └── llm-config.types.ts
```

There is **no** `llm.controller.ts` and **no** telemetry/utils folder — the layer is consumed in-process, not over HTTP.

---

## 2. Module wiring

`LlmModule` (`llm.module.ts`) is `@Global()`:
- **imports:** `forwardRef(() => DatabaseModule)` (for the `SEQUELIZE` provider used in billing), `GeminiModule`, `OllamaModule`
- **providers:** `LlmConfigService`, `PromptBuilderService`, `LlmProviderFactory`, `LlmOrchestratorService`, `GlobalResponses`
- **exports:** `LlmOrchestratorService`, `PromptBuilderService`

`GeminiModule` and `OllamaModule` are each `@Global()` and export their single provider service. Because everything is global, `LlmOrchestratorService` is injectable anywhere (e.g. `AIService`) without importing `LlmModule`.

---

## 3. The three operations

Source of truth: `GlobalEnums.LlmOperation` = `{ GENERATION, LOCALIZATION, SUGGESTION }` (values are the same uppercase strings). `LlmOperationName` is derived from it.

Each operation is an independent configuration unit (provider, model, generation params, transport). Public orchestrator methods map onto operations like this:

| Public method | Operation | Prompt builder | Consumer call site (`ai.service.ts`) |
|---|---|---|---|
| `suggestProductContent` | GENERATION | `buildProductPrompt` | L617 |
| `suggestFamilyAttributes` | GENERATION | `buildFamilyAttributesPrompt` | L1695 |
| `suggestProductDescription` | GENERATION | `buildDescriptionPrompt` | L1775 |
| `localizeText` | LOCALIZATION | `buildLocalizationPrompt` | L413 |
| `localizeAttributeValues` | LOCALIZATION | `buildAttributeLocalizationPrompt` | L1967 |
| `suggestAmazonAttributeValues` | SUGGESTION | `buildSuggestionPrompt` | L2025 |

Note: three GENERATION methods and two LOCALIZATION methods share the same operation config; only the prompt differs.

---

## 4. Types

**`LlmPrompt`** (provider-agnostic prompt) — `{ system: string; user: string; format: 'json' | 'text'; think: boolean }`.

**`LlmGenerateResponse`** (normalized provider output):
```
response: string
inputTokenCount / outputTokenCount / totalContextCount: number | null
provider: 'GEMINI' | 'OLLAMA'
model: string
durationMs: number
```
> `apiCallsCount` is **not** on this interface. The orchestrator injects `apiCallsCount: 1` only when handing the result to billing.

**`LlmOrchestrationResult`** extends `LlmGenerateResponse` with `prompt: LlmPrompt` — this is what every public orchestrator method returns, so callers get the exact prompt that produced the response.

**`OperationConfig`** — `{ operation, provider, model, generation: GenerationParams, apiKey?, host?, requestTimeoutMs?, warmupTimeoutMs?, maxAttempts?, retryBackoffMs? }`.

**`GenerationParams`** — `temperature, topP, topK, maxOutputTokens?, numPredict?, numCtx?, repeatPenalty?, think?, thinkingBudget?, geminiPlan?`.

**Input types** — `GenerationInput`, `FamilyAttributesInput`, `LocalizationInput`, `AttributeLocalizationInput`, `SuggestionInput`, `DescriptionInput` (in `llm-operation.types.ts`).

---

## 5. Configuration service (`LlmConfigService`)

### Build (eager, in constructor)
On instantiation it builds all three `OperationConfig`s via `buildConfig(op)` and caches them in a `Record<LlmOperationName, OperationConfig>`. **The constructor never throws** — it resolves raw env values, leaving required fields possibly empty; validation happens later at bootstrap.

`buildConfig(operation)` reads env by the canonical pattern `LLM_<OP>_*` (`<OP>` = uppercased operation name):
- `provider` = `normalizeProvider(LLM_<OP>_PROVIDER)` → uppercased raw value (no default)
- `model` = `LLM_<OP>_MODEL` (no default, no alias)
- `host` = `LLM_<OP>_HOST` (no default, no alias)
- `apiKey` = global `GEMINI_API_KEY`
- `generation` = `resolveGenerationParams(...)`
- `requestTimeoutMs` = `LLM_<OP>_REQUEST_TIMEOUT_MS` → `OLLAMA_REQUEST_TIMEOUT_MS` → `CONSTANTS.LLM_DEFAULT_REQUEST_TIMEOUT_MS` (300000)
- `warmupTimeoutMs` = `LLM_<OP>_WARMUP_TIMEOUT_MS` → `OLLAMA_WARMUP_TIMEOUT_MS` → `CONSTANTS.LLM_DEFAULT_WARMUP_TIMEOUT_MS` (60000)
- `maxAttempts` = `LLM_<OP>_MAX_ATTEMPTS`, clamped `[LLM_MIN_MAX_ATTEMPTS(1), LLM_MAX_MAX_ATTEMPTS(10)]`, default `LLM_DEFAULT_MAX_ATTEMPTS` (3)
- `retryBackoffMs` = `LLM_<OP>_RETRY_BACKOFF_MS` → `LLM_DEFAULT_RETRY_BACKOFF_MS` (1000)

`resolveGenerationParams` reads `LLM_<OP>_TEMPERATURE / _TOP_P / _TOP_K / _NUM_CTX / _REPEAT_PENALTY / _MAX_OUTPUT_TOKENS`, `think` from `LLM_<OP>_THINK` (=== 'true'), and `geminiPlan` from `LLM_<OP>_GEMINI_PLAN` (strictly per-operation, no global fallback). Defaults come from `operationDefaults`:
- base = `CONSTANTS.LLM_DEFAULT_GENERATION_PARAMS`
- LOCALIZATION overrides = `CONSTANTS.LLM_LOCALIZATION_PARAM_OVERRIDES` (higher temperature/topP/topK)

Env parsing helpers: `str` (trimmed string), `int` (first finite non-negative integer), `num` (first finite number), `bounded` (integer clamped to range).

### Validation (called at bootstrap)
- `validateAllOperations()` runs `validateOperation` for all three operations and returns a `LlmConfigurationIssue[]`.
- `validateOperation(name)`:
  - if provider not in `GlobalEnums.LlmProviders` → `provider:invalid(<value|unset>)`, returns immediately
  - always require `model`
  - provider = OLLAMA → require `host`
  - provider = GEMINI → require `apiKey`; require `geminiPlan` present **and** a key of `GEMINI_RATE_LIMIT` (else `geminiPlan:invalid(<value>)`)
  - returns an issue object listing everything missing, or `null` if valid
- `isSupportedProvider(value)` — membership check against `GlobalEnums.LlmProviders`.

### Feature gate
`isAiFeatureEnabled()` → `process.env.AI_FEATURE.toLowerCase() === 'true'`.

---

## 6. Orchestrator (`LlmOrchestratorService`)

Implements `OnApplicationBootstrap`. Injects: `GlobalResponses`, `LlmConfigService`, `LlmProviderFactory`, `OllamaService` (for warmup), `PromptBuilderService`, and `SEQUELIZE` (for billing).

### Bootstrap — `onApplicationBootstrap()`
1. If `AI_FEATURE` is off → **return immediately** (no validation, no warmup).
2. `validateAllOperations()`; if any issues → **throw `LlmConfigurationError`**, which aborts Nest startup (hard crash / fail-fast).
3. Ollama warmup: iterate the three operations, skip non-Ollama, dedupe by `host::model`, and fire `ollamaService.warmup(config)` wrapped in `withTimeout(...)`. Warmup is fire-and-forget (logs success/failure); a warmup failure does not stop boot.

### Public methods
Each of the six methods: resolves the operation config, builds the prompt via `PromptBuilderService`, and calls the private `execute(prompt, config, accountId?)`. All return `Promise<LlmOrchestrationResult>`.

### Core — `execute(prompt, config, accountId?)`
1. `assertFeatureEnabled()` — throws a localized `Error` (`ai_features_unavailable`) if AI is off.
2. Resolve the provider via the factory.
3. Retry loop up to `maxAttempts`:
   - call `provider.generate(prompt, config)`
   - on success: if `accountId`, fire `recordBilling(accountId, { ...result, apiCallsCount: 1 })` (**not awaited**), then return `{ ...result, prompt }`
   - on error: only retry if `error?.details?.retryable` is truthy **and** attempts remain (i.e. only `LlmProviderError` with `retryable: true`); otherwise rethrow. Backoff = `min(retryBackoffMs * attempt, CONSTANTS.LLM_MAX_RETRY_BACKOFF_MS)` (5000).

### Billing — `recordBilling(accountId, result)`
Single atomic SQL `UPDATE tbl_account_configurations SET consumed_ai_token = jsonb_build_object('input', COALESCE(...)+:input, 'output', COALESCE(...)+:output, 'apiCalls', COALESCE(...)+:apiCalls) WHERE account_id = :accountId`. `COALESCE` handles null/missing JSON keys; no read-modify-write race. Errors are caught and logged (billing never breaks a generate call).

### `withTimeout(promise, ms)`
Races a promise against a timeout that rejects with an `AbortError`. Used only for warmup.

---

## 7. Provider factory (`LlmProviderFactory`)

`resolve(name)`:
- `GEMINI` → `GeminiService`
- `OLLAMA` → `OllamaService`
- anything else → throws `LlmConfigurationError` (`provider:unsupported(...)`). No silent fallback.

## 8. Provider interface (`ILlmProvider`)

`{ providerName: string; generate(prompt, config): Promise<LlmGenerateResponse>; warmup?(config): Promise<void> }`.

---

## 9. Gemini provider (`GeminiService`)

- Uses `@google/genai` (`GoogleGenAI`). Clients cached in a `Map<apiKey, GoogleGenAI>` via `getOrCreateClient`.
- `generate(prompt, config)`:
  1. resolve client from `config.apiKey`
  2. `estimateTokens` via `client.models.countTokens` (best-effort; 0 on failure)
  3. `waitForQuota(estimatedTokens, model, plan)` — blocks until quota allows
  4. `client.models.generateContent({ model, contents:[user], config:{ systemInstruction, temperature, topP, topK, maxOutputTokens, responseMimeType:'application/json' when format==='json' } })`
  5. read `usageMetadata.promptTokenCount` / `candidatesTokenCount`
  6. `trackTokenUsage(input+output, model, plan)`
  7. return normalized `LlmGenerateResponse` (`totalContextCount = input+output`)
  - on error → `LlmProviderError` with `type: 'PROVIDER_FAILURE'`, `retryable: false`
- **Quota gating** (`waitForQuota`) — Redis-backed sliding windows, global per `model:plan`:
  - key `${GEMINI_QUOTA_KEY_PREFIX}<model>:<plan>`, guarded by a Redis lock (`GEMINI_QUOTA_LOCK_TTL_SECONDS`; retries every `GEMINI_QUOTA_LOCK_RETRY_DELAY_MS` if lock busy)
  - tracks minute window (requests + tokens) and day window (requests) against `GEMINI_RATE_LIMIT[plan]` (`MAX_REQUESTS_PER_MINUTE`, `MAX_TOKENS_PER_MINUTE`, `MAX_REQUESTS_PER_DAY`)
  - when allowed, increments counters (TTL `GEMINI_QUOTA_TTL_SECONDS`); when not, waits `min(computedWait, GEMINI_QUOTA_MAX_WAIT_MS)` (floor `GEMINI_QUOTA_MIN_WAIT_MS`) and loops
  - if `plan` has no entry in `GEMINI_RATE_LIMIT`, gating is skipped (`return`) — but strict validation prevents an invalid plan from reaching here.
- `trackTokenUsage` — after a call, adds actual tokens into the current minute window (same lock discipline).
- `think` is ignored by Gemini.

## 10. Ollama provider (`OllamaService`)

- Uses the `ollama` SDK. Clients pooled in `Map<host::requestTimeoutMs, Ollama>` via `getOrCreateClient`, each with a custom timeout-aware `fetch` (`createTimeoutFetch`) that aborts after `requestTimeoutMs` and normalizes timeouts to code `OLLAMA_REQUEST_TIMEOUT`.
- `generate` → `buildRequest` then `generateWithRetry`.
- `buildRequest(prompt, config)` maps generation params to Ollama `options` (`num_ctx`, `temperature`, `top_p`, `top_k` when >0, `repeat_penalty`), sets `format:'json'` when the prompt is json, and enables `think` only when **both** `generation.think` and `prompt.think` are true.
- `executeGenerate` calls `client.generate({ model, ...request, stream:false })`, reads `prompt_eval_count` / `eval_count` for token counts, returns normalized `LlmGenerateResponse`; on failure wraps in `LlmProviderError` (classified via `classifyLlmProviderError`).
- `generateWithRetry` — provider-level **transport retry** up to `maxAttempts`:
  - retries only when `isTransient(error)` is true; logs structured JSON events (`ollama_generate_retry_recovered/failed/retry_scheduled`)
  - backoff = `min((retryBackoffMs) * attempt, LLM_MAX_RETRY_BACKOFF_MS)` (5000)
  - `isTransient` — true for `LlmProviderError.details.retryable`, codes in `CONSTANTS.OLLAMA_TRANSIENT_ERROR_CODES`, HTTP statuses in `CONSTANTS.OLLAMA_RETRYABLE_HTTP_STATUSES` (429/500/502/503/504), or `TypeError: fetch failed`
- `warmup(config)` — sends a minimal `generate` (`prompt:'warmup'`, `num_predict: OLLAMA_WARMUP_NUM_PREDICT`) to load the model; called by the orchestrator at bootstrap.

---

## 11. Prompt building

`PromptBuilderService` is a thin dispatcher — each method delegates to a pure builder function in `generation.prompt.ts` / `localization.prompt.ts` / `suggestion.prompt.ts`, all returning `LlmPrompt`. The orchestrator never inlines prompt text.

---

## 12. Error model (`errors/llm-errors.ts`)

- **`LlmProviderError`** — runtime provider/transport failure. Carries `details` (`type`, `code`, `provider`, `model`, `durationMs?`, `timeoutMs?`, `retryable`). The orchestrator's retry decision keys entirely off `details.retryable`.
- **`LlmConfigurationError`** — boot-time / factory misconfiguration. Carries `issues: LlmConfigurationIssue[]` (`operation`, `provider`, `missing[]`); message aggregates all issues.
- **`classifyLlmProviderError(error)`** — maps raw SDK/transport errors to `{ type, code, retryable }`: TIMEOUT and SERVICE_UNAVAILABLE and MALFORMED_RESPONSE are `retryable: true`; PROVIDER_FAILURE is `retryable: false`.

---

## 13. End-to-end request flow

```
AIService (business)                       # e.g. localizeText / suggestProductContent
   │  calls llmOrchestrator.<method>(input, accountId?)
   ▼
LlmOrchestratorService.<method>
   │  resolveOperation(op) → OperationConfig
   │  promptBuilder.build<...>(input) → LlmPrompt
   ▼
execute(prompt, config, accountId?)
   │  assertFeatureEnabled()                # throws if AI_FEATURE != true
   │  factory.resolve(config.provider)      # Gemini | Ollama (else LlmConfigurationError)
   │  retry loop (maxAttempts):
   │      provider.generate(prompt, config) # normalized LlmGenerateResponse
   │        ├─ Gemini: quota-gate → generateContent → track usage
   │        └─ Ollama: buildRequest → generate (own transport retry)
   │      on success → recordBilling(...) [fire-and-forget] → return { ...result, prompt }
   │      on retryable LlmProviderError → backoff & retry
   ▼
LlmOrchestrationResult  →  back to AIService (parses response, post-processes)
```

---

## 14. Retry layering (two independent layers)

| Layer | Retries what | Bound | Backoff cap |
|---|---|---|---|
| `OllamaService.generateWithRetry` | transient transport errors (codes/statuses/fetch-failed) | `config.maxAttempts` | `LLM_MAX_RETRY_BACKOFF_MS` (5000) |
| `LlmOrchestratorService.execute` | any `LlmProviderError` with `details.retryable === true` | `config.maxAttempts` | `LLM_MAX_RETRY_BACKOFF_MS` (5000) |

Gemini has no internal generate-retry (it only blocks on quota); its failures surface as non-retryable `PROVIDER_FAILURE`, so the orchestrator loop does not retry them. Ollama's own retries can compound with the orchestrator loop for transient errors it ultimately rethrows.

---

## 15. Consumer (`AIService`)

`AIService` injects `LlmOrchestratorService` and calls the six public methods (see §3), passing `req.user.accountId` for billing. It owns all business post-processing: JSON parsing/sanitization of `result.response`, localization chunking + output validation, family/advisor normalization, etc. The orchestrator itself is provider-agnostic and returns only the normalized result + prompt.

---

## 16. Not currently implemented

- No HTTP controller for the orchestrator (in-process only).
- No streaming, provider fallback, response caching, cost estimation, or prompt versioning.
- No per-operation "enabled" flag — with `AI_FEATURE=true`, all three operations must be fully configured or boot fails.
- `numPredict` / `thinkingBudget` exist on `GenerationParams` but are not currently wired into provider requests.
