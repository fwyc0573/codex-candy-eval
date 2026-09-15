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
- Corrected dispatch to exclude `codex-hud`: `cx` and `codex-chatgpt` now call the raw npm Codex CLI; `scx` calls raw `stepcode-codex`. HUD environment variables are removed before spawning.
- RCA for hanging `cx`: raw CLI `codex-cli 0.154.0` with `gpt-6-astra` and `model_reasoning_effort=low` returned `boss YC 21` in about 24 seconds. The repo subprocess inherited the caller's stdin, so Codex printed `Reading additional input from stdin...` and could wait for EOF after the positional prompt. Set `stdin=subprocess.DEVNULL`.
- Full repo verification: `python3 codex_candy_eval.py -p cx -m gpt-6-astra -r low -n 1 --timeout 90` returned a parsed answer and token metrics in about 30 seconds; the model answer was `22`, so semantic grading was `OK=✗` while process execution passed.
- StepCode verification: raw `/data/ycfeng/stepcode-codex-home/bin/stepcode-codex` with `gpt-6-astra`, `low`, and a positional prompt returned `21` with exit code 0. The full repo command `python3 codex_candy_eval.py -p scx -m gpt-6-astra -r low -n 1 --timeout 90` returned `OK=✓`, `Graded 1/1`, and `accuracy=100.0%` in 27.1 seconds.
- ChatGPT provider verification: `vpn start` returned `state=running`; raw CLI with `CODEX_HOME=/data/ycfeng/codex-home-chatgpt` and `-p chatgpt` returned `21` with exit code 0. The full repo command `python3 codex_candy_eval.py -p codex-chatgpt -m gpt-6-astra -r low -n 1 --timeout 90` returned `OK=✓`, `Graded 1/1`, `accuracy=100.0%`, 21,639 input tokens, 1,053 output tokens, and 483 reasoning tokens in 50.3 seconds.
- Parallel execution and output verification: all provider/test jobs are submitted to a `ThreadPoolExecutor` before results are collected. A three-job timing harness completed in 0.25 seconds for 0.25-second tasks. A real all-provider run (`gpt-6-astra`, `low`, `n=1`) completed with rows for `scx`, `cx`, and `codex-chatgpt` in 45.4 seconds total; the table omits the long answer column and reports `scx ✓`, `cx ✗`, `codex-chatgpt ✓` with `Graded 3/3`, `correct=2`, `accuracy=66.7%`.
