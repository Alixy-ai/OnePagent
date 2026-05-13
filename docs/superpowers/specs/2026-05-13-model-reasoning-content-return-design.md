# Model Reasoning Content Return Design

## Goal

Make OpenAI-compatible `reasoning_content` preservation and multi-turn return configurable per model, while preserving the current DeepSeek behavior and supporting the same tool-call history return contract for any model where the setting is enabled.

## Context

OnePagent currently decides whether to preserve and return OpenAI-compatible `reasoning_content` with provider-level DeepSeek detection. This affects:

- Streaming parse behavior: whether reasoning deltas become stored `reasoning` blocks.
- Multi-turn request construction: whether prior assistant reasoning blocks are returned as `reasoning_content`.
- Usage accounting: whether reasoning token details are included.

The new behavior must be bound to model settings, not only provider detection.

## Required return contract

When reasoning-content return is enabled for the current model, the app must use the same multi-turn return method for DeepSeek, MiMo, and any other OpenAI-compatible model that exposes `reasoning_content`.

In agent-style multi-turn conversations, if historical assistant messages contain tool calls, every later user turn must return the complete `reasoning_content` for those assistant messages when it is available. If historical tool-call assistant reasoning is omitted, some providers can return HTTP 400 because the model context is incomplete. Missing reasoning can also reduce instruction following and increase hallucinations.

This is a generic contract for the enabled model, not a MiMo-only rule. The setting means “preserve and return reasoning_content for this model”; once enabled, the return behavior applies to all later request construction for that model.

## Chosen approach

Add per-model reasoning-content return settings.

The setting is model-bound and stored alongside provider profile settings. Each configured model uses one of three modes:

- `auto`: use built-in defaults.
- `on`: preserve and return reasoning content for that model.
- `off`: do not preserve or return reasoning content for that model.

Default effective behavior:

- DeepSeek provider / endpoint / default model / selected model matches `deepseek`: on.
- Selected model matches `xiaomimimo` or the MiMo model family pattern used in this codebase: on.
- Other models: off.

## Architecture

Add a per-model map similar to `MODEL_CONTEXTS`, for example:

```js
const MODEL_REASONING_CONTENT = { ...(CFG.reasoning_content_models || {}) };
```

Values are strings: `auto`, `on`, or `off`.

Add helpers:

```js
function getReasoningContentModeForModel(model = API_MODEL) { ... }
function isDefaultReasoningContentModel(model = API_MODEL) { ... }
function isMimoReasoningModel(model = API_MODEL) { ... }
function shouldKeepReasoningForModel(model = API_MODEL) { ... }
```

`shouldKeepReasoningForModel` returns true when:

1. The model is explicitly set to `on`.
2. The model is `auto` and default detection says DeepSeek or MiMo should preserve reasoning.

It returns false when the model is explicitly set to `off`, or when `auto` detection does not match a reasoning-content model.

Replace direct `isDeepSeekActiveProvider()` usage in reasoning paths with the helper:

- `assembleRequestBodyFromParts()` uses the helper to decide whether assistant reasoning blocks are returned as `reasoning_content`.
- `parseOpenAIStream()` call sites use the helper for `keepReasoning` and `includeReasoningUsage`.
- Sub-agent stream parsing uses the same helper.

The top-bar Think level remains separate. It controls request parameters such as `reasoning_effort`; this setting controls whether returned reasoning content is preserved and sent back in later turns.

## Request construction details

For OpenAI-compatible request assembly:

- If effective keep is on, preserve current behavior: assistant `reasoning` blocks are emitted as `reasoning_content`, tool calls remain attached, and incompatible tool results are skipped when reasoning is missing.
- If effective keep is off, assistant messages are emitted without `reasoning_content`.
- If a historical assistant tool-call message has no stored reasoning block because it was created before this feature, was captured while the model was Off, or was imported from another source, preserve the existing fallback behavior that avoids sending an invalid assistant tool-call message without reasoning; do not invent reasoning text.
- Turning a model On affects subsequent parsed reasoning blocks and subsequent request construction. It cannot restore reasoning that was never stored.

## Settings UI

Add a Settings section near “Model Context Lengths”: “Reasoning Content Return”.

Rows contain:

- Model id.
- Mode select: Auto / On / Off.
- Delete button.

Actions:

- “Import Selected” adds currently selected models. DeepSeek and MiMo models default to `auto` with effective On; other models default to `auto` with effective Off.
- Manual add validates non-empty model id and stores mode.
- Deleting a row removes the explicit model setting and reverts to default detection.

Provider profile save/load should include `reasoning_content_models`. Switching providers refreshes the UI and in-memory map. Cloud Sync/export/import inherit this through existing settings/provider profile persistence.

## Data flow

1. User opens Settings and configures per-model reasoning content mode.
2. Save writes `reasoning_content_models` into the active provider profile/settings.
3. Model selection or provider switching refreshes `MODEL_REASONING_CONTENT`.
4. Request construction calls `shouldKeepReasoningForModel(API_MODEL)`.
5. Stream parsing uses the same effective decision to retain or discard current-round reasoning blocks.
6. Token accounting includes reasoning details only when effective keep is on.

## Error handling and compatibility

- Missing `reasoning_content_models`: use default DeepSeek/MiMo detection.
- Invalid mode values: treat as `auto`.
- Empty model ids: do not save.
- Non-object imported maps: ignore.
- Existing conversations continue to render old reasoning blocks normally.
- If a model is Off and the provider rejects a later request because it expected historical reasoning, the user should enable reasoning-content return for that model and retry. The app should not silently fabricate missing reasoning.

## Testing

Manual and lightweight code-path tests should cover:

1. DeepSeek default: no explicit setting returns effective keep=true.
2. MiMo default: no explicit setting returns effective keep=true.
3. Normal OpenAI model default: effective keep=false.
4. DeepSeek explicit Off: effective keep=false.
5. MiMo explicit Off: effective keep=false.
6. Normal model explicit On: parse stream keeps reasoning block and request assembly returns `reasoning_content`.
7. Settings UI add/save/delete/switch provider preserves `reasoning_content_models`.
8. Multi-turn request assembly: On includes assistant `reasoning_content`; Off omits it.
9. Tool-call history with On: assistant tool-call messages return complete stored `reasoning_content` when available.
10. Stream parse call sites use the same helper in normal chat and sub-agent flows.

## Scope

This change only controls OpenAI-compatible `reasoning_content` preservation and return. It does not change Anthropic extended thinking, top-bar Think level semantics, or the visible rendering of existing reasoning blocks.