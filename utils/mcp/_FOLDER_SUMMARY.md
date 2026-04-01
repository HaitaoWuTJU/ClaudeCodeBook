# Summary of `utils/mcp/`

## Purpose of `mcp/`

The `utils/mcp/` directory contains utilities for handling **Model Context Protocol (MCP) elicitation workflows** — specifically, parsing human-readable date/time inputs and validating user-submitted values against schema definitions.

## Contents Overview

| File | Role |
|------|------|
| `dateTimeParser.ts` | Converts natural language date/time strings (e.g., "tomorrow at 3pm") into ISO 8601 format using AI (Haiku) |
| `elicitationValidation.ts` | Validates string inputs against JSON Schema definitions by converting them to Zod schemas; provides NL date/time parsing as a fallback |

## How Files Relate to Each Other

```
User Input (e.g., "tomorrow at 3pm")
        │
        ▼
┌─────────────────────────────────┐
│ elicitationValidation.ts       │
│  └─► Checks schema type         │
│      └─► If date-time format:   │
│          ▼                     │
│   ┌─────────────────────────┐   │
│   │ dateTimeParser.ts       │   │
│   │  └─► queryHaiku AI       │   │
│   │  └─► ISO 8601 output     │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
```

- **`dateTimeParser.ts`** is consumed by **`elicitationValidation.ts`** as a fallback mechanism for date/date-time schemas when standard validation fails
- Both files return typed `Result` objects (`DateTimeParseResult` / `ValidationResult`) for consistent error handling upstream
- `dateTimeParser.ts` provides `looksLikeISO8601()` to enable smart fallback triggering

## Key Takeaways

1. **AI-Assisted Parsing**: NL date/time parsing leverages Haiku via carefully crafted system prompts, providing rich context (timezone, current day/weekday) to maximize accuracy

2. **Schema-to-Zod Bridge**: `elicitationValidation.ts` translates JSON Schema primitives into Zod validation, supporting both legacy (`enum: [...]`) and modern (`oneOf: [...]`) enum formats

3. **Graceful Fallbacks**: When direct Zod validation fails for date-time fields and the input doesn't already look like ISO 8601, the system attempts natural language parsing before returning an error

4. **Format-Aware Validation**: Built-in support for `email`, `uri`, `date`, and `date-time` string formats with descriptive, pluralized error messages

5. **Multi-Select Support**: Handles both single-value enums and multi-select (array) enums with value/label extraction utilities
