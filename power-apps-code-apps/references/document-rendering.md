# Document Rendering in Code Apps

## Format Support Matrix

| Format | Renderer | Quality | Notes |
|---|---|---|---|
| PDF | `pdfjs-dist` (canvas) | ✅ Full | See PDF Rendering section below |
| DOCX | `mammoth` | ✅ Good | Converts to HTML, injects into scoped div |
| XLSX / XLS | `xlsx` (SheetJS) | ✅ Good | Converts to HTML table, supports multiple sheets |
| TXT / CSV / JSON / RTF | TextDecoder | ✅ Full | Inline `<pre>` display |
| DOC | ❌ None | — | Old OLE binary format; no viable JS parser |
| PPTX | ❌ None | — | No maintained JS renderer; text extraction only |
| PPT | ❌ None | — | Old OLE binary format; no viable JS parser |

### Why DOC / PPTX / PPT Have No Browser Renderer

- **DOC and PPT** are pre-2007 OLE Compound File Binary formats. No maintained JavaScript parsers render them visually — only system tools like LibreOffice/antiword work.
- **PPTX** is a ZIP of XML (like DOCX), but there is no `mammoth`-equivalent for presentations. Text can be extracted with JSZip + XML parsing, but slide layout/images cannot be reproduced.
- **WASM-based LibreOffice** exists but requires `'wasm-unsafe-eval'` in `script-src`, which the Power Platform CSP does not include. WASM is therefore blocked.

---

## PDF Rendering

### Why Iframes and Workers Don't Work

- **`frame-src 'self'`** blocks all iframes with blob: URLs or external content — including PDF viewer iframes
- **`worker-src 'none'`** blocks all Web Workers — including PDF.js's default separate worker
- **`script-src 'self' 'unsafe-inline'`** blocks blob: script loading, so fake-worker blob URLs also fail

### The Solution: PDF.js Main-Thread Mode via Side-Effect Import

PDF.js checks for `globalThis.pdfjsWorker?.WorkerMessageHandler` before attempting any Worker creation. If the global is set, it runs the entire PDF pipeline on the main thread with no Worker at all.

Importing the worker module as a side effect bundles it into the main JS file (served from `'self'`, passes `script-src`), and sets that global automatically:

```typescript
import * as pdfjsLib from 'pdfjs-dist'
// Importing the worker as a side-effect module bundles it into the main JS bundle.
// PDF.js detects globalThis.pdfjsWorker.WorkerMessageHandler and skips all Worker
// creation — runs entirely on the main thread, passing both worker-src and script-src.
import 'pdfjs-dist/build/pdf.worker.min.mjs'
```

**Do NOT set `pdfjsLib.GlobalWorkerOptions.workerSrc`** — that triggers Worker creation, which is blocked.

### Canvas-Based PDF Rendering Component (React)

```typescript
import * as pdfjsLib from 'pdfjs-dist'
import 'pdfjs-dist/build/pdf.worker.min.mjs'

const PdfPreview: React.FC<{ buffer: ArrayBuffer }> = ({ buffer }) => {
  const containerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!containerRef.current) return
    let cancelled = false
    const container = containerRef.current
    container.innerHTML = ''

    ;(async () => {
      try {
        const data = buffer.slice(0) // defensive copy
        const pdf = await pdfjsLib.getDocument({ data }).promise
        if (cancelled) return

        for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
          if (cancelled) return
          const page = await pdf.getPage(pageNum)
          const viewport = page.getViewport({ scale: 1.5 })
          const canvas = document.createElement('canvas')
          canvas.width = viewport.width
          canvas.height = viewport.height
          canvas.style.cssText = 'display:block;margin-bottom:8px;max-width:100%;'
          container.appendChild(canvas)
          // ⚠️ PDF.js v5 requires BOTH canvas AND canvasContext
          await page.render({ canvas, canvasContext: canvas.getContext('2d')!, viewport }).promise
        }
      } catch (err) {
        console.error('[PdfPreview]', err)
      }
    })()

    return () => { cancelled = true }
  }, [buffer])

  return <div ref={containerRef} />
}
```

### PDF.js Gotchas

| Gotcha | Detail |
|---|---|
| Do NOT set `workerSrc` | Setting it causes PDF.js to create a Worker, which `worker-src 'none'` blocks |
| PDF.js v5 needs both `canvas` and `canvasContext` | `RenderParameters` requires the HTMLCanvasElement (`canvas`) AND its context (`canvasContext`) |
| Import order matters | Import `pdfjs-dist` before importing the worker side-effect |
| Power Platform storage proxy wrong MIME type | `.mjs` files may be served as `application/octet-stream`; the side-effect import via bundler avoids this |

---

## DOCX Rendering with mammoth

`mammoth` converts DOCX to HTML entirely in the browser. Inject into a div (NOT an iframe — use `dangerouslySetInnerHTML`) with scoped CSS for document-like appearance.

```typescript
import mammoth from 'mammoth'

const DocxPreview: React.FC<{ buffer: ArrayBuffer }> = ({ buffer }) => {
  const [html, setHtml] = useState<string | null>(null)

  useEffect(() => {
    mammoth.convertToHtml({ arrayBuffer: buffer }).then(r => setHtml(r.value))
  }, [buffer])

  if (html === null) return <div>Rendering…</div>
  return <div className="docx-body" dangerouslySetInnerHTML={{ __html: html }} />
}
```

**Note on `srcdoc` iframes:** `frame-src 'self'` blocks iframes with `blob:` URLs, but `srcdoc` iframes ARE allowed — their content is considered same-origin. However, `dangerouslySetInnerHTML` is simpler and equally correct; use scoped CSS classes to prevent style bleed.

**mammoth only supports DOCX, not DOC.** For DOC files, offer download only.

---

## XLSX / XLS Rendering with SheetJS

```typescript
import * as XLSX from 'xlsx'

const XlsxPreview: React.FC<{ buffer: ArrayBuffer }> = ({ buffer }) => {
  const [html, setHtml] = useState<string | null>(null)
  const [sheets, setSheets] = useState<string[]>([])
  const [active, setActive] = useState('')
  const wbRef = useRef<XLSX.WorkBook | null>(null)

  useEffect(() => {
    const wb = XLSX.read(buffer, { type: 'array' })
    wbRef.current = wb
    setSheets(wb.SheetNames)
    const first = wb.SheetNames[0] ?? ''
    setActive(first)
    if (first) setHtml(XLSX.utils.sheet_to_html(wb.Sheets[first]))
  }, [buffer])

  const switchSheet = (name: string) => {
    setActive(name)
    setHtml(XLSX.utils.sheet_to_html(wbRef.current!.Sheets[name]))
  }

  if (html === null) return <div>Rendering…</div>
  return (
    <>
      {sheets.length > 1 && sheets.map(s => (
        <button key={s} onClick={() => switchSheet(s)}>{s}</button>
      ))}
      <div dangerouslySetInnerHTML={{ __html: html }} />
    </>
  )
}
```

---

## On-the-Fly Conversion for Unsupported Formats (DOC / PPTX / PPT)

The only viable approach is **server-side conversion to PDF**, then rendering with PDF.js:

1. **Power Automate instant flow:**
   - Word Online (Business) connector → "Convert to PDF" (for DOC/DOCX)
   - OneDrive for Business connector → "Convert file" (for PPTX/PPT)
   - Accepts a Dataverse record ID, returns base64 PDF

2. **Expose via Dataverse Custom Action** so the Code App can call it through the SDK bridge

3. **Render the returned PDF** with the existing PDF.js canvas renderer

This requires Power Platform infrastructure (a flow + custom action) but keeps everything inside the platform with no external dependencies.
