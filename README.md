# dyt.nvim

Neovim plugin for [DictateYourTerms](https://github.com/nicolasayotte/dictate-your-terms) — zero-latency voice dictation inside the editor.

Opens a centered floating terminal running `dyt --record`, waits for you to speak and press Enter, auto-closes the float, then inserts the transcript at the cursor. Works from normal and insert mode.

## Requirements

- [`dyt`](https://github.com/nicolasayotte/dictate-your-terms) binary on `PATH` in the environment that launches Neovim.
- `dyt-daemon` running before invoking the keymap.

## Installation

### lazy.nvim — minimal

```lua
{
  'nicolasayotte/dyt.nvim',
  opts = {},
}
```

### lazy.nvim — lazy-loaded by key

The plugin loads only when the keymap is first pressed. Pass `keys` matching
your `keymap` option (default `<leader>v`).

```lua
{
  'nicolasayotte/dyt.nvim',
  keys = {
    { '<leader>v', desc = 'DictateYourTerms: voice dictation', mode = { 'n', 'i' } },
  },
  opts = {},
}
```

### lazy.nvim — full spec with all defaults shown

```lua
{
  'nicolasayotte/dyt.nvim',
  keys = {
    { '<leader>v', desc = 'DictateYourTerms: voice dictation', mode = { 'n', 'i' } },
  },
  opts = {
    keymap     = '<leader>v',             -- trigger key; false to disable
    daemon     = 'http://127.0.0.1:3030', -- dyt-daemon base URL
    win_width  = 0.5,                     -- float width as fraction of editor width
    win_height = 10,                      -- float height in rows
    border     = 'rounded',               -- nvim_open_win border style
    notify     = true,                    -- emit vim.notify status messages
  },
}
```

### lazy.nvim — local clone

```lua
{
  dir = '/path/to/dyt.nvim',
  opts = {},
}
```

<details>
<summary>Other plugin managers</summary>

**packer.nvim**

```lua
use 'nicolasayotte/dyt.nvim'
```

**vim-plug**

```vim
Plug 'nicolasayotte/dyt.nvim'
```

The plugin auto-initialises with defaults on startup via `plugin/dyt.lua`. Call
`require('dyt').setup(opts)` yourself if you want to override options.

</details>

## Configuration

All keys are optional. With lazy.nvim, pass options in `opts`; otherwise call
`require('dyt').setup(opts)` anywhere in your config.

| Option       | Type             | Default                   | Description                                            |
|--------------|------------------|---------------------------|--------------------------------------------------------|
| `keymap`     | `string\|false`  | `'<leader>v'`             | Key bound in normal and insert mode. `false` disables. |
| `daemon`     | `string`         | `'http://127.0.0.1:3030'` | HTTP base URL of the running `dyt-daemon`.             |
| `win_width`  | `number`         | `0.5`                     | Float width as a fraction of the editor width.         |
| `win_height` | `number`         | `10`                      | Float height in rows.                                  |
| `border`     | `string`         | `'rounded'`               | Border style passed to `nvim_open_win`.                |
| `notify`     | `boolean`        | `true`                    | Emit `vim.notify` status messages.                     |

### Custom keymap

Set `keymap = false` and bind `M.start_dictation` yourself:

```lua
{
  'nicolasayotte/dyt.nvim',
  keys = {
    { '<C-r>', function() require('dyt').start_dictation() end, desc = 'DictateYourTerms: voice dictation', mode = { 'n', 'i' } },
  },
  opts = { keymap = false },
}
```

## Behaviour

1. The keymap opens a centered floating terminal and runs `dyt --record`.
2. Speak. Press Enter in the terminal to stop recording.
3. The float closes automatically.
4. The transcript is read from the system clipboard (`+` register) and inserted at the cursor.
5. A re-entrancy guard prevents a second invocation while the float is open.
6. On non-zero exit, an error notification is shown and state is cleaned up.

## License

MIT
