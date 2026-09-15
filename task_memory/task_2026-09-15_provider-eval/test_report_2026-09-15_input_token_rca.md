## Modification History

| Date | Summary of Changes |
|---|---|
| 2026-09-15 | Recorded reproducibility and attribution checks for provider input-token differences. |

# Input-token RCA verification

## Execution

- Working directory: `/data/ycfeng/codex-candy-eval`
- Python: `python3` (used for arithmetic/config inspection)
- Source inspection: `sed -n '1,260p' codex_candy_eval.py`
- Configuration inspection: `python3` read of the three `config.toml` files and `stepcode-base-instructions.json`
- Recorded provider runs from the repository's completed all-provider execution in `progress.md` and `input_token_rca.md`.

## Criteria

The RCA must establish whether the visible prompt or local parser causes the difference, quantify stable offsets, and identify configuration/runtime causes supported by local artifacts.

## Evidence

PASS — Minimal-prompt values were `scx=13,555`, `cx=19,775`, `codex-chatgpt=21,555`; full-prompt values were `13,746`, `19,966`, and `21,746`. The full-minus-minimal increment is `191` for every provider. Fixed offsets are `6,220` (`cx-scx`) and `1,780` (`codex-chatgpt-cx`) in both prompts.

PASS — `codex_candy_eval.py` appends the same positional prompt, passes the same model/effort, disables memories, removes HUD variables, sets `stdin=DEVNULL`, and copies `turn.completed.usage` without recalculation.

PASS — Local configuration proves distinct request paths: `stepcode-api` at `models-proxy.stepfun-inc.com/v1`; npm `stepcode` at `subworker.bestony.com`; ChatGPT profile `openai` with `service_tier=default`. The StepCode `gpt-6-astra` instruction catalog entry is 21,261 bytes and is separate from the npm binary's compiled catalog.

PASS — Cache metadata differs: StepCode reports `cache_write_input_tokens=13,552`; `cx` reports `cached_input_tokens=3,840`; ChatGPT reports neither. This confirms provider-specific usage decomposition.

LIMITATION — No captured redacted HTTP request exists, so individual token counts for each hidden instruction/tool block cannot be separated. Gateway payload logging is required for that finer attribution.
