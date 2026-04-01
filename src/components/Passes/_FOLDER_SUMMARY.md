# Summary of `components/Passes/`

## Purpose of `Passes/`

The `Passes/` directory contains the guest passes dialog component for Claude Code's CLI application. It enables users to view their referral guest pass status (up to 3 passes), see which have been redeemed, and copy referral links to share with others. The components render terminal-based UI using the `ink` library (React for terminals).

## Contents Overview

| File | Purpose |
|------|---------|
| `Passes.tsx` | Main component: fetches eligibility/redemption data, renders ASCII art tickets, manages keyboard input, copies referral links to clipboard |
| `Pane.tsx` | Reusable pane wrapper component with title and optional spinner |
| `ShareLink.tsx` | Renders the referral link display section with copy functionality and terms text |
| `package.json` | Package configuration with `private: true`, no external dependencies (all imports from parent) |

## How Files Relate to Each Other

```
Passes.tsx (main)
    ├── Pane.tsx (title container + optional loading spinner)
    └── ShareLink.tsx (referral link display with copy button + terms)
```

- **`Passes.tsx`** is the orchestrating component that:
  - Fetches referral data from API services
  - Manages loading/error states
  - Renders a `Pane` containing the ASCII art tickets
  - Includes a `ShareLink` section for copying the referral link
  - Handles keyboard shortcuts (`Esc` to cancel, `Enter` to copy)

- **`Pane.tsx`** provides a simple container with a header bar for section titles, optionally displaying a spinner during loading

- **`ShareLink.tsx`** renders the referral link with a simulated button, handles click events for copying to clipboard, and displays terms/conditions text with dynamic URLs based on referrer reward availability

## Key Takeaways

1. **ASCII Art Rendering**: Passes are displayed as box-drawing character tickets with `TEARDROP_ASTERISK` symbols; available passes appear bright, redeemed passes are dimmed with slash variants

2. **State Management**: The component manages complex async state including loading status, pass availability array, referral link, and referrer reward information

3. **Keyboard Handling**: Uses `useInput` and `useExitOnCtrlCDWithKeybindings` hooks for terminal-appropriate interaction patterns (`Esc` cancels, `Enter` confirms)

4. **Clipboard Integration**: Leverages OSC escape sequences via `setClipboard()` to write to the system clipboard, with fallback raw response output

5. **Analytics**: Logs `tengu_guest_passes_link_copied` events for tracking copy behavior

6. **Error Resilience**: Gracefully handles API errors by showing passes as unavailable rather than failing completely

7. **No Runtime Dependencies**: All subcomponents have no external npm dependencies—they import everything from the parent monorepo packages
