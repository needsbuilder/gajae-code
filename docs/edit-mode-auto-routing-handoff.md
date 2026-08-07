# Automatic edit-mode routing implementation handoff

## Status

Product direction is decided and ready for implementation.

Gajae-Code should default `edit.mode` to `auto`. Automatic mode chooses an edit protocol from the active model family, independent of which provider serves the model. Explicit user configuration always wins.

This handoff supersedes the earlier proposal to keep an unconditional hashline default. `hashline` remains the fallback for unknown models and the explicit emergency override.

## Product decision

Initial built-in mapping:

| Detected model family | Built-in edit mode | Reason |
| --- | --- | --- |
| GPT (`gpt-*`) | `apply_patch` | Matches the OpenAI/Codex patch grammar and native custom-tool support |
| Codex (`*codex*` model names) | `apply_patch` | Matches the model family's editing harness |
| Claude (`claude-*`) | `replace` | Anthropic's editor contract is exact, unique old/new string replacement |
| DeepSeek (`deepseek-*`) | `replace` | Search/replace is the strongest portable default for DeepSeek models |
| Qwen (`qwen-*`) | `replace` | Large Qwen/Coder models generally perform better with search/replace than patch syntax |
| MiniMax (`minimax-*`) | `hashline` | Hash-anchored editing performs materially better than generic patch/replacement in available harness comparisons |
| GLM (`glm-*`) | `hashline` | GLM has high patch-format failure rates and benefits from hash anchors |
| Kimi/Moonshot (`kimi-*`, `moonshot-*`) | `hashline` | Hash anchors outperform generic patch and ordinary replacement in available comparisons |
| Unknown | `hashline` | Safest fail-closed fallback with Gajae-Code's stale-read and Harmony recovery |

Use the existing mode identifier `apply_patch`, not `apply-patch`.

Do not automatically include OpenAI `o1`, `o3`, or `o4` in the GPT rule unless their model ID also identifies them as Codex. They require separate benchmark evidence or an explicit catalog/user override.

## Required precedence

Resolve the active mode in this order:

1. Valid `GJC_EDIT_VARIANT` or legacy `PI_EDIT_VARIANT` environment force.
2. Matching user `edit.modelVariants` rule.
3. Explicit `edit.mode` when it is not `auto`.
4. Model-catalog edit recommendation, when present.
5. Built-in model-family mapping.
6. `hashline` fallback.

Invalid forced environment values must fail fast. A matched but invalid `edit.modelVariants` value must also fail closed with a diagnostic; it must not silently fall through to another mode.

Examples:

```yaml
edit:
  mode: auto
  modelVariants:
    "custom-company/gpt-5.4": hashline
    "local/qwen3-coder-small": hashline
```

```sh
GJC_EDIT_VARIANT=replace gjc
```

The environment force is the emergency kill switch and must beat automatic routing and per-model configuration.

## Configuration types

Keep the execution-mode union separate from the setting value:

```ts
export type EditMode = "replace" | "patch" | "hashline" | "vim" | "apply_patch";
export type EditModeSetting = "auto" | EditMode;
```

Do not add `auto` to APIs that require an executable edit mode. Resolve `auto` before constructing mode-specific tool schemas, prompts, custom wire names, custom grammars, or file-display behavior.

Change the schema default:

```ts
"edit.mode": {
  type: "enum",
  values: EDIT_MODE_SETTINGS,
  default: "auto",
  // ...
}
```

Compatibility behavior:

- Missing `edit.mode` now means `auto`.
- Persisted explicit modes remain authoritative.
- Existing `atom` migrations continue to produce explicit `hashline`.
- Environment variables continue accepting executable modes only; `GJC_EDIT_VARIANT=auto` is unnecessary and should either be rejected or explicitly documented if supported.

## Model identity and custom-provider detection

Routing must be based primarily on the model portion of the normalized model ID, not the provider name. Equivalent models served through different providers should select the same built-in mode:

```text
openai/gpt-5.4
openrouter/openai/gpt-5.4
company-gateway/gpt-5.4
```

All three should resolve to GPT → `apply_patch` unless an earlier rule overrides it.

Likewise:

```text
anthropic/claude-sonnet-4-6
openrouter/anthropic/claude-sonnet-4-6
custom-proxy/claude-sonnet-4-6
```

All should resolve to Claude → `replace`.

### Normalization contract

1. Trim and lowercase the active model string.
2. Split on `/` and treat the final non-empty segment as the model name.
3. Match anchored family tokens in a fixed order.
4. Use catalog metadata before heuristics when the catalog explicitly declares a recommendation.
5. Return `unknown` rather than making a weak substring guess.

Suggested family type:

```ts
export type ModelEditFamily =
  | "gpt"
  | "codex"
  | "claude"
  | "deepseek"
  | "qwen"
  | "minimax"
  | "glm"
  | "kimi"
  | "unknown";
```

Suggested detector shape:

```ts
export function detectModelEditFamily(modelId: string): ModelEditFamily {
  const normalized = modelId.trim().toLowerCase();
  const segments = normalized.split("/").filter(Boolean);
  const model = segments.at(-1) ?? normalized;

  if (/^(?:gpt-|chatgpt-)/.test(model)) return "gpt";
  if (/(?:^|[-_.])codex(?:$|[-_.])/.test(model)) return "codex";
  if (/^claude(?:$|[-_.])/.test(model)) return "claude";
  if (/^deepseek(?:$|[-_.])/.test(model)) return "deepseek";
  if (/^qwen(?:$|[-_.])/.test(model)) return "qwen";
  if (/^minimax(?:$|[-_.])/.test(model)) return "minimax";
  if (/^glm(?:$|[-_.])/.test(model)) return "glm";
  if (/^(?:kimi|moonshot)(?:$|[-_.])/.test(model)) return "kimi";

  return "unknown";
}
```

Treat this as a starting contract, not code to paste without checking the repository's actual canonical model-string format. In particular, verify whether nested IDs such as `openrouter/moonshotai/kimi-k2.5` reach `resolveEditMode` intact or are already normalized by `formatModelString()`.

Avoid unrestricted checks such as `modelId.includes("glm")`; they can misclassify provider names, deployment labels, or unrelated model names.

## Catalog policy

Where the model catalog knows the correct edit protocol, metadata should beat family heuristics:

```ts
export interface ModelEditPolicy {
  recommendedEditMode?: EditMode;
}
```

Use existing `applyPatchToolType` metadata only to choose the OpenAI wire representation after `apply_patch` is selected. It should not by itself override an explicit user mode.

Recommended separation:

- `recommendedEditMode`: which Gajae-Code edit mode to select in `auto`.
- `applyPatchToolType`: how the selected `apply_patch` tool is represented to that model/API.

Do not edit generated `packages/ai/src/models.json` directly. If catalog metadata is added, change the generator or model-policy inference and run `bun run generate-models`.

## Resolver result and provenance

Preserve the existing scalar API for current callers and add a details API:

```ts
export type EditModeSource =
  | "environment"
  | "model-override"
  | "setting"
  | "catalog"
  | "builtin-family"
  | "fallback";

export type ResolvedEditModeDetails = {
  mode: EditMode;
  source: EditModeSource;
  modelId?: string;
  family?: ModelEditFamily;
  matchedRule?: string;
};

export function resolveEditModeDetails(session: EditModeSessionLike): ResolvedEditModeDetails;
export function resolveEditMode(session: EditModeSessionLike): EditMode;
```

`resolveEditMode()` should delegate to `resolveEditModeDetails()` and return `.mode`.

The details result is required for debugging, status output, tests, and future telemetry. It must never include credentials or provider secrets.

## Runtime synchronization

A mode change alters more than execution. Verify every dependent surface is rebuilt from the same resolved mode:

- `EditTool.description`
- `EditTool.parameters`
- `EditTool.customWireName`
- `EditTool.customFormat`
- read/search hashline versus line-number display
- write-time hashline-prefix stripping
- streaming edit preview strategy
- model-switch system-prompt rebuild
- SDK tool signatures and activation refresh

The existing `AgentSession.#syncEditToolModeAfterModelChange()` and prompt-rebuild paths should remain the model-switch owner. Add tests showing that switching from Claude to GPT changes the tool from `replace` to `apply_patch`, and switching to an unknown model changes it to `hashline`.

## Likely implementation files

Primary changes:

- `packages/coding-agent/src/utils/edit-mode.ts`
  - Add `EditModeSetting` and `ModelEditFamily`.
  - Add family detection and built-in mapping.
  - Add catalog-policy lookup if the session/model surface exposes it.
  - Implement precedence and provenance.
- `packages/coding-agent/src/config/settings-schema.ts`
  - Permit `auto` for `edit.mode` and make it the default.
  - Schemaize `edit.modelVariants` as executable edit-mode values if it is still outside the public schema.
- `packages/coding-agent/src/config/settings.ts`
  - Return a discriminated model-override result so invalid matches cannot silently fall through.
  - Preserve existing `atom` → `hashline` migration.
- `packages/coding-agent/src/edit/index.ts`
  - Consume the shared resolver only; remove any separate environment-precedence cache.
- `packages/coding-agent/src/utils/file-display-mode.ts`
  - Continue consuming the resolved executable mode.
- `packages/coding-agent/src/session/agent-session.ts`
  - Verify model changes rebuild the mode-specific tool prompt/schema.
- `packages/coding-agent/src/edit/streaming.ts`
  - Verify automatic routing selects the correct preview strategy.
- `docs/tools/edit.md`
  - Replace the unconditional hashline-default wording with `auto` routing and document the fallback.
- `docs/environment-variables.md`
  - Document force precedence and remove the obsolete `atom` value.
- `packages/coding-agent/CHANGELOG.md`
  - Add an Unreleased entry describing the new default and compatibility behavior.

Possible catalog changes, only if required:

- `packages/ai/src/types.ts`
- `packages/ai/src/model-thinking.ts` or the appropriate model-policy generator
- generated schema/model tests

Do not change model catalog generation until the family-based path works for arbitrary custom-provider model IDs.

## Test matrix

### Unit: family detection

Test provider-independent IDs:

| Model ID | Expected family | Expected built-in mode |
| --- | --- | --- |
| `openai/gpt-5.4` | `gpt` | `apply_patch` |
| `openrouter/openai/gpt-5.4` | `gpt` | `apply_patch` |
| `custom/gpt-oss-120b` | `gpt` | `apply_patch` |
| `openai/gpt-5.3-codex` | `gpt` or `codex` | `apply_patch` |
| `custom/codex-specialized` | `codex` | `apply_patch` |
| `anthropic/claude-sonnet-4-6` | `claude` | `replace` |
| `custom/claude-opus-4-5` | `claude` | `replace` |
| `deepseek/deepseek-v3.2` | `deepseek` | `replace` |
| `custom/qwen3-coder` | `qwen` | `replace` |
| `minimax-code/minimax-m2.5` | `minimax` | `hashline` |
| `zai/glm-4.7` | `glm` | `hashline` |
| `moonshotai/kimi-k2.5` | `kimi` | `hashline` |
| `custom/moonshot-v1-code` | `kimi` | `hashline` |
| `custom/company-code-model` | `unknown` | `hashline` |
| `custom/not-a-glm-model` | `unknown` | `hashline` |

Confirm the expected behavior of `gpt-oss-*`. It matches the requested `gpt-*` rule, even though it may not be trained for OpenAI's native patch harness. Keep the test if that is intentional; otherwise narrow GPT detection to catalog-confirmed OpenAI GPT models.

### Unit: precedence

Cover at least:

- Environment force beats every other source.
- Exact/matching `modelVariants` beats explicit `edit.mode`.
- Explicit non-auto `edit.mode` beats catalog and family mapping.
- Catalog recommendation beats built-in family mapping.
- Built-in mapping beats fallback.
- Unknown model falls back to hashline.
- Invalid environment value fails fast.
- Invalid matched model override fails closed.
- Non-matching invalid entries do not affect another model.

### Integration: model switching

- Claude → GPT: `replace` schema/prompt becomes `apply_patch` schema/prompt.
- GPT → MiniMax: `apply_patch` becomes hashline; read/search output begins emitting hashline anchors.
- MiniMax → Qwen: hashline becomes `replace`; read/search stop emitting hashline anchors.
- Known → unknown: fallback becomes hashline.
- Explicit `edit.mode: replace`: model switching does not change the mode.
- Environment force: model switching does not change the mode.

### Regression

- Existing hashline parser, recovery, read/search anchor, and write-prefix tests remain unchanged when mode resolves to hashline.
- Existing `apply_patch` freeform/function-wire tests remain unchanged when mode resolves to `apply_patch`.
- Existing `atom` migration tests continue passing.
- SDK/public-surface manifests are regenerated only when the intended public type surface changes.

## Focused verification commands

Use repository-supported commands rather than root `tsc`:

```sh
bun test packages/coding-agent/test/settings-manager.test.ts
bun test packages/coding-agent/test/agent-session-default-model-selection.test.ts
bun test packages/coding-agent/test/agent-session-tool-rebuild-skip.test.ts
bun test packages/coding-agent/test/sdk-tool-activation.test.ts
bun test packages/coding-agent/test/tools/read-goldens.test.ts
bun --cwd=packages/coding-agent run check
```

Add and run a focused `edit-mode` test file for family detection and precedence. After schema or generated-surface changes:

```sh
bun run generate-schemas
bun run generate-models  # only if catalog generation changed
bun scripts/check-visible-definitions.ts
bun scripts/verify-g002-gates.ts
bun scripts/rebrand-inventory.ts --strict
bun test packages/coding-agent/test/default-gjc-definitions.test.ts
bun run check:ts
```

Run formatting once after all edits; do not use edit operations to reformat files.

## Rollout and observability

The behavior change affects users who never explicitly configured `edit.mode`:

- GPT/Codex sessions change from hashline to `apply_patch`.
- Claude, DeepSeek, and Qwen sessions change from hashline to `replace`.
- MiniMax, GLM, Kimi, and unknown models remain on hashline.

Expose the resolved mode and source in debug/session diagnostics so routing is explainable:

```text
edit mode: apply_patch (builtin-family: gpt)
edit mode: replace (model-override: custom/claude-opus)
edit mode: hashline (fallback: unknown)
```

Minimum rollout counters by model ID, family, selected mode, and source:

- edit calls,
- first-attempt success,
- malformed-input failures,
- no-match/non-unique failures,
- stale/hash mismatch failures,
- recovery success,
- retry success,
- output/tool-input characters.

Never record source file contents, patch bodies, prompts, credentials, or raw provider responses.

Emergency rollback remains:

```sh
GJC_EDIT_VARIANT=hashline gjc
```

## Acceptance criteria

- Default `edit.mode` is `auto`.
- Explicit environment, per-model, and global settings follow the documented precedence.
- The built-in family table exactly matches the product-decision table above.
- Custom-provider IDs route by normalized model name rather than provider allowlists.
- Unknown models resolve to hashline.
- Model switching updates the edit schema, prompt, custom wire format, file-display mode, and streaming preview coherently.
- Invalid forced or matched values fail closed without filesystem mutation.
- Existing explicit configurations remain behaviorally stable.
- Documentation, schema, changelog, generated artifacts, and tests agree.

## Research sources

- OpenAI Apply Patch: https://developers.openai.com/api/docs/guides/tools-apply-patch
- OpenAI Codex patch instructions: https://github.com/openai/codex/blob/main/codex-rs/core/prompt_with_apply_patch_instructions.md
- Anthropic text editor: https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool
- Anthropic SWE-bench engineering report: https://www.anthropic.com/engineering/swe-bench-sonnet
- Aider edit formats: https://aider.chat/docs/more/edit-formats.html
- Aider model settings: https://raw.githubusercontent.com/Aider-AI/aider/main/aider/resources/model-settings.yml
- Aider leaderboards: https://aider.chat/docs/leaderboards/
- Gajae-Code edit benchmark data: `packages/typescript-edit-benchmark/all_models_results.json`
- Gajae-Code Harmony recovery evidence: `docs/ERRATA-GPT5-HARMONY.md`
