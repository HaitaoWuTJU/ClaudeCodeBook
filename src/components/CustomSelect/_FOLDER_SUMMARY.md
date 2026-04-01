# Summary of `components/CustomSelect/`

## Purpose of `CustomSelect/`

Provides a terminal-friendly (Ink.js) custom select/dropdown component system for CLI applications. Supports single/multi selection, text input within options, image paste attachments, keyboard-driven navigation, and external editor integration.

---

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Public API barrel — re-exports all types and components |
| `option-map.ts` | Data structure — doubly-linked `Map` for O(1) bidirectional option traversal |
| `select.tsx` | Root component — renders the full select UI with state coordination |
| `select-option.tsx` | Leaf component — wraps `ListItem` for individual option rows |
| `select-input-option.tsx` | Leaf component — text input with image paste, external editor, cursor management |
| `use-select-state.ts` | State composition — wires selection state with navigation state |
| `use-select-navigation.ts` | Navigation logic — focus, viewport scrolling, circular wrapping, page jumping |
| `use-select-input.ts` | Input handling — keyboard events for navigation, Tab toggle, number quick-select, image selection mode |

---

## How Files Relate to Each Other

```
index.ts (public API)
│
└── select.tsx (root component)
    │
    ├── useSelectState.ts (state orchestration)
    │   │
    │   ├── useSelectNavigation.ts (focus + viewport state)
    │   │   └── option-map.ts (linked list for next/prev)
    │   │
    │   └── local value state (selected value)
    │
    ├── useSelectInput.ts (keyboard handling)
    │
    ├── select-option.tsx (renders each visible option)
    │
    └── select-input-option.tsx (renders input-mode option)
        └── TextInput, ClickableImageRef, Byline
```

### Data Flow Summary

```
options[] (flat array)
    │
    ▼
OptionMap (built once on options change)
    │ → first/last pointers for quick nav
    │ → each item has previous/next for O(1) traversal
    │
    ▼
useSelectNavigation
    │ → manages focusedValue, visibleFromIndex, visibleToIndex
    │ → exposes focusNextOption, focusPreviousOption, focusNextPage...
    │
    ▼
useSelectState
    │ → composes navigation state + local value state
    │ → exposes selectFocusedOption()
    │
    ▼
select.tsx
    │ → slices visibleOptions from viewport window
    │ → renders select-option.tsx for each visible item
    │ → renders select-input-option.tsx if focused option is input-type
    │
    ▼
Keyboard events → useSelectInput
    │ → dispatches focus/select actions to useSelectState
    │ → manages Tab toggle, number keys, image selection mode
```

---

## Key Takeaways

### Architecture Pattern
The component uses a **pure state + composition** pattern:
- State lives in hooks (`useSelectNavigation`, `useSelectState`)
- Components are pure renderers — they receive state and callbacks as props
- No class components or class state

### Performance Optimizations
- **Virtual windowing**: Only `visibleOptionCount` options are rendered at a time (default: 5), regardless of total options
- **Memoized callbacks**: All callbacks use `useCallback` to prevent child re-renders
- **React Compiler**: Uses `_c` pattern for optimized re-rendering via `react/compiler-runtime`
- **Deep equality check**: Options array compared via `util.isDeepStrictEqual` before expensive state resets

### Keyboard Navigation Features
- Arrow keys (j/k) for single-item navigation with **circular wrapping**
- PageUp/PageDown for **page-based jumping**
- Number keys (1–9) for **quick direct selection**
- Enter to select, Escape to cancel
- Tab to **toggle between select and input mode**
- Ctrl+G to open **external editor** from input fields
- Image paste via clipboard integration
- Image attachment navigation with left/right arrows

### Type Safety
- Generic `<T>` used throughout for type-safe option values
- `OptionWithDescription<T>` provides the standard option shape with optional description
- All props marked `readonly` to enforce immutability
