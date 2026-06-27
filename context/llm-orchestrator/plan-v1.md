# LLM Orchestration Layer - Design (V1)

## Objective

Create a centralized LLM orchestration layer that provides a common interface for all AI operations in the application. The implementation should abstract provider-specific logic (Gemini, Ollama, etc.) and allow providers, models, prompts, and generation settings to be configured externally without modifying business logic.

---

# Goals

* Single entry point for all LLM operations.
* Business logic should never depend on a specific provider.
* Easily switch providers/models using configuration.
* Maintain a single prompt definition for every operation.
* Normalize responses from different providers.
* Make the architecture extensible for future providers and features.

---

# Supported Providers (Phase 1)

* Gemini
* Ollama

Future providers should be pluggable with minimal implementation effort.

Examples:

* OpenAI
* Claude
* Azure OpenAI

---

# High-Level Architecture

```
Business Layer
      │
      ▼
LLM Orchestrator
      │
      ├── Configuration Resolver
      ├── Prompt Resolver
      ├── Provider Factory
      ├── Provider Adapter
      └── Response Normalizer
      │
      ▼
Gemini / Ollama / Future Providers
```

---

# LLM Operations

Each business use case is treated as an operation.

Examples:

* Localization
* Product Content Generation
* Attribute Suggestions
* SEO Generation
* Product Summarization

Business code should invoke the orchestrator by specifying only the operation and input data.

Example:

```
generate({
    operation: "localization",
    input: {...}
})
```

---

# Configuration

Configuration should be externalized.

The following should be configurable without changing code:

* Provider
* Model
* Temperature
* Top P
* Top K
* Max Output Tokens
* Provider-specific parameters
* API credentials
* Base URLs

Environment variables (`.env`) will be used as the source of configuration.

---

# Configuration Structure

Each operation owns its own configuration.

Example:

Localization

* Provider
* Model
* Generation Settings

Product Content

* Provider
* Model
* Generation Settings

Attribute Suggestions

* Provider
* Model
* Generation Settings

Provider-specific settings should also be configurable.

Examples:

Gemini

* topK
* thinkingBudget

Ollama

* repeatPenalty
* numPredict
* numCtx

Unused settings should simply be ignored by providers that do not support them.

---

# Prompt Management

Each operation maintains a single prompt definition.

Every prompt consists of:

* System Prompt
* User Prompt

Example:

Localization

```
System Prompt

You are an expert localization engine...

User Prompt

Translate the following product...

{{productDetails}}
```

The orchestrator is responsible for transforming this prompt into the format required by the selected provider.

Examples:

Gemini

* Combined prompt

Ollama

* System + User

OpenAI

* Chat messages

The business layer should never know how prompts are formatted for individual providers.

---

# Provider Adapter

Each provider implements the same interface.

Responsibilities:

* Build provider request
* Execute API call
* Return provider response

Each provider is responsible only for its own SDK/API.

---

# Provider Factory

Based on configuration, the orchestrator resolves the correct provider implementation.

Example:

```
GeminiProvider

OllamaProvider

OpenAIProvider (future)
```

Business logic never creates provider instances directly.

---

# Request Flow

```
Business Layer

↓

LLM Orchestrator

↓

Load Operation Configuration

↓

Resolve Prompt

↓

Resolve Provider

↓

Execute Provider

↓

Normalize Response

↓

Return Result
```

---

# Standard Request

The orchestrator should accept a common request format.

Contains:

* Operation
* Input Variables
* Optional Overrides (future)

---

# Standard Response

All providers should return a normalized response.

Example fields:

* Generated Content
* Provider
* Model
* Input Tokens
* Output Tokens
* Total Tokens
* Finish Reason
* Execution Time
* Raw Response (optional)

Business logic should consume only this standardized response.

---

# Folder Structure (Proposed)

```
src/

llm/

    orchestrator/

    providers/
        gemini/
        ollama/

    prompts/
        localization/
        productContent/
        attributeSuggestions/

    config/

    types/

    utils/

    telemetry/
```

---

# Future Enhancements

The architecture should support the following without requiring major refactoring:

* Additional LLM providers
* Retry policies
* Provider fallback
* Streaming responses
* Response caching
* Request logging
* Token usage tracking
* Cost estimation
* Rate limiting
* Prompt versioning
* Prompt testing

---

# Design Principles

* Single Responsibility
* Provider Agnostic
* Configuration Driven
* Extensible
* Easy to Maintain
* Minimal Business Layer Knowledge

Business logic should only know:

* Which operation to execute
* Input data

Everything else should be handled internally by the orchestration layer.

---

# Phase 1 Deliverables

* Common LLM Orchestrator
* Gemini Provider
* Ollama Provider
* Provider Factory
* Prompt Resolver
* Configuration Resolver
* Standard Request/Response Models
* Response Normalizer
* Externalized Configuration
* Prompt Repository

---

# Success Criteria

* Switching providers should only require configuration changes.
* Switching models should not require code changes.
* Prompt definitions should exist only once per operation.
* Business logic should remain completely provider-agnostic.
* New providers should be added by implementing a new provider adapter without modifying existing business logic.
