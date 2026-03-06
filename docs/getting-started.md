# Getting Started

> **AI Context Summary**: dyt.nvim requires the `dyt` CLI binary on PATH and a running `dyt-daemon`
> process — the plugin does not manage either. For local development, clone the repo and point
> lazy.nvim at the local directory using `dir =`. There are no build steps; all code is plain Lua
> loaded directly by Neovim.

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Neovim ≥ 0.9 | Needs `vim.fn.termopen`, `nvim_open_win` with `style='minimal'` |
| `dyt` binary on PATH | From [DictateYourTerms](https://github.com/nicolasayotte/dictate-your-terms) |
| `dyt-daemon` running | Default: `http://127.0.0.1:3030` — must be started before invoking the keymap |
| System clipboard | Neovim must have clipboard access (`+` register); `xclip`/`xsel` on Linux |

The plugin fails gracefully if `dyt` is missing (error notify from `termopen`) or if `dyt-daemon`
is not running (`dyt` exits non-zero → error notify). Neither condition crashes Neovim or
corrupts state.

## Local Development Setup

Clone and point lazy.nvim at the local path:

```lua
-- In your Neovim config (e.g. ~/.config/nvim/lua/plugins/dyt.lua)
{
  dir = '/path/to/dyt.nvim',
  opts = {},
}
```

No build steps. Neovim loads `plugin/dyt.lua` on startup, which calls `setup({})` unless you
have already called it via `opts`.

To reload the plugin during a live session after editing `lua/dyt/init.lua`:

```vim
:lua package.loaded['dyt'] = nil
:lua require('dyt').setup({})
```

Note: this leaves the previous keymap bound. Use `vim.keymap.del` if you need a clean slate:

```lua
vim.keymap.del({ 'n', 'i' }, '<leader>v')
package.loaded['dyt'] = nil
require('dyt').setup({ keymap = '<leader>v' })
```

## Manual Testing

There is no automated test suite. Verification is done by running Neovim with the plugin loaded
and exercising the dictation flow:

1. Ensure `dyt-daemon` is running.
2. Open any writable buffer.
3. Press `<leader>v` (or your configured keymap).
4. Confirm the floating terminal opens and `dyt --record` starts.
5. Speak a phrase, press Enter.
6. Confirm the float closes and the transcript appears at the cursor.

**Edge cases to verify after changes:**

- Trigger from insert mode — confirm mode is restored and text inserts cleanly.
- Trigger twice rapidly — second press should show "Already recording." warning.
- Kill `dyt-daemon` before triggering — confirm error notification and clean state reset.
- Trigger with `notify = false` — confirm silent operation, transcript still inserts.

## Updating Vimdoc

`doc/dyt.txt` is the canonical help file (`:h dyt`). Keep it in sync with `lua/dyt/init.lua`
whenever options or the public API change. Neovim generates help tags automatically; you can
regenerate them manually with:

```vim
:helptags doc/
```

After editing `doc/dyt.txt`, verify with `:h dyt-options` in a Neovim session that has the local
plugin loaded.

## Publishing a Release

1. Update `README.md` and `doc/dyt.txt` with any new options or behaviour.
2. Commit with a clear message.
3. Push to `main` — lazy.nvim users pin by tag or commit SHA; no build artifacts are needed.

## Cross-References

- Plugin internals and lifecycle: [docs/architecture.md](./architecture.md)
- Configuration options: `doc/dyt.txt` (`:h dyt-options`) and `README.md`
