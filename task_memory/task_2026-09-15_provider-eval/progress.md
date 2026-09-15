## Modification History

| Date | Summary of Changes |
|---|---|
| 2026-09-15 | Diagnosed stdin/provider dispatch and implemented provider registry. |

- `codex exec --help` confirmed prompt may be passed as positional `[PROMPT]`.
- Original invocation reproduced `Reading prompt from stdin...`.
- Added `scx`, `cx`, and `codex-chatgpt` provider selection and aggregate output.
- Added per-request timeout and positional prompt dispatch.
- Updated `codex-chatgpt` to run `vpn start` before each request; failures and timeout are surfaced as provider errors.
- Verified `vpn start` returned `state=running` on 2026-09-15.
- Rechecked provider isolation: `scx` uses `/data/ycfeng/stepcode-codex-home/.stepcode/codex` and `/data/ycfeng/stepcode-codex-home/bin/stepcode-codex`; `cx` uses `/data/ycfeng/codex-home`; `codex-chatgpt` uses `/data/ycfeng/codex-home-chatgpt` with the `chatgpt` profile. Explicitly fixed `cx` `CODEX_HOME` to prevent shell inheritance.
