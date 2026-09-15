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
