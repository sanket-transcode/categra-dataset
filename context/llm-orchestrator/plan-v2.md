# LLM Orchestration Layer — Implementation Plan (V2, Revised)

> Incorporates all clarifications. Supercedes the original V2 draft.

---

## Clarification Resolutions (applied throughout)

| # | Decision |
|---|---|
| 1 | `@google/generative-ai` removed entirely; `@google/genai` only |
| 2 | `GEMINI_PLAN` env-configurable, per-operation override supported, ignored by Ollama |
| 3 | Exactly **3 operations**: `generation`, `localization`, `suggestion` |
| 4 | Ollama warmup runs only for operations where provider = `ollama` |
| 5 | `AI_FEATURE` is a universal gate at orchestrator; throws on disabled, all providers blocked |
| 6 | Billing at orchestrator via atomic SQL; providers return `inputTokenCount + outputTokenCount + apiCallsCount` |
| 7 | Debug cache writes (`AiPayloadForLabel`, `AiResponse`) removed |
| 8 | Provider handles own transport retry; orchestrator retries on empty/failed response; business layer handles output validation; localization chunk logic simplified |
| 9 | `think` flag as `LLM_<OPERATION>_THINK=true/false` env var, Ollama-only, ignored by Gemini |
| 10 | `LlmController` kept, endpoint updated to call orchestrator |
| 11 | All commented-out code in old `promptBuilder.ts` deleted, not preserved |
| 12 | Old `LlmService` hard-deleted after all call sites migrated |
| + | **`translation` renamed to `localization` everywhere** — method names, variable names, error codes, log keys, type names, env vars |

---

## The 3 Operations

Each operation is an independent configuration unit (provider, model, all generation params). Business methods map onto them as follows:

| Operation | Business methods that use it |
|---|---|
| `generation` | `generateProduct`, `suggestFamilyAttributes` |
| `localization` | `localizeText` (was `translateText`), `localizeAttributes` (was `localizeAttributeValues`) |
| `suggestion` | `suggestAmazonAttributes` |

The orchestrator builds a different prompt per business method but uses the same provider/model/params for all methods within the same operation.

---

## Final File Structure

```
api/src/core/llm/
│
├── llm.module.ts                          ← REWRITTEN
├── llm.controller.ts                      ← UPDATED (inject orchestrator)
│
├── orchestrator/
│   ├── llm-orchestrator.service.ts        ← NEW: single public entry point
│   └── llm-provider-factory.service.ts    ← NEW
│
├── providers/
│   ├── llm-provider.interface.ts          ← NEW
│   ├── gemini/
│   │   ├── gemini.module.ts               ← NEW
│   │   └── gemini.service.ts              ← NEW (extracted from ai.service.ts)
│   └── ollama/
│       ├── ollama.module.ts               ← NEW
│       └── ollama.service.ts              ← NEW (refactored from llm.service.ts)
│
├── prompts/
│   ├── prompt-builder.service.ts          ← NEW: unified, replaces both old builders
│   ├── generation.prompt.ts               ← NEW (product + family attributes)
│   ├── localization.prompt.ts             ← NEW (text + attribute localization)
│   └── suggestion.prompt.ts              ← NEW (Amazon attributes + future)
│
├── config/
│   └── llm-config.service.ts              ← NEW
│
├── errors/
│   └── llm-errors.ts                      ← MOVED (content unchanged)
│
└── types/
    ├── llm-prompt.types.ts
    ├── llm-response.types.ts
    ├── llm-operation.types.ts
    └── llm-config.types.ts
```

### Files Deleted After Migration

| Deleted | Replaced by |
|---|---|
| `api/src/core/llm/llm.service.ts` | `ollama.service.ts` + `llm-orchestrator.service.ts` |
| `api/src/core/llm/prompt-builder.ts` | `prompts/prompt-builder.service.ts` |
| `api/src/core/llm/llm-errors.ts` | `errors/llm-errors.ts` (moved) |
| `api/src/core/base/promptBuilder.ts` | `prompts/prompt-builder.service.ts` (all commented code deleted, not preserved) |

---

## Types

### `types/llm-prompt.types.ts`

```ts
export interface LlmPrompt {
  system: string;
  user: string;
  format?: 'json' | 'text';   // defaults to 'json' for all current operations
  think?: boolean;             // Ollama-only; set from OperationConfig, ignored by Gemini
}
```

### `types/llm-response.types.ts`

```ts
export interface LlmGenerateResponse {
  response: string;
  inputTokenCount: number | null;
  outputTokenCount: number | null;
  totalContextCount: number | null;
  apiCallsCount: number;         // always 1 per generate(); used for billing
  provider: string;
  model: string;
  durationMs: number;
}
```

Both `GeminiService` and `OllamaService` must populate all fields. `apiCallsCount` is always `1` — aggregation happens atomically in the DB.

### `types/llm-operation.types.ts`

```ts
export type LlmOperationName = 'generation' | 'localization' | 'suggestion';

// Business-level input types
export interface GenerationInput {
  languageCode: string;
  productName: string;
  family?: string;
  existingFamilies?: { id: string; name: string }[];
  selectedFamilyId?: string;
  selectedFamilySummary?: any;
  candidateFamilySummaries?: any[];
  canManageFamilies?: boolean;
  Images?: string[];
  Description?: string;
  attributes?: any[];
  variantAttributes?: any[];
}

export interface FamilyAttributesInput {
  languageCode: string;
  family: string;
  attributes?: any[];
  variantAttributes?: any[];
}

export interface LocalizationInput {
  inputText: Record<string, string>;
  languages: string[];
  context?: Record<string, unknown>;
}

export interface AttributeLocalizationInput {
  targetLanguage: string;
  sourceData: Record<string, any>;
  attributeMetadata: Record<string, any>;
  productContext?: string;
}

export interface SuggestionInput {
  marketplaceCountry: string;
  marketplaceLanguage: string;
  productType: string;
  productAttributesPayload: any;
  imageUrls: string[];
  schema: any;
}
```

### `types/llm-config.types.ts`

```ts
export type LlmProviderName = 'gemini' | 'ollama';

export interface GenerationParams {
  temperature: number;
  topP: number;
  topK: number;
  maxOutputTokens?: number;   // Gemini
  numPredict?: number;        // Ollama
  numCtx?: number;            // Ollama
  repeatPenalty?: number;     // Ollama
  think?: boolean;            // Ollama extended thinking
  thinkingBudget?: number;    // Gemini extended thinking (future)
  // Gemini quota
  geminiPlan?: string;        // e.g. 'TIER_1'; ignored by Ollama
}

export interface OperationConfig {
  operation: LlmOperationName;
  provider: LlmProviderName;
  model: string;
  generation: GenerationParams;
  // Ollama transport
  host?: string;
  requestTimeoutMs?: number;
  warmupTimeoutMs?: number;
  maxAttempts?: number;
  retryBackoffMs?: number;
}
```

---

## Config Service

### `config/llm-config.service.ts`

Reads env vars on first call, caches resolved configs. Validates on startup.

**Env variable schema:**

```bash
# ── AI feature gate (universal) ─────────────────────────────────
AI_FEATURE=true                              # 'true' enables all LLM operations

# ── Gemini global ───────────────────────────────────────────────
GEMINI_API_KEY=...
GEMINI_PLAN=TIER_1                           # global default plan

# ── Operation: generation ────────────────────────────────────────
LLM_GENERATION_PROVIDER=ollama              # 'gemini' | 'ollama'
LLM_GENERATION_MODEL=llama3.2
LLM_GENERATION_HOST=http://localhost:11434  # Ollama only
LLM_GENERATION_TEMPERATURE=0
LLM_GENERATION_TOP_P=0.9
LLM_GENERATION_TOP_K=0
LLM_GENERATION_NUM_CTX=8192                 # Ollama only
LLM_GENERATION_REPEAT_PENALTY=1.05          # Ollama only
LLM_GENERATION_THINK=false                  # Ollama only
LLM_GENERATION_REQUEST_TIMEOUT_MS=300000    # Ollama only
LLM_GENERATION_WARMUP_TIMEOUT_MS=60000      # Ollama only
LLM_GENERATION_MAX_ATTEMPTS=3
LLM_GENERATION_RETRY_BACKOFF_MS=1000
LLM_GENERATION_GEMINI_PLAN=TIER_1           # Gemini only (overrides GEMINI_PLAN)

# ── Operation: localization ──────────────────────────────────────
LLM_LOCALIZATION_PROVIDER=gemini
LLM_LOCALIZATION_MODEL=gemini-3.1-flash-lite-preview
LLM_LOCALIZATION_TEMPERATURE=0.5
LLM_LOCALIZATION_TOP_P=0.95
LLM_LOCALIZATION_TOP_K=64
LLM_LOCALIZATION_THINK=false
LLM_LOCALIZATION_REQUEST_TIMEOUT_MS=300000
LLM_LOCALIZATION_MAX_ATTEMPTS=3
LLM_LOCALIZATION_RETRY_BACKOFF_MS=1000
LLM_LOCALIZATION_GEMINI_PLAN=TIER_1

# ── Operation: suggestion ────────────────────────────────────────
LLM_SUGGESTION_PROVIDER=ollama
LLM_SUGGESTION_MODEL=llama3.2
LLM_SUGGESTION_HOST=http://localhost:11434
LLM_SUGGESTION_TEMPERATURE=0
LLM_SUGGESTION_NUM_CTX=8192
LLM_SUGGESTION_THINK=false
LLM_SUGGESTION_REQUEST_TIMEOUT_MS=300000
LLM_SUGGESTION_WARMUP_TIMEOUT_MS=60000
LLM_SUGGESTION_MAX_ATTEMPTS=3
LLM_SUGGESTION_RETRY_BACKOFF_MS=1000
```

**Backward-compat aliases** consulted when the per-operation var is absent:

```
OLLAMA_HOST → LLM_<OP>_HOST
OLLAMA_MODEL → LLM_<OP>_MODEL
OLLAMA_REQUEST_TIMEOUT_MS → LLM_<OP>_REQUEST_TIMEOUT_MS
OLLAMA_WARMUP_TIMEOUT_MS → LLM_<OP>_WARMUP_TIMEOUT_MS
AI_TRANSLATE_HOST → LLM_LOCALIZATION_HOST
AI_TRANSLATE_MODEL → LLM_LOCALIZATION_MODEL
AI_PRODUCT_HOST → LLM_GENERATION_HOST
AI_PRODUCT_MODEL → LLM_GENERATION_MODEL
AI_TRANSLATE_TRANSLATION_MAX_ATTEMPTS / AI_TRANSLATION_MAX_ATTEMPTS → LLM_LOCALIZATION_MAX_ATTEMPTS
AI_TRANSLATE_TRANSLATION_RETRY_BACKOFF_MS → LLM_LOCALIZATION_RETRY_BACKOFF_MS
```

Exposed API:
```ts
resolveOperation(name: LlmOperationName): OperationConfig
isAiFeatureEnabled(): boolean
```

---

## Provider Interface

### `providers/llm-provider.interface.ts`

```ts
export interface ILlmProvider {
  readonly providerName: string;
  generate(prompt: LlmPrompt, config: OperationConfig): Promise<LlmGenerateResponse>;
  warmup?(config: OperationConfig): Promise<void>;
}
```

---

## Gemini Module & Service

### `providers/gemini/gemini.module.ts`

```ts
@Global()
@Module({
  providers: [GeminiService],
  exports: [GeminiService],
})
export class GeminiModule {}
```

### `providers/gemini/gemini.service.ts`

**Extracted from `ai.service.ts`:**
- One `GoogleGenAI` client per API key (cached in `Map<apiKey, GoogleGenAI>`).
- Uses `@google/genai` exclusively. `@google/generative-ai` removed.
- System prompt passed as `config.systemInstruction` in the `@google/genai` request (native support).
- Rate limiting: Redis-backed per-minute request + token quota, per-day request quota.
  - Key: `gemini:quota:global:<model>:<plan>` — plan comes from `OperationConfig.generation.geminiPlan`.
  - `waitForQuota(estimatedTokens, model, plan)` — blocks until quota is available.
  - `trackTokenUsage(tokens, model, plan)` — updates quota after the call.
- `generate(prompt, config)` returns `LlmGenerateResponse` with `apiCallsCount: 1`.
- `think` flag from prompt is ignored (Gemini manages its own thinking config separately).

**No longer in `ai.service.ts`:**
- `genAI` / `genAiClient` fields.
- `executeGeminiRequest` method.
- `waitForGeminiQuota` / `trackGeminiTokenUsage` methods.

---

## Ollama Module & Service

### `providers/ollama/ollama.module.ts`

```ts
@Global()
@Module({
  providers: [OllamaService],
  exports: [OllamaService],
})
export class OllamaModule {}
```

### `providers/ollama/ollama.service.ts`

**Ported and refactored from `llm.service.ts`:**
- Client pool: `Map<host::requestTimeoutMs, Ollama>`.
- Custom timeout-aware `fetch` wrapper (unchanged logic).
- Does **not** query `LlmConfigService` directly (avoids circular dependency: `LlmModule` → `OllamaModule` → `LlmModule`). Warmup is instead triggered by `LlmOrchestratorService.onApplicationBootstrap` (see Orchestrator section). `OllamaService` exposes `warmup(config: OperationConfig): Promise<void>` which the orchestrator calls per relevant operation.
- `generate(prompt, config)` — applies `numCtx`, `temperature`, `topP`, `topK`, `repeatPenalty`, `think` from `config.generation`; returns `LlmGenerateResponse` with `apiCallsCount: 1`.
- **Provider-level retry** (transport only): `ECONNRESET`, `ETIMEDOUT`, `AbortError`, 5xx — bounded by `config.maxAttempts`. Uses existing `isTransientError` logic.
- `think` flag in `LlmPrompt` is passed directly to Ollama's `think` field.

**What moves out vs stays:**
- Retry for transport errors: stays in `OllamaService`.
- Retry for empty/malformed LLM response: moves to orchestrator.
- `generateTranslationResponseWithRetry` becomes generic `generateWithRetry` (not operation-specific).
- The 3 hard-coded operation configs become fully dynamic from `LlmConfigService`.

**Error type**:
`OllamaService` throws `LlmProviderError` on all failures (same as now). The orchestrator catches and re-throws or retries.

---

## Unified Prompt Builder

### `prompts/prompt-builder.service.ts`

Replaces both `api/src/core/llm/prompt-builder.ts` and `api/src/core/base/promptBuilder.ts`.
All commented-out functions in `promptBuilder.ts` are deleted, not preserved.
All methods return `LlmPrompt` (`{ system, user, format, think }`).

```ts
@Injectable()
export class PromptBuilderService {
  // generation operation
  buildProductPrompt(input: GenerationInput): LlmPrompt;
  buildFamilyAttributesPrompt(input: FamilyAttributesInput): LlmPrompt;

  // localization operation
  buildLocalizationPrompt(input: LocalizationInput): LlmPrompt;
  buildAttributeLocalizationPrompt(input: AttributeLocalizationInput): LlmPrompt;

  // suggestion operation
  buildSuggestionPrompt(input: SuggestionInput): LlmPrompt;
}
```

Each method's content is identical to the current prompt content — only the return type changes from `string` or old `LlmPromptPayload` to `LlmPrompt`.

**Prompt files under `prompts/`:**

| File | Contains |
|---|---|
| `generation.prompt.ts` | Product content prompt + family attributes prompt |
| `localization.prompt.ts` | Text localization prompt + attribute localization prompt |
| `suggestion.prompt.ts` | Amazon attributes prompt + future suggestion prompts |

`PromptBuilderService` imports from these files. The old naming "translation" is replaced everywhere — method names, comments, example strings — with "localization".

---

## Orchestrator

### `orchestrator/llm-orchestrator.service.ts`

The **only** class that business code ever imports from the LLM layer.

```ts
@Injectable()
export class LlmOrchestratorService implements OnApplicationBootstrap {
  constructor(
    private readonly config: LlmConfigService,
    private readonly promptBuilder: PromptBuilderService,
    private readonly factory: LlmProviderFactory,
    private readonly ollamaService: OllamaService,  // direct inject for warmup only
    private readonly sequelize: Sequelize,
  ) {}
```

#### Warmup on Bootstrap

```ts
async onApplicationBootstrap(): Promise<void> {
  const ops: LlmOperationName[] = ['generation', 'localization', 'suggestion'];
  const seen = new Set<string>();
  for (const op of ops) {
    const config = this.config.resolveOperation(op);
    if (config.provider === 'ollama') {
      const key = `${config.host}::${config.model}`;
      if (!seen.has(key)) {
        seen.add(key);
        await this.ollamaService.warmup(config);
      }
    }
  }
}
```

Deduplicates by `host::model` so a shared model across two operations warms up only once. If all operations use Gemini, no warmup runs. `OllamaService` is injected directly into the orchestrator (in addition to via the factory) for this purpose.

#### AI Feature Gate

```ts
private assertFeatureEnabled(): void {
  if (!this.config.isAiFeatureEnabled()) {
    throw new LlmProviderError('AI feature is disabled', {
      type: 'PROVIDER_FAILURE',
      code: 'AI_FEATURE_DISABLED',
      provider: 'none',
      model: null,
      retryable: false,
    });
  }
}
```

Called as the first line of every public method. Replaces the existing per-provider flag check in `ai.service.ts`.

#### Core `generate` Method

```ts
async generate(
  operation: LlmOperationName,
  prompt: LlmPrompt,
  options?: { accountId?: number },
): Promise<LlmGenerateResponse> {
  this.assertFeatureEnabled();
  const opConfig = this.config.resolveOperation(operation);
  const provider = this.factory.resolve(opConfig.provider);

  const result = await this.generateWithCommonRetry(provider, prompt, opConfig);

  if (options?.accountId) {
    await this.recordUsage(options.accountId, result);
  }

  return result;
}
```

#### Common Retry (orchestrator level)

Catches non-transport failures (empty response, complete provider failure that the provider itself could not retry) and retries up to `config.maxAttempts`. Transport-specific errors from providers are already retried inside the provider and arrive here as final failures.

```ts
private async generateWithCommonRetry(
  provider: ILlmProvider,
  prompt: LlmPrompt,
  config: OperationConfig,
): Promise<LlmGenerateResponse> {
  let lastError: unknown;
  for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
    try {
      return await provider.generate(prompt, config);
    } catch (error) {
      lastError = error;
      if (!this.isRetryableAtOrchestratorLevel(error) || attempt >= config.maxAttempts) {
        throw error;
      }
      await this.delay(this.getBackoffMs(attempt, config));
    }
  }
  throw lastError;
}
```

`isRetryableAtOrchestratorLevel` returns `true` only for `LlmProviderError` instances where `details.retryable === true` and the error was not already exhausted inside the provider.

#### Atomic Billing

```ts
private async recordUsage(
  accountId: number,
  result: LlmGenerateResponse,
): Promise<void> {
  await this.sequelize.query(`
    UPDATE account_configurations
    SET consumed_ai_token = jsonb_build_object(
      'input',    COALESCE((consumed_ai_token->>'input')::bigint,    0) + :input,
      'output',   COALESCE((consumed_ai_token->>'output')::bigint,   0) + :output,
      'apiCalls', COALESCE((consumed_ai_token->>'apiCalls')::bigint, 0) + :apiCalls
    )
    WHERE account_id = :accountId
  `, {
    replacements: {
      accountId,
      input: result.inputTokenCount ?? 0,
      output: result.outputTokenCount ?? 0,
      apiCalls: result.apiCallsCount,
    },
    type: QueryTypes.UPDATE,
  });
}
```

- Handles `consumed_ai_token IS NULL` (COALESCE returns 0).
- Handles `consumed_ai_token = {}` (missing keys return null → COALESCE 0).
- Single atomic statement, no read-modify-write race.

#### Typed Convenience Methods

All accept optional `accountId` for billing; all call the core `generate` method.

```ts
async generateProduct(input: GenerationInput, accountId?: number): Promise<LlmGenerateResponse>;
async suggestFamilyAttributes(input: FamilyAttributesInput, accountId?: number): Promise<LlmGenerateResponse>;
async localizeText(input: LocalizationInput, accountId?: number): Promise<LlmGenerateResponse>;
async localizeAttributes(input: AttributeLocalizationInput, accountId?: number): Promise<LlmGenerateResponse>;
async suggestAmazonAttributes(input: SuggestionInput, accountId?: number): Promise<LlmGenerateResponse>;
```

### `orchestrator/llm-provider-factory.service.ts`

```ts
@Injectable()
export class LlmProviderFactory {
  constructor(
    private readonly gemini: GeminiService,
    private readonly ollama: OllamaService,
  ) {}

  resolve(name: LlmProviderName): ILlmProvider {
    if (name === 'gemini') return this.gemini;
    if (name === 'ollama') return this.ollama;
    throw new Error(`Unknown LLM provider: ${name}`);
  }
}
```

---

## Updated `LlmModule`

```ts
@Global()
@Module({
  imports: [
    GeminiModule,
    OllamaModule,
    forwardRef(() => DatabaseModule),  // for Sequelize injection in orchestrator
  ],
  providers: [
    LlmConfigService,
    PromptBuilderService,
    LlmProviderFactory,
    LlmOrchestratorService,
  ],
  exports: [LlmOrchestratorService, PromptBuilderService],
  controllers: [LlmController],
})
export class LlmModule {}
```

---

## Updated `LlmController`

```ts
@Controller('llm')
export class LlmController {
  constructor(private readonly orchestrator: LlmOrchestratorService) {}

  @Post('generateProduct')
  generateProduct(@Body() body: GenerationInput) {
    return this.orchestrator.generateProduct(body);
  }
}
```

No `accountId` on this endpoint (unauthenticated internal tool endpoint — billing not applicable here).

---

## Changes to `ai.service.ts`

### Removed

| Removed | Moved to |
|---|---|
| `genAI` / `genAiClient` fields | `GeminiService` |
| `geminiModel`, `geminiPlan`, `googleApiKey`, `searchEngineId` | `LlmConfigService` / `GeminiService` |
| `executeGeminiRequest` | `GeminiService.generate` |
| `waitForGeminiQuota` / `trackGeminiTokenUsage` | `GeminiService` |
| `translateTextWithUsageFallback` | replaced by `llmOrchestrator.localizeText` |
| `aiFeatureEnabled` field + check | `LlmConfigService.isAiFeatureEnabled()` in orchestrator |
| Account billing inline code | `LlmOrchestratorService.recordUsage` |
| `setKey('AiPayloadForLabel')` + `setKey('AiResponse')` | deleted |

### Kept in `ai.service.ts`

- All product post-processing (family hydration, advisor output normalization, etc.).
- `parseAiJsonResponse` / `sanitizeJsonString`.
- Localization chunking + output validation (simplified — see below).
- `isValidImage`.

### Injection Change

```ts
// Before
private readonly llmService: LlmService

// After
private readonly llmOrchestrator: LlmOrchestratorService
```

### Simplified Localization Flow

**Current complexity being replaced:**

`translateChunkWithValidation` (retry loop + telemetry) + `translateChunkWithSplitFallback` (recursive halving) + `translateTextWithUsageFallback` (dual-path shim) = ~200 lines of interleaved logic.

**Replacement — `runLocalizationChunk`:**

```ts
private async runLocalizationChunk(
  entries: LocalizationEntry[],
  languages: string[],
  accountId: number,
): Promise<Record<string, Record<string, string>>> {
  for (let attempt = 1; attempt <= 2; attempt++) {
    const result = await this.llmOrchestrator.localizeText(
      { inputText: Object.fromEntries(entries), languages },
      accountId,
    );
    try {
      const parsed = this.parseAiJsonResponse(result.response);
      return this.validateLocalizationOutput(parsed, languages, entries);
    } catch (error) {
      if (attempt >= 2) throw error;
      // retry once on parse/validation failure
    }
  }
}
```

- Max 2 attempts per chunk (fixed, not env-driven — validation retry is simple, not configurable).
- `validateLocalizationOutput` (was `validateTranslationChunkOutput`) retained — it's genuinely useful correctness logic.
- No recursive splitting on validation failure. Chunk size is controlled upstream by `buildLocalizationChunks`.
- `buildLocalizationChunks` (was `buildTranslationChunks`): flat batch size only (`LLM_LOCALIZATION_MAX_KEYS_PER_CHUNK` or `DEFAULT_AI_LOCALIZATION_BATCH_SIZE`). The complex budget estimation (`effectiveContextChars`, `expansionFactor`, etc.) is removed.

**`localizeText` method** (was `translateText`):

```ts
async localizeText(
  req: AuthenticatedRequest,
  inputText: Record<string, string>,
  languages: string[],
  context?: { productId?: number; channelId?: number; externalId?: string; source?: string },
  batchSize?: number,
): Promise<any>
```

Same shape but:
- `__attempt` retry context removed (queued retry was using this; the orchestrator handles retry now).
- `translateTextWithUsageFallback` call replaced with `runLocalizationChunk`.
- Billing delegated to orchestrator (no explicit account config update after the call).

---

## Naming Convention: `translation` → `localization`

All occurrences of "translation" in the LLM layer and `ai.service.ts` become "localization". Applied to:

| Before | After |
|---|---|
| `translateText` | `localizeText` |
| `translateTextWithUsage` | `localizeTextWithUsage` |
| `TranslationEntry` | `LocalizationEntry` |
| `TranslationChunk` | `LocalizationChunk` |
| `buildTranslationChunks` | `buildLocalizationChunks` |
| `validateTranslationChunkOutput` | `validateLocalizationOutput` |
| `mergeTranslationResults` | `mergeLocalizationResults` |
| `logAiTranslationChunk` | `logAiLocalizationChunk` |
| `getAiTranslationMaxAttempts` | removed (simplified) |
| `getAiTranslationRetryBackoffMs` | removed (simplified) |
| `AiTranslationValidationError` | `AiLocalizationValidationError` |
| Error code `AI_TRANSLATION_FAILED` | `AI_LOCALIZATION_FAILED` |
| Error code `AI_TRANSLATION_TIMEOUT` | `AI_LOCALIZATION_TIMEOUT` |
| Log event `ollama_translation_failed` | `ollama_localization_failed` |
| Env `AI_TRANSLATION_MAX_ATTEMPTS` | `LLM_LOCALIZATION_MAX_ATTEMPTS` (compat alias kept) |
| `buildTranslateTextPrompt` | `buildLocalizationPrompt` |
| Prompt method/file name `translation.prompt.ts` | `localization.prompt.ts` |

**Scope**: `api/src/core/llm/` and `api/src/modules/app/ai/ai.service.ts`. External API response keys (e.g. `translation_success` response message keys) are outside this scope — only code internals are renamed.

---

## Retry Policy Summary

| Layer | Handles | Mechanism |
|---|---|---|
| `OllamaService` | Transport: `ECONNRESET`, timeout, 5xx, `fetch failed` | `generateWithRetry`, bounded by `config.maxAttempts` + `retryBackoffMs` |
| `GeminiService` | Quota: rate limit waits | `waitForQuota` blocks before each call |
| `LlmOrchestratorService` | Retryable `LlmProviderError` that survived provider | `generateWithCommonRetry`, same attempt budget |
| `ai.service.ts` | Localization output shape errors: parse fail, missing keys | 2-attempt fixed retry in `runLocalizationChunk` |

No layer retries errors that belong to another layer.

---

## End-to-End Implementation Order

### Phase 1 — Types and Error Move

1. Create all files under `types/`.
2. Create `providers/llm-provider.interface.ts`.
3. Move `llm-errors.ts` → `errors/llm-errors.ts`. Update all import paths.

### Phase 2 — Config Service

4. Create `config/llm-config.service.ts` with full env resolution and backward-compat aliases.

### Phase 3 — Gemini Provider

5. Create `providers/gemini/gemini.service.ts`: extract quota logic from `ai.service.ts`, switch to `@google/genai` exclusively.
6. Create `providers/gemini/gemini.module.ts`.

### Phase 4 — Ollama Provider

7. Create `providers/ollama/ollama.service.ts`: port from `llm.service.ts`, dynamic config from `LlmConfigService`, warmup only for Ollama-configured operations.
8. Create `providers/ollama/ollama.module.ts`.

### Phase 5 — Unified Prompt Builder

9. Create `prompts/generation.prompt.ts`, `prompts/localization.prompt.ts`, `prompts/suggestion.prompt.ts` — extract content from both old builders, apply localization naming.
10. Create `prompts/prompt-builder.service.ts`.

### Phase 6 — Orchestrator

11. Create `orchestrator/llm-provider-factory.service.ts`.
12. Create `orchestrator/llm-orchestrator.service.ts` — feature gate, `generate`, atomic billing, typed methods.

### Phase 7 — Wire `LlmModule`

13. Rewrite `llm.module.ts`.
14. Update `llm.controller.ts` to inject `LlmOrchestratorService`.

### Phase 8 — Migrate `ai.service.ts`

15. Remove Gemini client fields and extracted methods.
16. Replace `this.llmService.*` calls with `this.llmOrchestrator.*` calls, passing `req.user.accountId`.
17. Remove debug cache writes (`AiPayloadForLabel`, `AiResponse`).
18. Simplify localization flow: replace `translateChunkWithValidation` + split fallback with `runLocalizationChunk`.
19. Apply full naming rename (`translation` → `localization` everywhere in this file).
20. Remove `@google/generative-ai` import.

### Phase 9 — Cleanup

21. Delete `api/src/core/llm/llm.service.ts`.
22. Delete `api/src/core/llm/prompt-builder.ts`.
23. Delete `api/src/core/base/promptBuilder.ts` (all commented code gone, not preserved).
24. Confirm no remaining import of `LlmService` anywhere in the codebase, then delete the class.
25. Audit `ai.service.ts` for any remaining "translation" occurrences in variable names, error codes, and log keys.

---

## Design Principles Summary

| Principle | Enforcement |
|---|---|
| Single entry point | All LLM calls go through `LlmOrchestratorService` |
| Universal feature gate | Orchestrator throws `AI_FEATURE_DISABLED` if `AI_FEATURE != true` |
| Provider agnostic | Business code never imports `GeminiService` or `OllamaService` |
| Configuration driven | Provider + model + all params resolved from env; switch = one env var change |
| One prompt per use case | Unified `prompts/` directory; no duplicate definitions |
| Atomic billing | Single SQL statement per generate call; no read-modify-write race |
| Retry isolation | Each layer retries only its own failure category |
| No dead code | All commented-out prompts and debug writes removed |
| Clean migration | Old `LlmService` hard-deleted after all call sites confirmed migrated |
