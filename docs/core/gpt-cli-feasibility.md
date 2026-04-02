# Feasibility: replacing Gemini API calls with ChatGPT for a GPT CLI

This document summarizes whether this repository can be adapted from Gemini API
calls to ChatGPT/OpenAI API calls.

## Short answer

Yes, but **not as a simple endpoint swap**. The project has a useful provider
boundary (`ContentGenerator`), but much of the runtime logic and type system is
currently bound to `@google/genai` request/response structures and Gemini
features.

## What is already favorable

- There is an abstraction for model invocation via `ContentGenerator`
  (`generateContent`, `generateContentStream`, `countTokens`, `embedContent`).
- Model routing and model-name resolution already allow custom model strings,
  which helps with introducing OpenAI model IDs.

## Main blockers to a direct swap

1. **Provider SDK coupling in core types**
   - `ContentGenerator` and many call sites use `@google/genai` types
     (`GenerateContentParameters`, `GenerateContentResponse`, token and
     embedding types), so the contract is Gemini-shaped.
2. **Gemini-specific chat runtime**
   - The chat engine and turn processing use Gemini `Content`/`Part` semantics,
     including Gemini-specific `functionCall`, `functionResponse`, thought
     handling, and finish reasons.
3. **Gemini-specific auth + env UX**
   - Auth detection and configuration are built around `GEMINI_API_KEY`,
     `GOOGLE_GENAI_USE_GCA`, and Vertex AI modes.
4. **Gemini-specific implementation selection**
   - `createContentGenerator` constructs `GoogleGenAI` directly for key-based
     and Vertex auth paths.

## Suggested migration strategy

### Phase 1: Introduce provider-neutral request/response types

- Define internal, provider-neutral message/tool/result types for streaming,
  tool calls, finish reasons, usage, and citations.
- Update `GeminiChat`/turn handling to depend on these internal types instead of
  `@google/genai` types.

### Phase 2: Add provider adapters

- Keep `ContentGenerator` as the boundary, but make it provider-neutral.
- Implement:
  - `GeminiContentGeneratorAdapter` (existing behavior)
  - `OpenAIContentGeneratorAdapter` (ChatGPT/OpenAI Responses or Chat
    Completions)

### Phase 3: Add provider selection in config

- Add a top-level provider setting (for example `provider: gemini | openai`).
- Add provider-specific env vars and validation paths (for example
  `OPENAI_API_KEY`).

### Phase 4: Tool-call and streaming parity

- Ensure tool invocation maps cleanly between provider schemas and the internal
  tool-call abstraction.
- Normalize streamed text/tool chunks into the same internal event pipeline.

### Phase 5: Rebrand CLI packaging

- Current package/bin naming is Gemini-branded (`@google/gemini-cli`, `gemini`
  binary). For a GPT CLI, add a new package/bin identity and docs path.

## Practical conclusion

- **Feasible:** yes.
- **Effort profile:** medium-to-large refactor, mostly in core runtime types and
  streaming/tool-call translation.
- **Fastest low-risk path:** add an OpenAI adapter behind a provider-neutral
  internal model contract while preserving the existing Gemini adapter.
