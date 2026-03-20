# Architecture

> **AI Context Summary**: dyt.nvim is a single-module Lua plugin (`lua/dyt/init.lua`) that manages a
> dictation lifecycle via three phases: setup → active recording (floating terminal) → transcript insertion.
> All mutable state lives in module-level fields prefixed `M._`; two guards (`_setup_called`,
> `_recording`) prevent double-init and re-entrancy. State must always be reset via `reset_state()`
> **before** side-effecting Neovim calls to avoid soft-locking on errors.

## Overview

The plugin is intentionally minimal: one Lua module, one auto-init shim, one vimdoc file. There
is no external dependency beyond the `dyt` binary provided by
[DictateYourTerms](https://github.com/nicolasayotte/dictate-your-terms).

```
plugin/dyt.lua          ← auto-init shim (Neovim loads this on startup)
lua/dyt/init.lua        ← all plugin logic
doc/dyt.txt             ← vimdoc help (:h dyt)
```

The auto-init shim (`plugin/dyt.lua`) calls `setup({})` only when the user has not already called
it, making eager configuration via `lazy.nvim opts` and lazy plugin-authored config work
identically.

## Module State

All mutable state is stored in module-level fields on `M` (`lua/dyt/init.lua:3-11`):

```lua
M._setup_called  = false   -- idempotency guard for setup()
M._config        = {}      -- merged user + default config
M._recording     = false   -- re-entrancy guard for start_dictation()
M._origin_win    = nil     -- window handle to return focus to after dictation
M._origin_mode   = nil     -- 'n', 'i', etc. — mode at invocation time
M._float_win     = nil     -- handle of the dictation float window
M._float_buf     = nil     -- buffer backing the float window
M._job_id        = nil     -- channel id returned by jobstart
M._output_file   = nil     -- temp file path passed to dyt -o
```

These fields are reset atomically by `reset_state()` (`lua/dyt/init.lua:49-60`). The reset
cleans up the temp output file and must happen **before** `nvim_put` — if the paste fails
(e.g. read-only buffer), `_recording` must already be `false` so the user can retry without
restarting Neovim.

## Lifecycle

```
User presses keymap
      │
      ▼
start_dictation()
  ├─ Guard: _recording? → WARN, return
  ├─ Save _origin_win, _origin_mode
  ├─ Escape insert mode if active (feedkeys <Esc>)
  ├─ Create scratch buffer (nvim_create_buf)
  ├─ Open centered float (nvim_open_win)
  ├─ Generate _output_file via tempname()
  ├─ Start terminal job: dyt --record --daemon <url> --no-clipboard --output <tempfile>
  │     (jobstart with term = true)
  │     ├─ FAIL → notify error, close_float(), reset_state(), return
  │     └─ OK → set _job_id, _recording = true, notify "Recording..."
  └─ Enter terminal insert mode (startinsert)

User speaks → presses Enter in terminal
      │
      ▼
on_exit() [vim.schedule callback]
  ├─ close_float()
  ├─ Non-zero exit code?
  │     └─ notify error, reset_state(), return
  ├─ Read transcript from _output_file (vim.fn.readfile)
  ├─ File missing or empty? → notify warn/error, reset_state(), return
  ├─ Restore _origin_win focus (nvim_set_current_win)
  ├─ reset_state()          ← BEFORE nvim_put (intentional ordering)
  │     └─ Deletes the temp output file
  └─ Insert lines at cursor (nvim_put)
```

## Configuration Flow

`M.setup(opts)` merges user-supplied opts over `defaults` using `vim.tbl_deep_extend('force', ...)`
(`lua/dyt/init.lua:187-191`). All config is stored in `M._config` for the plugin's lifetime;
there is no runtime reconfiguration.

```lua
local defaults = {
  keymap     = '<leader>v',
  daemon     = 'http://127.0.0.1:3030',
  win_width  = 0.5,
  win_height = 10,
  border     = 'rounded',
  notify     = true,
}
```

`keymap = false` (or `''`) suppresses binding entirely, enabling callers to bind
`M.start_dictation` themselves (`lua/dyt/init.lua:178-185`).

## Float Window Sizing

`float_opts()` (`lua/dyt/init.lua:28-47`) computes a centered float each time a session starts
(not cached), so editor resizes between sessions are handled automatically.

```lua
local ui     = vim.api.nvim_list_uis()[1]   -- FAILS fast if headless (no UI)
local width  = math.floor(ui.width  * M._config.win_width)
local height = M._config.win_height
local row    = math.floor((ui.height - height) / 2)
local col    = math.floor((ui.width  - width)  / 2)
```

Minimum dimensions are clamped to `20×3` to prevent zero-size windows when the editor is very
narrow. In headless Neovim (CI, `--headless`), `nvim_list_uis()` returns an empty table;
`float_opts()` raises an explicit error rather than crashing opaquely.

## Keymap Registration

`register_keymap()` (`lua/dyt/init.lua:178-185`) sets both `n` and `i` mode mappings with
`noremap = true, silent = true`. The `i`-mode mapping relies on `start_dictation()` escaping
insert mode via `feedkeys('<Esc>')` before opening the float — terminal buffers cannot be opened
while in insert mode of a regular buffer.

## Error Handling Invariants

| Failure point | Behaviour |
|---------------|-----------|
| `jobstart` returns ≤ 0 | Error notify, `close_float()`, `reset_state()` |
| `dyt` exits non-zero | Error notify, `reset_state()` |
| Output file missing | Error notify, `reset_state()` |
| Transcript is empty | Warn notify, `reset_state()` |
| `_origin_win` is invalid | Skip focus restore, proceed to paste |
| Headless Neovim (no UI) | `float_opts()` raises immediately with clear message |

In all failure paths `_recording` is set back to `false` before returning so the user can invoke
the keymap again without restarting Neovim.

## Extending the Plugin

When adding features:

1. **New config option** → add to `defaults` (`init.lua`), options table (`doc/dyt.txt`), and
   options table (`README.md`). All three must stay in sync.
2. **New state field** → add to module-level declarations (`init.lua:3-11`) and include in
   `reset_state()` (`init.lua:49-60`).
3. **New public API** → expose on `M`, document in `doc/dyt.txt` under section 7 (Lua API).

## Cross-References

- Getting started / local dev setup: [docs/getting-started.md](./getting-started.md)
- User-facing configuration reference: `doc/dyt.txt` (`:h dyt-options`)
- Installation examples: `README.md`
