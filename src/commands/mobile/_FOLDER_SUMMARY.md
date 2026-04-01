# Summary of `commands/mobile/`

## Purpose of `mobile/`

This subdirectory implements the CLI command for displaying QR codes that allow users to download the Claude mobile app on iOS and Android devices. It provides an interactive terminal UI where users can view and scan QR codes with their phones.

## Contents Overview

| File | Role | Description |
|------|------|-------------|
| `mobile.ts` | Entry point | Command configuration with lazy loading |
| `mobile.tsx` | Implementation | Interactive QR code display component |

## How Files Relate to Each Other

```
mobile.ts                    mobile.tsx
┌──────────────────────┐    ┌─────────────────────────────────────┐
│  Command config      │    │  MobileQRCode component            │
│  - name: "mobile"    │────┼──► Called via load() on invocation  │
│  - aliases: [ios,    │    │                                     │
│          android]    │    │  Features:                          │
│  - load: () =>       │    │  - QR code generation               │
│    import(...)      │    │  - Platform switching (keyboard)     │
└──────────────────────┘    │  - Interactive terminal UI          │
         │                  └─────────────────────────────────────┘
         ▼
    Lazy import
```

**Flow:**
1. User runs `claude mobile` (or `claude ios`/`claude android`)
2. `mobile.ts` configuration is loaded
3. Lazy `load()` function triggers, importing `mobile.tsx`
4. `call()` export from `mobile.tsx` instantiates the `MobileQRCode` component
5. User sees QR codes for iOS/Android app stores

## Key Takeaways

- **Dual-platform support**: Single command handles both iOS and Android with app store deep links
- **Three command aliases**: `mobile`, `ios`, `android` all invoke the same functionality
- **Lazy loading**: Actual component loads only when command is executed, keeping CLI startup fast
- **QR code generation**: Uses the `qrcode` package with UTF-8 output optimized for terminal rendering
- **Pre-computed codes**: Both QR codes generated once on mount to ensure instant switching without flicker
- **Keyboard-driven UX**: `tab`/`arrows` switch platforms; `q`/`esc`/`ctrl+c` close the view
- **Visual feedback**: Active platform highlighted with bold + underline styling
