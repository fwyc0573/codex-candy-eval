## Modification History

| Date | Summary of Changes |
|---|---|
| 2026-09-15 | Recorded dispatch regression checks. |

## Execution

- Command: `python3 -m py_compile codex_candy_eval.py`
- Command: `python3 codex_candy_eval.py --help`
- Command: `timeout 35s python3 codex_candy_eval.py -p scx -m gpt-6 -r high -n 1`
- Environment: system Python 3, Codex CLI 0.154.0.

## Criteria and Evidence

- PASS: Python compilation and CLI help expose `--provider {all,scx,cx,codex-chatgpt}` and `--timeout`.
- PASS: Output has a Provider column and runs each requested provider/test count.
- PASS: Prompt is supplied as a positional argument, removing the original stdin-only path.
- LIMITATION: live `scx` execution could not complete within 35 seconds because its provider process remained pending; this is reported by the per-request timeout in normal runs.

## Parallel and Layout Verification

- Command: `python3 -m py_compile codex_candy_eval.py && git diff --check`
- Timing harness: three provider jobs each slept for 0.25 seconds; total elapsed time was 0.25 seconds, confirming concurrent submission.
- Real command: `timeout 150s python3 codex_candy_eval.py -m gpt-6-astra -r low -n 1 --timeout 120`
- Real result: `scx ✓`, `cx ✗`, `codex-chatgpt ✓`; `Graded 3/3`, `correct=2`, `accuracy=66.7%`.
- Layout result: output contains Provider, Run, token, timing, TPS, and OK columns; the long Codex answer column is removed and rows remain readable.

## Latest Result Artifact

- The script writes the latest completed run to `/data/ycfeng/codex-iq.md`.
- The artifact uses Markdown tables and includes run configuration plus the same summary shown in the terminal.
