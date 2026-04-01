# Summary of `components/ui/`

## Purpose of `ui/`

The `ui/` directory contains reusable React components for rendering hierarchical data structures in terminal UI (using `ink.js`). These components handle the visual presentation of nested content—whether as numbered lists, list items, or tree-based navigation—abstracting away the complexity of state management, context propagation, and keyboard interaction.

---

## Contents Overview

| File | Component | Primary Role |
|------|-----------|--------------|
| `OrderedList.tsx` | `OrderedList` | Renders a numbered list with automatic hierarchical numbering (e.g., `1.`, `2.`, `1.1.`, `1.2.`) |
| `OrderedListItem.tsx` | `OrderedListItem` | Individual list item displaying a dimmed marker followed by content |
| `TreeSelect.tsx` | `TreeSelect<T>` | Generic tree-view selector with expand/collapse, keyboard navigation, and selection |

---

## How Files Relate to Each Other

```
OrderedList.tsx
    └── OrderedListItem.tsx
            └── OrderedListItemContext (shared state)
            └── OrderedListContext (nested numbering context)

TreeSelect.tsx
    └── Select (from CustomSelect/select.js)
    └── Box, Text (from ink.js)
    └── KeyboardEvent (from ink/events)
```

### Shared Patterns

1. **Context-based State Propagation**:
   - `OrderedList` → `OrderedListContext` → `OrderedListItem` → `OrderedListItemContext`
   - Both use React's `createContext`/`useContext` to pass hierarchical state down the component tree

2. **ink.js Integration**: All components render inside `<Box>` components from the local `ink.js` library for terminal UI layout

3. **React Compiler Compatibility**: `OrderedListItem.tsx` and `TreeSelect.tsx` both include `react/compiler-runtime` integration for optimized memoization

4. **Hierarchical Data Handling**:
   - `OrderedList` and `OrderedListItem` work as a compound component (parent-child pattern)
   - `TreeSelect` manages its own internal tree flattening logic

---

## Key Takeaways

| Pattern | Description |
|---------|-------------|
| **Compound Components** | `OrderedList.Item` syntax allows semantically grouping parent and child elements |
| **Context Isolation** | Separate contexts (`OrderedListContext` vs `OrderedListItemContext`) keep concerns decoupled |
| **Generic Programming** | `TreeSelect<T>` uses TypeScript generics to preserve user data types through the component |
| **Memoization Strategy** | Components use React Compiler runtime (`_c`) or manual cache arrays (`$[]`) to optimize re-renders |
| **Hierarchical Numbering** | Automatic numbering accumulation (`1.` → `1.1.` → `1.1.1.`) via context inheritance |
| **Keyboard Navigation** | `TreeSelect` handles `←`/`→` keys for expand/collapse with parent-traversal fallback |
| **Controlled vs Uncontrolled** | `TreeSelect` supports both internal state management and fully controlled mode via props |
