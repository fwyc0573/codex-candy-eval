## Modification History

| Date | Summary of Changes |
|---|---|
| 2026-09-15 | Diagnosed stdin/provider dispatch and implemented provider registry. |

- `codex exec --help` confirmed prompt may be passed as positional `[PROMPT]`.
- Original invocation reproduced `Reading prompt from stdin...`.
- Added `scx`, `cx`, and `codex-chatgpt` provider selection and aggregate output.
- Added per-request timeout and positional prompt dispatch.
