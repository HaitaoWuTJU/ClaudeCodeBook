# Summary of `tools/BriefTool/`

## BriefTool Directory Summary

### Purpose

`BriefTool/` implements the AI assistant's primary visible output channel for sending messages to users with optional file attachments. It provides a typed, validated tool interface that transforms file paths into metadata, optionally uploads attachments for web preview, and renders results in multiple CLI view modes.

### Contents Overview

| File | Role |
|------|------|
| `BriefTool.ts` | Tool definition: input/output schemas, `isBriefEnabled()` gate, `call()` handler, UI renderers, progress handler |
| `attachments.ts` | Shared attachment resolution: path validation → file stats → MIME detection → conditional upload |
| `upload.ts` | BRIDGE_MODE-only: multipart upload to `/api/oauth/file_upload` with 30 MB limit and best-effort error handling |
| `prompt.ts` | Pure exports: tool name (`SendUserMessage`), descriptions, and communication guidelines |
| `UI.tsx` | React/Ink renderer: three-mode output (transcript/brief-only/default) with attachment list |

### How Files Relate to Each Other

```
prompt.ts
  └─ exports: tool name, description, guidelines
       │
BriefTool.ts
  ├─ imports DESCRIPTION from prompt.ts
  ├─ imports resolveAttachments from attachments.ts
  └─ imports renderToolResultMessage from UI.tsx
       │
attachments.ts
  └─ [BRIDGE_MODE] dynamically imports uploadBriefAttachment from upload.ts
       │
upload.ts
  └─ reads files, POSTs to private API, returns file_uuid
```

**Call chain for a message with attachments:**

1. `BriefTool.call(message, attachments)` validates input via `attachments.ts`
2. `resolveAttachments()` runs `stat()` serially, detects images, uploads in parallel
3. Output flows to `mapToolResultToToolResultBlockParam()` → `renderToolResultMessage()`
4. UI renders in one of three modes based on context flags

### Key Takeaways

- **Feature-gated upload**: The entire `upload.ts` subtree (axios, crypto, zod, auth utils, MIME map) is eliminated from non‑BRIDGE_MODE builds via `if (feature('BRIDGE_MODE'))` positive-guard pattern
- **Activation requires two gates**: `userMsgOptIn` (user choice) **AND** GrowthBook entitlement (`KAIROS`/`KAIROS_BRIEF`) with 5‑minute refresh — either can disable mid-session
- **Optional attachments**: Intentional — session replay may carry message payloads without attachments; required field would break replay rendering
- **Best-effort uploads**: Any failure (no token, network, size limit, parse error) silently returns `undefined` for `file_uuid`; the attachment still renders locally with path/size/isImage
- **Three UI modes**: Transcript mode shows a `⏺` gutter marker; brief-only mode shows "Claude" + timestamp to match user message styling; default mode omits metadata for clean output
- **Communication contract**: `prompt.ts` codifies the rule — only content inside `BriefTool` is guaranteed visible to users; everything else is "detail view" content most users won't open
