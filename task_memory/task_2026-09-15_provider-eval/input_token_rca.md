## Modification History

| Date | Summary of Changes |
|---|---|
| 2026-09-15 | Recorded read-only RCA for provider input-token differences. |

# Input Token Difference RCA

## Scope

This is a read-only analysis. No implementation or configuration was changed.

The same raw prompt, model `gpt-6-astra`, and reasoning effort `low` were compared across the three provider paths.

## Observed Evidence

| Provider | Raw CLI | CODEX_HOME / profile | Provider endpoint | Minimal prompt input tokens | Full prompt input tokens |
|---|---|---|---|---:|---:|
| `scx` | `stepcode-codex` | `/data/ycfeng/stepcode-codex-home/.stepcode/codex` | `https://models-proxy.stepfun-inc.com/v1` (`stepcode-api`) | 13,555 | 13,746 |
| `cx` | npm Codex CLI 0.154.0 | `/data/ycfeng/codex-home` | `https://subworker.bestony.com` (`stepcode`) | 19,775 | 19,966 |
| `codex-chatgpt` | npm Codex CLI 0.154.0 | `/data/ycfeng/codex-home-chatgpt`, profile `chatgpt` | profile provider `openai` | 21,555 | 21,746 |

The minimal prompt was `Reply with exactly 21.`. The full prompt was the unchanged `CODEX_PROMPT` in the repository. The minimal raw calls returned `21` with exit code 0.

The full all-provider run used one request per provider and produced `scx=13,746`, `cx=19,966`, and `codex-chatgpt=21,746` input tokens.

The full-prompt deltas are:

- `cx - scx = 6,220`
- `codex-chatgpt - cx = 1,780`
- `codex-chatgpt - scx = 8,000`

The minimal-prompt deltas are the same: `6,220`, `1,780`, and `8,000`. This is the strongest evidence that the extra tokens are fixed provider-side or system-side content, rather than the candy question.

The usage metadata also differs:

- `scx`: `cached_input_tokens=0`, `cache_write_input_tokens=13,552`
- `cx`: `cached_input_tokens=3,840`, `cache_write_input_tokens=0`
- `codex-chatgpt`: `cached_input_tokens=0`, `cache_write_input_tokens=0`

## Configuration Differences

`scx` uses a StepCode-specific config with `model_provider=stepcode-api`, a 200,000-token context window, and a StepCode model catalog. `cx` uses the regular Codex config with `model_provider=stepcode`, a 250,000-token context window, service tier `fast`, and the `subworker.bestony.com` endpoint. `codex-chatgpt` layers `chatgpt.config.toml`, which changes the provider to `openai`, sets service tier `default`, and uses its own home.

The regular and ChatGPT homes contain MCP server and plugin declarations; the StepCode home contains a separate MCP configuration and StepCode base-instructions catalog. These settings affect the system/tool declarations assembled before the user prompt.

All three homes also have different instruction/catalog files. The StepCode base-instructions JSON is about 194 KB, while the regular Codex homes use the npm Codex instruction set. The `AGENTS.md` files are not byte-identical in the StepCode home, although the regular and ChatGPT copies are identical.

## Root Cause

The input token count is returned by the selected model provider in the `turn.completed.usage` event. It is not a count performed by this Python table. The three requests use different provider routes and different profile/configuration layers, so the server receives different system/tool/request envelopes before the same user prompt:

1. `scx` uses the StepCode runtime and `stepcode-api` request builder.
2. `cx` uses the npm Codex runtime with the `stepcode` provider adapter and a different system/tool setup.
3. `codex-chatgpt` uses the same npm runtime but the `openai` profile/provider and a separate home.

The stable 6,220 and 1,780 token offsets across minimal and full prompts prove that the main cause is these provider-specific hidden prefixes and tool/instruction declarations. The different cache fields show that the providers also account for prompt-cache tokens differently. Therefore, the displayed input-token values are valid per-provider usage values, but they are not directly comparable as a pure count of the visible candy prompt.

## Excluded Causes

- The visible prompt is identical for all three calls.
- Model and reasoning effort are identical.
- The Python code passes the prompt as the same positional argument and disables inherited stdin for all providers.
- The difference is present even with the minimal one-line prompt, so the candy table is not responsible.
- The provider table only parses the usage event; it does not alter `input_tokens`.

## Practical Conclusion

The observed difference is expected for these three independent Codex/provider configurations. A cross-provider comparison should treat `input_tokens` as provider-reported usage, and compare each provider against its own repeated baseline. Making the numbers equal would require changing provider/profile/system/tool configuration or normalizing the reported metric, which is outside this analysis request.
