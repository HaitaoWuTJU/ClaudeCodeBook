# Summary of `commands/terminalSetup/`

## Purpose of `terminalSetup/`

Provides a unified terminal setup command that detects the user's terminal environment and installs **Shift+Enter key binding** support for multi-line prompts in terminals that don't natively support the CSI u / Kitty keyboard protocol.

## Contents Overview

| File | Purpose |
|------|---------|
| `terminalSetup.ts` | **Entry point** — lightweight command configuration that conditionally shows/hides itself based on terminal support, lazy-loads the implementation |
| `terminalSetup.tsx` | **Implementation** — platform-specific setup logic for Apple Terminal (plist), VSCode/Cursor/Windsurf (JSON keybindings), and other editors |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                   commands/terminalSetup/                        │
│                                                                 │
│  terminalSetup.ts ────────── lazy-loads ──────► terminalSetup.tsx│
│  (Command config)          import()            (Actual setup)    │
│                                                                 │
│  • Exported as default command                                  │
│  • Reads env.terminal to determine visibility                   │
│  • isHidden: true for native CSI u terminals                   │
│                                                  • Adds to plist │
│                                                  • Modifies     │
│                                                    keybindings  │
└─────────────────────────────────────────────────────────────────┘
```

**Flow**: When the user runs the `terminal-setup` command:

1. `terminalSetup.ts` evaluates `env.terminal` against `NATIVE_CSIU_TERMINALS`
2. If the terminal isn't natively supported, the command becomes visible
3. On execution, `terminalSetup.tsx` is dynamically imported
4. Platform-specific logic installs Shift+Enter support:
   - **Apple Terminal** → modifies `com.apple.Terminal.plist` via PlistBuddy
   - **VSCode/Cursor/Windsurf** → appends to `keybindings.json`
   - **Other editors** → may handle Alacritty, Zed, etc.

## Key Takeaways

1. **Progressive enhancement pattern** — detection happens early (at command registration) to hide unsupported options; setup logic loads only on demand

2. **Five terminals skip setup entirely** — Ghostty, Kitty, iTerm2, WezTerm, and Warp are excluded via `isHidden` logic since they already support CSI u natively

3. **Cross-platform support** — handles macOS plist manipulation, VSCode-compatible editors, and different OS config directories (Windows AppData, macOS ~/Library, Linux ~/.config)

4. **Safe file modifications** — all writes create timestamped/random backups before modification

5. **JSONC comment preservation** — `addItemToJSONCArray()` ensures existing comments in `keybindings.json` aren't lost when appending new entries
