# Project Context

**Mission:** Neovim plugin bridging the editor to DictateYourTerms — zero-latency voice dictation that inserts transcripts at the cursor.

Thin Lua wrapper around the `dyt` CLI binary. Opens a centered floating terminal, runs `dyt --record`, then inserts the clipboard transcript on exit.

## About This Project

- Language: Lua (Neovim plugin)
- Plugin manager compatible: lazy.nvim, packer, vim-plug
- Entry point auto-inits via `plugin/dyt.lua`; user config goes through `M.setup()`

## Key Directories

- `lua/dyt/init.lua` — All plugin logic: setup, float window, dictation lifecycle
- `plugin/dyt.lua` — Auto-init shim (calls `setup({})` if not already called)
- `doc/dyt.txt` — Vimdoc help file (keep in sync with options in `init.lua`)

## Standards

- `M._setup_called` flag makes `setup()` idempotent — `plugin/dyt.lua` must stay a no-op guard
- State is always reset via `reset_state()` before side-effecting operations (e.g., `nvim_put`)
- Transcript source is the system clipboard `+` register — `dyt` CLI writes there on exit
- All user-facing messages are prefixed `[dyt]` and gated by `M._config.notify`
- New options must be added to `defaults` table, the options table in `doc/dyt.txt`, and `README.md`

## Notes

- `dyt-daemon` must already be running; the plugin does not start it
- `dyt` binary must be on PATH in the environment that launches Neovim (not just the shell)
- Headless Neovim (CI, `--headless`) has no UI — `float_opts()` fails fast with a clear error
- Re-entrancy is guarded by `M._recording`; a second keymap press is a no-op with a warning

## Workflow

When implementing multi-part features:
1. Spawn a planner agent (`.claude/agents/planner.md`) with the feature description
2. Review the task description it returns
3. Spawn a builder agent (`.claude/agents/builder.md`) with that task description
4. Builder reports completion status; surface any blockers back to planner

## Additional Documentation

Before specific tasks, read relevant documentation:
- Internals, lifecycle, state machine: `docs/architecture.md`
- Local dev setup, manual testing: `docs/getting-started.md`

Read only what's relevant to your current task.
