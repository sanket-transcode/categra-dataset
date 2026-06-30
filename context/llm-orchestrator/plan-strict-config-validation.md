# Plan: Strict LLM Config Validation (provider + required params)

> Sibling addendum to `plan-v2.md`. Hardens `LlmConfigService` / `LlmOrchestratorService` so misconfiguration fails loudly at boot instead of silently degrading at runtime.

---

## 1. Context & Problem

The LLM config layer currently masks misconfiguration with silent fallbacks, so a broken `.env` produces wrong-but-running behavior instead of a clear failure:

- **`LlmConfigService.buildConfig()`** falls back on every required field:
  - provider → `OLLAMA`
  - model → alias → `OLLAMA_MODEL` → `''`
  - host → alias → `http://localhost:11434`
  - geminiPlan → `TIER_1`
- **`LlmProviderFactory.resolve()`** treats **any** non-Gemini value as Ollama (`if (name === GEMINI) return gemini; return ollama;`), so an unsupported/typo provider name silently routes to Ollama.
- **`GeminiService.generate()`** reads `process.env.GEMINI_API_KEY` directly; a missing key is not caught until the API call fails deep inside a user request.
- A missing/invalid `geminiPlan` makes `waitForQuota()` return early (`if (!limits) return;`) → **all quota gating silently disabled**.

**Goal:** Validate each operation's required params strictly, classified by provider approach. Reject unsupported providers. On any missing/invalid required param, throw a clear error **eagerly at app boot** so a misconfigured deploy never starts and never serves degraded behavior.

**Confirmed decisions:**
| Decision | Choice |
|---|---|
| Validation timing | **Eager at app boot** (fail-fast) |
| When `AI_FEATURE` is OFF | **No AI bootstrap at all** — skip both validation **and** Ollama warmup |
| When `AI_FEATURE` is ON | **All 3 operations** (`GENERATION`, `LOCALIZATION`, `SUGGESTION`) are mandatory and fully validated |
| Env sourcing for `provider`/`model`/`host` | **Strictly per-operation** — no global defaults, no hardcoded fallbacks, **no legacy aliases** |
| Gemini API key | **Single global `GEMINI_API_KEY`**, used by every operation whose provider is `GEMINI` |
| Required set | Core (provider + model) + Ollama host + Gemini key & plan |
| Generation tuning params | **Not** required — keep existing sensible defaults |
| Error surface | **New `LlmConfigurationError` class** (non-retryable, structured) |
| On validation failure | **Hard crash** — error propagates out of `onApplicationBootstrap`, Nest aborts startup, server does not start |
| Legacy `AIService` Gemini path (`getGeminiBatchByName`) | **Out of scope** — left untouched |

---

## 2. Required-Param Classification

All required fields are sourced **only** from the canonical `LLM_<OP>_*` variables (where `<OP>` ∈ `GENERATION` / `LOCALIZATION` / `SUGGESTION`). The legacy `AI_PRODUCT_*` / `AI_TRANSLATE_*` / `OLLAMA_*` / `localhost` fallbacks are removed so all three operations follow one uniform contract.

### Common (both providers)
| Param | Env source | Rule |
|---|---|---|
| `provider` | `LLM_<OP>_PROVIDER` | Normalize-uppercase, must be exactly `OLLAMA` or `GEMINI` (`GlobalEnums.LlmProviders`). No default. Unknown → reject. |
| `model` | `LLM_<OP>_MODEL` | Required, non-empty. Drop the alias / `OLLAMA_MODEL` / `''` fallback. |

### Gemini-specific (when provider = GEMINI)
| Param | Env source | Rule |
|---|---|---|
| `apiKey` | `GEMINI_API_KEY` (global) | Required, non-empty. Shared by all Gemini-provider operations. |
| `geminiPlan` | `LLM_<OP>_GEMINI_PLAN` | Required, **strictly per-operation** — no global `GEMINI_PLAN` fallback, no `TIER_1` default. Must be a key of `GEMINI_RATE_LIMIT` (`TIER_0`/`TIER_1`/`TIER_2`/`TIER_3`). |

### Ollama-specific (when provider = OLLAMA)
| Param | Env source | Rule |
|---|---|---|
| `host` | `LLM_<OP>_HOST` | Required, non-empty. Drop the alias / `http://localhost:11434` fallback. |

### Stay optional (keep current defaults — NOT validated)
- Generation tuning: `temperature`, `topP`, `topK`, `maxOutputTokens`, `numCtx`, `repeatPenalty`, `think`, `thinkingBudget`
- Transport: `requestTimeoutMs`, `warmupTimeoutMs`, `maxAttempts`, `retryBackoffMs`

---

## 3. Changes by File

### 3.1 `src/core/llm/errors/llm-errors.ts` — new error class
Add alongside `LlmProviderError`:
```ts
export interface LlmConfigurationIssue {
	operation: string;          // 'GENERATION' | 'LOCALIZATION' | 'SUGGESTION'
	provider: string | null;    // normalized provider or null if unresolved
	missing: string[];          // e.g. ['model', 'host'] | ['provider:invalid(openai)'] | ['geminiPlan:invalid(TIER_9)']
}

export class LlmConfigurationError extends Error {
	readonly issues: LlmConfigurationIssue[];

	constructor(issues: LlmConfigurationIssue[]) {
		const summary = issues
			.map((i) => `${i.operation}[${i.provider ?? 'unknown'}]: ${i.missing.join(', ')}`)
			.join(' | ');
		super(`Invalid LLM configuration -> ${summary}`);
		this.name = 'LlmConfigurationError';
		this.issues = issues;
	}
}
```
Aggregates issues across all three operations so the operator sees every problem in a single boot failure.

### 3.2 `src/core/llm/types/llm-config.types.ts` — extend config
Add to `OperationConfig`:
```ts
apiKey?: string;   // Gemini API key, centralized here instead of read inside the provider
```

### 3.3 `src/core/llm/config/llm-config.service.ts` — resolve strictly + validate
- **`buildConfig()`**: resolve raw values **without strict-required fallbacks**:
  - provider: normalized `LLM_<OP>_PROVIDER` (no `OLLAMA` default)
  - `model`: `LLM_<OP>_MODEL` only (no alias / `OLLAMA_MODEL` / `''`)
  - `host`: `LLM_<OP>_HOST` only (no alias / `localhost` default)
  - `apiKey`: from global `GEMINI_API_KEY`
  - `geminiPlan`: `LLM_<OP>_GEMINI_PLAN` only (no global `GEMINI_PLAN`, no `TIER_1` default)
  - generation/transport defaults **unchanged**
  - **Constructor must NOT throw** — it builds configs leaving required fields possibly empty/invalid; the validator (called from the orchestrator at bootstrap) is what throws. Keeps DI construction clean and validation in one controllable place.
- **`resolveModel()` / `resolveHost()`**: simplify to the single canonical `LLM_<OP>_*` lookup — **remove the alias branches and the hardcoded final fallbacks** (return `''` when unset).
- **`resolveGenerationParams()`**: `geminiPlan` resolved from `LLM_<OP>_GEMINI_PLAN` only — remove the global `GEMINI_PLAN` fallback and the hardcoded default.
- **New `isSupportedProvider(value: string): boolean`** — checks normalized value against `GlobalEnums.LlmProviders`.
- **New `validateOperation(name: LlmOperationName): LlmConfigurationIssue | null`**:
  - resolve config; collect `missing[]`
  - provider not supported → `missing.push('provider:invalid(<raw>)')` and return early (can't check provider-specific params)
  - always require `model`
  - provider=OLLAMA → require `host`
  - provider=GEMINI → require `apiKey`; require `geminiPlan` present **and** `geminiPlan in GEMINI_RATE_LIMIT` (else `geminiPlan:invalid(<value>)`)
  - return issue object if `missing.length`, else `null`
- **New `validateAllOperations(): LlmConfigurationIssue[]`** — maps the three `GlobalEnums.LlmOperation` values through `validateOperation`, filters nulls.

### 3.4 `src/core/llm/orchestrator/llm-orchestrator.service.ts` — AI-gated boot validation + warmup
Restructure `onApplicationBootstrap()` so the **entire** body is gated on the feature flag:
```ts
onApplicationBootstrap(): void {
	if (!this.configService.isAiFeatureEnabled()) return;   // AI off → no validation, no warmup

	const issues = this.configService.validateAllOperations();
	if (issues.length) throw new LlmConfigurationError(issues);   // hard crash → server won't start

	// ... existing Ollama warmup loop (unchanged), now runs only on a validated config
}
```
- `AI_FEATURE` OFF → returns immediately; nothing LLM-related runs at boot.
- `AI_FEATURE` ON → validate all three operations; any issue throws and aborts startup; otherwise warmup proceeds.

### 3.5 `src/core/llm/orchestrator/llm-provider-factory.service.ts` — hard guard
Replace the silent fallback with an explicit switch:
```ts
resolve(name: LlmProviderName): ILlmProvider {
	if (name === GlobalEnums.LlmProviders.GEMINI) return this.geminiService;
	if (name === GlobalEnums.LlmProviders.OLLAMA) return this.ollamaService;
	throw new LlmConfigurationError([{ operation: 'unknown', provider: name, missing: [`provider:unsupported(${name})`] }]);
}
```
Defense-in-depth — boot validation already catches this, but the factory no longer silently coerces unknown providers to Ollama.

### 3.6 `src/core/llm/providers/gemini/gemini.service.ts` — read key from config
In `generate()`, replace:
```ts
const apiKey = String(process.env.GEMINI_API_KEY || '');
```
with:
```ts
const apiKey = String(config.apiKey || '');
```
Key is guaranteed present by boot validation; config becomes the single source of truth.
> Note: the legacy `AIService.genAiClient` / `getGeminiBatchByName` path keeps its own `GOOGLE_API_KEY`-based client and is intentionally left untouched.

---

## 4. Edge Cases / Notes
- **Provider casing:** uppercase the env value before comparison so `gemini`/`Gemini` are accepted; genuinely unknown values reported as `provider:invalid(<value>)`.
- **Aggregated reporting:** all three operations validated in one pass; a single boot failure lists every problem rather than failing one at a time.
- **All-or-nothing per AI_FEATURE:** with AI on, all three operations must be fully configured — there is no per-operation enable flag.
- **No DI-time throw:** `LlmConfigService` constructor stays side-effect-light; only the orchestrator bootstrap throws.
- **Dropping legacy aliases is a behavior change:** deployments relying on `AI_PRODUCT_MODEL` / `AI_TRANSLATE_MODEL` / `AI_PRODUCT_HOST` / `AI_TRANSLATE_HOST` / `OLLAMA_MODEL` / `OLLAMA_HOST` must migrate to `LLM_<OP>_MODEL` / `LLM_<OP>_HOST`.

---

## 5. Verification
1. **Build:** `npm run build` in `api/` — compiles clean.
2. **Unsupported provider:** `LLM_GENERATION_PROVIDER=openai`, `AI_FEATURE=true` → app fails to boot; `LlmConfigurationError` names `GENERATION` / `provider:invalid(openai)`.
3. **Missing Ollama params:** provider=OLLAMA with unset `LLM_<OP>_MODEL` / `LLM_<OP>_HOST` → boot error lists `model`, `host`.
4. **Missing Gemini params:** provider=GEMINI with unset `GEMINI_API_KEY` or bad `GEMINI_PLAN=TIER_9` → boot error lists `apiKey` / `geminiPlan:invalid(TIER_9)`.
5. **Aggregation:** misconfigure two operations at once → single boot error lists both.
6. **All three required:** leave `SUGGESTION` entirely unconfigured with `AI_FEATURE=true` → boot fails for `SUGGESTION`.
7. **Happy path:** all three operations fully configured → boots, Ollama warmup runs, real generate/localize/suggest calls succeed.
8. **AI disabled:** `AI_FEATURE=false` with incomplete LLM env → boots fine, **no validation and no warmup** run.
