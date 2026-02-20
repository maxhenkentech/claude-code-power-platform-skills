# File Upload & Download in Code Apps

## CSP Constraints (Complete Picture)

Code Apps run in a sandboxed iframe with a strict Content Security Policy enforced by the Power Platform player. All four of these directives are active simultaneously:

| Directive | Value | Consequence |
|---|---|---|
| `connect-src` | `'none'` | All `fetch()` and XHR blocked — including same-origin |
| `frame-src` | `'self'` | Iframes with blob: URLs, external URLs, or cross-origin content blocked |
| `worker-src` | `'none'` | All Web Workers blocked — including `new Worker(blobUrl)` |
| `script-src` | `'self' 'unsafe-inline'` | Only scripts from same origin; blob: script loading blocked |

**The only safe HTTP channel** is the SDK's internal postMessage bridge (`AppHttpClientPlugin.sendHttpAsync`). The parent player frame makes the actual HTTP request outside the iframe's CSP.

**Consequences for common patterns:**
- ❌ `fetch()` / `XMLHttpRequest` — always fails (`connect-src 'none'`)
- ❌ `<iframe src="blobUrl">` — blocked (`frame-src 'self'`, blob: not 'self')
- ❌ `<iframe src="externalUrl">` — blocked (`frame-src 'self'`)
- ❌ `new Worker('blobUrl')` — blocked (`worker-src 'none'`)
- ❌ Binary GET responses through SDK bridge — corrupted (bridge passes binary through TextDecoder internally, replacing invalid UTF-8 bytes with U+FFFD, inflating file size ~1.72x)
- ✅ SDK bridge with JSON bodies — all API calls, file upload, file download
- ✅ Canvas rendering (PDF.js, docx-preview) — no iframe, no worker needed

---

## Dataverse FileType Column Uploads

### The Three-Step Block Upload API

Dataverse FileType columns require:
1. `InitializeFileBlocksUpload` (POST, JSON) → returns `FileContinuationToken`
2. `UploadBlock` (POST, JSON with base64 chunk) × N (max 4 MB per block)
3. `CommitFileBlocksUpload` (POST, JSON) → commits the file

All three use JSON bodies — no binary/multipart — which is why they work through the SDK's standard HTTP channel.

### Required Setup

**1. Vite alias** — Vite v5 enforces `package.json` exports map strictly. The plugin bridge path is not exported, so add a `resolve.alias` to bypass it:

```typescript
// vite.config.ts
import path from 'path'
export default defineConfig({
  resolve: {
    alias: {
      '@microsoft/power-apps/lib/internal/plugins': path.resolve(
        './node_modules/@microsoft/power-apps/lib/internal/plugins/index.js',
      ),
    },
  },
})
```

**2. TypeScript ambient declaration** — `tsconfig` with `moduleResolution: "bundler"` also respects the exports map. Add a type declaration file:

```typescript
// src/types/power-apps-internal.d.ts
declare module '@microsoft/power-apps/lib/internal/plugins' {
  export function executePluginAsync(
    service: string,
    action: string,
    params: unknown[],
  ): Promise<unknown>
}
```

### Complete Upload Implementation

```typescript
import { executePluginAsync } from '@microsoft/power-apps/lib/internal/plugins'

function arrayBufferToBase64(buffer: ArrayBuffer): string {
  const bytes = new Uint8Array(buffer)
  let binary = ''
  const CHUNK = 8192
  for (let i = 0; i < bytes.length; i += CHUNK) {
    binary += String.fromCharCode(...bytes.subarray(i, i + CHUNK))
  }
  return btoa(binary)
}

async function getDataverseEnv() {
  const dsConfigs: any = await executePluginAsync(
    'AppPowerAppsClientPlugin',
    'getAppCdsDataSourceConfigsAsync',
    [],
  )
  let runtimeUrl = ''
  for (const val of Object.values(dsConfigs as object)) {
    const v = val as any
    const url = v?.runtimeUrl ?? v?.runtimeurl ?? v?.RuntimeUrl
    if (typeof url === 'string' && url.length > 0) { runtimeUrl = url; break }
  }
  const instanceUrl = runtimeUrl.match(/^(https?:\/\/[^/]+)/)?.[1] ?? ''

  const token = await executePluginAsync(
    'AppIdentityServicePlugin',
    'getAppDynamicResourceAccessTokenAsync',
    ['default.cds'],
  ) as string

  return { instanceUrl, token }
}

async function callDataverseAction(instanceUrl: string, token: string, actionName: string, payload: object) {
  const url = `${instanceUrl}/api/data/v9.0/${actionName}`
  const blob = new Blob([JSON.stringify(payload)], { type: 'application/json' })

  const baseUrl = url.match(/^(https?:\/\/[^/]+\/api\/data\/v9\.0)/)?.[1] ?? ''
  const rawPath = url.match(/\/api\/data\/v9\.0\/(.+)$/)?.[1] ?? actionName
  const encodedPath = encodeURIComponent(rawPath).replace(/\(/g, '%28').replace(/\)/g, '%29')

  const headers = {
    Accept: 'application/json',
    'x-ms-protocol-semantics': 'Dataverse',
    Authorization: `dynamicauth ${token}`,
    ServiceNamespace: actionName,
    'x-ms-pa-client-custom-headers-options': '{"addCustomHeaders":true}',
    'x-ms-enable-selects': 'true',
    'x-ms-pa-client-telemetry-options': `paclient-telemetry {"operationName":"fileUpload_${actionName}"}`,
    'x-ms-pa-client-telemetry-additional-data': '{"apiId":"Dataverse"}',
    BatchInfo: JSON.stringify({
      baseUrl,
      encodedPath,
      headers: { Accept: 'application/json', Prefer: 'return=representation,odata.include-annotations=*', 'Content-Type': 'application/json' },
      batchId: '',
    }),
  }

  const rawResult: any = await executePluginAsync('AppHttpClientPlugin', 'sendHttpAsync', [
    { url, method: 'POST', requestSource: 'PublishedApp', allowSessionStorage: true, returnDirectResponse: true, headers },
    blob,
    'arraybuffer',
  ])

  const status: number = rawResult?.[0]?.status ?? 0
  const buffer: ArrayBuffer = rawResult?.[1]

  if (!buffer || buffer.byteLength === 0) {
    if (status >= 200 && status < 300) return {}
    throw new Error(`${actionName} HTTP ${status} empty body`)
  }

  const text = new TextDecoder('utf-8').decode(buffer)
  const parsed = JSON.parse(text)
  if (status < 200 || status >= 300) throw new Error(`${actionName} HTTP ${status}: ${text}`)
  return parsed
}

export async function uploadDataverseFile(docId: string, file: File, tableLogicalName: string, fileAttributeName: string) {
  const { instanceUrl, token } = await getDataverseEnv()

  // Step 1: Initialize — FileAttributeName goes HERE only
  const initResp: any = await callDataverseAction(instanceUrl, token, 'InitializeFileBlocksUpload', {
    Target: {
      '@odata.type': `Microsoft.Dynamics.CRM.${tableLogicalName}`,
      [`${tableLogicalName}id`]: docId,
    },
    FileAttributeName: fileAttributeName,
    FileName: file.name,
  })
  const fileContinuationToken: string = initResp?.FileContinuationToken  // NOT FileId
  if (!fileContinuationToken) throw new Error('No FileContinuationToken in InitializeFileBlocksUpload response')

  // Step 2: Upload blocks (≤ 4 MB each, base64-encoded)
  const buffer = await file.arrayBuffer()
  const BLOCK_SIZE = 4 * 1024 * 1024
  const totalBlocks = Math.max(1, Math.ceil(buffer.byteLength / BLOCK_SIZE))
  const blockIds: string[] = []

  for (let i = 0; i < totalBlocks; i++) {
    const chunk = buffer.slice(i * BLOCK_SIZE, Math.min((i + 1) * BLOCK_SIZE, buffer.byteLength))
    const blockId = btoa(String(i).padStart(32, '0'))
    blockIds.push(blockId)
    await callDataverseAction(instanceUrl, token, 'UploadBlock', {
      FileContinuationToken: fileContinuationToken,
      BlockId: blockId,
      BlockData: arrayBufferToBase64(chunk),
    })
  }

  // Step 3: Commit — NO FileAttributeName here; BlockList is required
  await callDataverseAction(instanceUrl, token, 'CommitFileBlocksUpload', {
    FileContinuationToken: fileContinuationToken,
    FileName: file.name,
    MimeType: file.type || 'application/octet-stream',
    BlockList: blockIds,
    // ⚠️ FileAttributeName is NOT a valid parameter here — omit it
  })
}
```

### Upload Gotchas

| Gotcha | Detail |
|---|---|
| `executeAsync` does NOT support file upload | The SDK's `executeAsync` only handles `getEntityMetadata`. Use `executePluginAsync` directly. |
| Response token is `FileContinuationToken` | Not `FileId`, `FileToken`, or `ContinuationToken`. The exact key matters. |
| `FileAttributeName` is invalid in `CommitFileBlocksUpload` | Only valid in `InitializeFileBlocksUpload`. Including it causes `0x80048d19`. |
| `BlockList` is required in `CommitFileBlocksUpload` | Must pass the array of all `BlockId` strings uploaded in step 2. |
| `runtimeUrl` key is lowercase in dsConfigs | The SDK lowercases config keys. Check `runtimeUrl`, `runtimeurl`, AND `RuntimeUrl`. |
| SDK URL uses `/api/data/v9.0/` | The player translates to the live API version. Don't use the version from `runtimeUrl`. |
| Vite exports map blocks the import | Must add `resolve.alias` — Vite v5 strictly enforces the package exports map. |

---

## Dataverse FileType Column Downloads

### Why You Cannot Use the `$value` GET Endpoint

The natural approach — `GET hek_lldocuments(id)/hek_filecontent/$value` — is fatally broken in Code Apps:

1. **`connect-src 'none'`** blocks direct `fetch()` entirely.
2. Even routing through `AppHttpClientPlugin.sendHttpAsync`, **the SDK bridge internally passes binary GET responses through `TextDecoder`**, replacing every invalid UTF-8 byte sequence with the Unicode replacement character U+FFFD. Binary files (PDF, DOCX, images) are corrupted — typically inflated ~1.72× in size.

### The Two-Step Block Download API

Use `InitializeFileBlocksDownload` + `DownloadBlock`. These return **base64 JSON** — text-safe end-to-end, no corruption:

1. `InitializeFileBlocksDownload` (POST, JSON) → returns `FileContinuationToken`, `FileSizeInBytes`, `FileName`
2. `DownloadBlock` (POST, JSON, iterate with `Offset`/`BlockLength`) → returns `{ Data: "<base64>" }`

### Complete Download Implementation

Uses the same `getDataverseEnv()` and `callDataverseAction()` helpers from the upload section above.

```typescript
export async function downloadDataverseFile(
  docId: string,
  tableLogicalName: string,
  fileAttributeName: string,
): Promise<ArrayBuffer> {
  const { instanceUrl, token } = await getDataverseEnv()

  // Step 1: Initialize
  const initResp: any = await callDataverseAction(instanceUrl, token, 'InitializeFileBlocksDownload', {
    Target: {
      '@odata.type': `Microsoft.Dynamics.CRM.${tableLogicalName}`,
      [`${tableLogicalName}id`]: docId,
    },
    FileAttributeName: fileAttributeName,
  })

  const fileContinuationToken: string = initResp?.FileContinuationToken
  // ⚠️ The field is FileSizeInBytes, not FileSizeCode
  const fileSize = parseInt(String(
    initResp?.FileSizeInBytes ?? initResp?.FileSizeCode ?? initResp?.filesizecode ?? 0
  ), 10)
  if (!fileContinuationToken || fileSize === 0) {
    throw new Error(`InitializeFileBlocksDownload failed: ${JSON.stringify(initResp)}`)
  }

  // Step 2: Download blocks (base64 JSON responses — no binary corruption)
  const BLOCK_SIZE = 4 * 1024 * 1024
  const chunks: Uint8Array[] = []

  for (let offset = 0; offset < fileSize; offset += BLOCK_SIZE) {
    const blockLength = Math.min(BLOCK_SIZE, fileSize - offset)
    const blockResp: any = await callDataverseAction(instanceUrl, token, 'DownloadBlock', {
      FileContinuationToken: fileContinuationToken,
      Offset: offset,
      BlockLength: blockLength,
    })
    const base64Data: string = blockResp?.Data
    const binaryString = atob(base64Data)
    const bytes = new Uint8Array(binaryString.length)
    for (let i = 0; i < binaryString.length; i++) bytes[i] = binaryString.charCodeAt(i)
    chunks.push(bytes)
  }

  // Step 3: Combine chunks
  const totalLength = chunks.reduce((sum, c) => sum + c.length, 0)
  const result = new Uint8Array(totalLength)
  let pos = 0
  for (const chunk of chunks) { result.set(chunk, pos); pos += chunk.length }
  return result.buffer
}
```

### Download Gotchas

| Gotcha | Detail |
|---|---|
| Binary GET via SDK bridge corrupts files | Bridge runs GET responses through `TextDecoder`; use block download (base64 JSON) instead |
| `FileSizeInBytes` not `FileSizeCode` | The response field is `FileSizeInBytes`. Check `FileSizeCode`/`filesizecode` as fallbacks |
| `FileAttributeName` goes in `InitializeFileBlocksDownload` only | Same rule as upload — not valid in `DownloadBlock` |
| `window.location.origin` is NOT the Dataverse URL | The Code App is hosted on `powerplatformusercontent.com`. Always get `instanceUrl` from `getAppCdsDataSourceConfigsAsync` |
