# quiver-ttyd

Fork of [tsl0922/ttyd](https://github.com/tsl0922/ttyd) for [Quiver](https://github.com/jimweller/quiver) bolt terminals. Adds embedded Nerd Fonts and Shift+Enter multiline input support for Claude Code.

## Patches Over Upstream

| Patch           | Files                                                                            | Purpose                                                                                                      |
| --------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Nerd Fonts      | `html/src/style/webfont/`, `html/src/style/index.scss`, `html/webpack.config.js` | Embeds JetBrains Mono and Sarasa Mono SC as inline woff2 assets with `font-display: block`                   |
| Font preload    | `html/src/components/terminal/index.tsx`                                         | Calls `document.fonts.load('14px JetBrains')` before opening the terminal to prevent FOUT                    |
| Font re-measure | `html/src/components/terminal/xterm/index.ts`                                    | Toggles fontSize after delayed timeouts to force xterm.js character grid re-measurement after late font load |
| Shift+Enter     | `html/src/components/terminal/xterm/index.ts`                                    | Sends `\x1b\r` (ESC+CR) via `attachCustomKeyEventHandler`, blocking both keydown and keypress events         |

### Shift+Enter Signal Chain

```text
Browser keydown -> xterm.js handler -> sendData('\x1b\r') -> websocket -> ttyd server -> PTY -> tmux -> application
```

ESC+CR (`\x1b\r`) was chosen over CSI u (`\x1b[13;2u`) because Claude Code handles ESC+CR unconditionally. CSI u requires kitty keyboard protocol negotiation that xterm.js does not support.

## Build

CI runs on GitHub Actions (`.github/workflows/build.yml`). Single-arch x86_64 build using musl cross-compilation.

```bash
# Frontend (Node 18, yarn 3.6.3)
cd html && yarn install && yarn run build

# Backend (musl cross-compile)
BUILD_TARGET=x86_64 scripts/cross-build.sh
```

## Release

Trigger a release via workflow dispatch with a tag input:

```bash
gh workflow run build.yml --repo jimweller/quiver-ttyd -f tag=1.7.7-quiver.N
```

The build uploads `ttyd.x86_64` to a GitHub release tagged with the input value.

## Usage in Quiver

The Quiver bolt Dockerfile downloads the release binary:

```dockerfile
wget -q -O /usr/sbin/ttyd.nerd \
    "https://github.com/jimweller/quiver-ttyd/releases/download/1.7.7-quiver.4/ttyd.${TTYD_ARCH}"
```

tmux must have `extended-keys always` set to pass ESC+CR through to applications inside tmux sessions.

## Project Structure

```text
quiver-ttyd/
├── .github/workflows/      # CI/CD pipeline
├── html/                    # Frontend (Preact, xterm.js, TypeScript)
│   └── src/
│       ├── components/
│       │   ├── terminal/    # Terminal component with Shift+Enter handler
│       │   └── modal/       # File upload modal
│       └── style/
│           └── webfont/     # Embedded Nerd Font woff2 files
├── src/                     # Backend C source (upstream ttyd)
├── cmake/                   # CMake build configuration
├── scripts/                 # Cross-compilation scripts
└── man/                     # Man pages
```

## Upstream

Forked from [tsl0922/ttyd](https://github.com/tsl0922/ttyd) at v1.7.7. Upstream remote configured as `upstream` for future rebasing.

## License

MIT (inherited from upstream).
