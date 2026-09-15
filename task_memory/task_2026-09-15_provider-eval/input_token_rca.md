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

## Deeper attribution

The fixed offsets can be reproduced from the two prompts alone. `cx - scx` is `19,775 - 13,555 = 6,220` for the one-line prompt and `19,966 - 13,746 = 6,220` for the candy prompt. `codex-chatgpt - cx` is `21,555 - 19,775 = 1,780` and `21,746 - 19,966 = 1,780`. Because changing only the user prompt changes every provider by the same `191` tokens, these offsets are injected before the user content and are unrelated to the candy text, answer length, or reasoning.

The local launcher contributes no provider-specific prompt text. `codex_candy_eval.py` appends the identical positional prompt, uses the same `gpt-6-astra` and `model_reasoning_effort=low`, disables memories, removes HUD variables, and sets `stdin=DEVNULL`. Its parser copies the last `turn.completed.usage` object verbatim. Therefore the divergence occurs inside each CLI's request builder or upstream gateway accounting.

There are three concrete sources of envelope divergence visible on disk:

1. **Different runtime instruction catalogs.** `scx` loads `/data/ycfeng/stepcode-codex-home/.stepcode/codex/stepcode-base-instructions.json`; its `gpt-6-astra` instruction string is 21,261 bytes. The npm CLI is 0.154.0 and uses its compiled OpenAI Codex instruction catalog. These are different text and potentially different tool-policy sections, even though both identify as Codex.
2. **Different home configuration and enabled extensions.** `scx` selects `model_provider=stepcode-api`, `https://models-proxy.stepfun-inc.com/v1`, a 200,000-token context, and only the MCP entries in its StepCode config. `cx` selects `model_provider=stepcode`, `https://subworker.bestony.com`, a 250,000-token context, `service_tier=fast`, and enables the `i-have-adhd` plugin plus its marketplace metadata. `codex-chatgpt` inherits the npm home setup but layers `chatgpt.config.toml`, changing the provider to `openai` and `service_tier=default`; it has its own home state and the same plugin declarations.
3. **Provider accounting/cache semantics.** The usage records are not shaped the same: `scx` reports `cached_input_tokens=0` and `cache_write_input_tokens=13,552`; `cx` reports `cached_input_tokens=3,840` and no cache-write count; ChatGPT reports neither. `input_tokens` is the provider's total for its serialized envelope, while cache fields are provider-specific accounting fields. A cache hit or write can therefore alter the decomposition without making the visible prompt different.

The exact token contribution of each individual instruction/tool block cannot be recovered from this repository because none of the three gateways exposes the serialized request in the captured artifacts. The exact, evidence-backed statement is that the gateway receives three different pre-prompt envelopes and reports them with different cache accounting. Capturing redacted outgoing Responses payloads (or gateway request logs) is required to split the 6,220 and 1,780 offsets into individual blocks.

The stable 6,220 and 1,780 token offsets across minimal and full prompts prove that the main cause is these provider-specific hidden prefixes and tool/instruction declarations. The different cache fields show that the providers also account for prompt-cache tokens differently. Therefore, the displayed input-token values are valid per-provider usage values, but they are not directly comparable as a pure count of the visible candy prompt.

## Excluded Causes

- The visible prompt is identical for all three calls.
- Model and reasoning effort are identical.
- The Python code passes the prompt as the same positional argument and disables inherited stdin for all providers.
- The difference is present even with the minimal one-line prompt, so the candy table is not responsible.
- The provider table only parses the usage event; it does not alter `input_tokens`.

## Practical Conclusion

The observed difference is expected for these three independent Codex/provider configurations. A cross-provider comparison should treat `input_tokens` as provider-reported usage, and compare each provider against its own repeated baseline. Making the numbers equal would require changing provider/profile/system/tool configuration or normalizing the reported metric, which is outside this analysis request.
