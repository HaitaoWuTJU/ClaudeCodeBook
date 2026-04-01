# Summary of `utils/dxt/`

## Purpose of `dxt/`

The `dxt/` directory provides low-level utilities for processing **DXT (Desktop Extension) packages**. DXT files are ZIP archives containing extension manifests and resources. This directory handles the full extraction and validation pipeline:

1. **ZIP extraction** with security protections
2. **Manifest parsing** from extracted or raw data
3. **Manifest validation** against the schema
4. **Extension ID generation** for uniquely identifying extensions

## Contents Overview

| File | Responsibility |
|------|----------------|
| `helpers.ts` | Manifest parsing, validation (JSON/text/binary), and extension ID generation |
| `zip.ts` | Secure ZIP extraction with path traversal prevention, zip bomb detection, and Unix permission preservation |

## How Files Relate to Each Other

```
ZIP Archive (DXT file)
       │
       ▼
┌──────────────────┐
│   zip.ts         │ ──► unzipFile() extracts contents
│ (extraction)     │     + validateZipFile() checks each entry
└────────┬─────────┘
         │ Record<string, Uint8Array>
         ▼
┌──────────────────┐
│  helpers.ts      │ ──► parseAndValidateManifestFromBytes()
│ (manifest)       │     + validateManifest() against McpbManifestSchema
└────────┬─────────┘     + generateExtensionId() creates unique ID
         │
         ▼
   McpbManifest + extensionId string
```

The two files form a processing pipeline: `zip.ts` handles the container format, `helpers.ts` processes the contained manifest. Both use lazy imports to defer heavy dependencies (`fflate`, `zod`) until actually needed.

## Key Takeaways

- **Memory optimization**: Both modules use lazy imports to avoid loading ~900KB of closures/tables at startup (fflate + Zod combined)
- **Security-first design**: `zip.ts` implements defense-in-depth with path traversal checks, per-file and total size limits, and zip bomb ratio detection (50:1 threshold)
- **Type safety**: Manifests are validated against `McpbManifestSchema` (Zod), producing typed `McpbManifest` objects
- **Error handling**: Validation errors are flattened into readable single-line messages with field-level details
- **Cross-platform compatibility**: Uses `getFsImplementation()` abstraction rather than Node.js `fs` directly
- **Bun compatibility**: `zip.ts` uses synchronous `unzipSync` to avoid fflate worker crashes in Bun environments
