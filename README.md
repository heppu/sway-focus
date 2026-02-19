# sway-focus

[![CI](https://github.com/heppu/sway-focus/actions/workflows/ci.yml/badge.svg)](https://github.com/heppu/sway-focus/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/heppu/sway-focus/branch/main/graph/badge.svg)](https://codecov.io/gh/heppu/sway-focus)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Move seamlessly within sway and applications.

## Why

Sway's built-in `focus` command moves between windows, but it has no awareness of
application-internal splits. If you have Neovim with vertical splits inside a tmux
pane, pressing your sway focus key jumps straight to the next sway window -- skipping
over the remaining Neovim splits or tmux panes in that direction.

sway-focus fixes this by trying the innermost application first and only bubbling up
to the next layer when it reaches an edge.

## How it works

1. Get the focused window PID from sway via IPC.
2. Walk the process tree under that PID and detect supported applications.
3. Try hooks **innermost-first** (e.g. nvim before tmux):
   - **Can move?** Ask the application if it has more splits in the requested direction.
   - **Yes** -- tell the application to move focus internally. Done.
   - **No / error** -- at edge, bubble up to the next outer hook.
4. If all hooks are at their edge (or none detected), move sway window focus.
5. After crossing a sway window boundary, detect hooks in the newly focused window
   and call `moveToEdge` with the **opposite** direction so the user enters from the
   correct side.

```
 sway window A                      sway window B
+----------------------------+     +----------------------------+
| tmux                       |     | tmux                       |
| +---------++--------------+|     |+--------------++---------+ |
| |  nvim   ||              ||     ||              ||  nvim   | |
| | [split1]||   pane 2     || --> ||   pane 1     || [split1]| |
| | *split2 ||              ||     ||              ||  split2 | |
| +---------++--------------+|     |+--------------++---------+ |
+----------------------------+     +----------------------------+

  focus right from nvim split2:
    1. nvim: at right edge -> bubble up
    2. tmux: at right edge -> bubble up
    3. sway: move focus to window B
    4. tmux detected in B: moveToEdge(left)
    5. nvim detected in B: moveToEdge(left) -> split1 focused
```

## Supported applications

| Application | Status | Communication |
|-------------|--------|---------------|
| **Neovim**  | Full   | msgpack-RPC via Unix socket |
| **tmux**    | Full   | tmux CLI |
| **VS Code** | Detect only | Not yet implemented |

## Installation

### Download binary

Prebuilt Linux amd64 binaries are available from
[GitHub Releases](https://github.com/heppu/sway-focus/releases):

```sh
curl -Lo sway-focus https://github.com/heppu/sway-focus/releases/latest/download/sway-focus-linux-amd64
chmod +x sway-focus
sudo mv sway-focus /usr/local/bin/
```

### Build from source

Requires [Zig](https://ziglang.org/download/) >= 0.15.2. No other dependencies.

```sh
git clone https://github.com/heppu/sway-focus.git
cd sway-focus
zig build release
# Binary at zig-out/bin/sway-focus-linux-amd64
```

## Usage

```
Usage: sway-focus <left|right|up|down> [options]

Generic focus navigation between sway windows and applications.

Options:
  -t, --timeout <ms>      IPC timeout in milliseconds (default: 100)
  --hooks <hook,hook,...>  Comma-separated hooks to enable (default: all)
                           Available: nvim, tmux, vscode
  -v, --version            Print version
  -h, --help               Print this help

Environment:
  SWAY_FOCUS_DEBUG=1       Enable debug logging to stderr
```

### Sway configuration

Replace your existing focus bindings in `~/.config/sway/config`:

```
bindsym $mod+h exec sway-focus left
bindsym $mod+j exec sway-focus down
bindsym $mod+k exec sway-focus up
bindsym $mod+l exec sway-focus right
```

### Selective hooks

If you only use Neovim (no tmux), you can skip tmux detection:

```
bindsym $mod+h exec sway-focus --hooks nvim left
```

## Configuration

| Variable | Description |
|----------|-------------|
| `SWAYSOCK` | Path to sway IPC socket (set automatically by sway) |
| `SWAY_FOCUS_DEBUG` | Set to `1` to enable debug logging to stderr |
| `XDG_RUNTIME_DIR` | Used to locate Neovim sockets |
| `TMUX_TMPDIR` | Tmux socket directory (defaults to `/tmp`) |

## Development

```sh
# Run tests
zig build test

# Run tests with coverage (requires kcov)
zig build coverage -- --junit zig-out/test-results.xml

# Check formatting
zig fmt --check src/ build.zig test_runner.zig

# Debug build and run
zig build run -- left
```

## License

[MIT](LICENSE)
