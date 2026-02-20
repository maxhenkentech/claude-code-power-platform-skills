# Claude Code Power Platform Skills

Claude Code skills for Microsoft Power Platform development. These skills teach Claude hard-won patterns and gotchas for building on Power Platform — the stuff that isn't in the official docs.

## Skills

| Skill | What it covers |
|---|---|
| [power-apps-code-apps](./power-apps-code-apps/) | Power Apps Code Apps — Dataverse CRUD, file upload/download, PDF/document rendering, CSP constraints, PAC CLI |

---

## power-apps-code-apps

A Claude Code skill for building **Power Apps Code Apps** — standalone React/TypeScript apps hosted inside Power Platform, connected to Dataverse and other Power Platform data sources via the Power Apps SDK.

This skill focuses on the **non-obvious stuff**: the CSP sandbox constraints, the Dataverse SDK quirks, the file upload/download block APIs, and the PAC CLI version gotchas that you only find out about by hitting them in production.

### What's covered

**Core SDK & PAC CLI**
- Project structure, PAC CLI commands (auth, data sources, deployment)
- `getContext()`, Dataverse CRUD patterns with generated service classes
- `IOperationResult` error handling (does not throw — always check `result.error`)

**Dataverse gotchas** (`references/dataverse-gotchas.md`)
- PAC CLI <2.x bug: `dataSourceName` uses friendly name instead of `entitySetName` → runtime "Data source not found" error + fix
- Lookup columns: `_tablename_value` (read-only) vs `@odata.bind` (write) — what breaks if you mix them up
- `_*_value` virtual properties are silently dropped from `$filter` queries → client-side filtering workaround
- Lookup GUIDs not returned unless explicitly listed in `select`

**File upload & download** (`references/file-operations.md`)
- Why `fetch()` and XHR are completely blocked (`connect-src 'none'`)
- The only working HTTP channel: `AppHttpClientPlugin.sendHttpAsync` via postMessage bridge
- Full 3-step block upload implementation (`InitializeFileBlocksUpload` → `UploadBlock` → `CommitFileBlocksUpload`)
- Full 2-step block download implementation (binary GET corrupts files — base64 block download is the fix)
- Vite v5 alias required to import `executePluginAsync` from internal plugin path
- All critical gotchas (`FileContinuationToken` not `FileId`, `FileAttributeName` invalid in `CommitFileBlocksUpload`, etc.)

**Document rendering** (`references/document-rendering.md`)
- PDF.js in main-thread mode (no Worker — `worker-src 'none'` blocks all workers)
- Why `workerSrc` must NOT be set, and how the side-effect import bypasses it
- DOCX rendering with `mammoth` (inline `dangerouslySetInnerHTML`, not iframe)
- XLSX/XLS rendering with SheetJS with multi-sheet support
- Format support matrix (PDF ✅, DOCX ✅, XLSX ✅, DOC ❌, PPTX ❌) + why the unsupported formats have no viable JS renderer
- Server-side conversion via Power Automate for DOC/PPTX

### Complements

This skill covers the deep gotchas. For getting-started guidance (scaffolding, Vite config, initial PAC CLI setup), see [DanielKerridge/claude-code-power-platform-skills](https://github.com/DanielKerridge/claude-code-power-platform-skills).

---

## Installation

### Personal (all your projects)

**macOS & Linux**
```bash
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git /tmp/pp-skills && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/pp-skills/power-apps-code-apps ~/.claude/skills/ && \
rm -rf /tmp/pp-skills
```

**Windows (PowerShell)**
```powershell
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git "$env:TEMP\pp-skills"
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse "$env:TEMP\pp-skills\power-apps-code-apps" "$env:USERPROFILE\.claude\skills\"
Remove-Item -Recurse -Force "$env:TEMP\pp-skills"
```

**Windows (Git Bash / WSL)** — use the macOS & Linux commands above.

Claude will automatically load the skill when you're working on Power Apps Code Apps.

> **Note:** The `mkdir -p` / `New-Item -Force` step is required. Without it, if the `skills/` directory doesn't exist yet, `cp -r` / `Copy-Item` will rename the source directory to `~/.claude/skills` instead of copying it inside — and the skill won't be found.

### Project-level (shared with everyone on the repo)

**macOS & Linux**
```bash
mkdir -p .claude/skills && \
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git /tmp/pp-skills && \
cp -r /tmp/pp-skills/power-apps-code-apps .claude/skills/ && \
rm -rf /tmp/pp-skills
```

**Windows (PowerShell)**
```powershell
New-Item -ItemType Directory -Force ".claude\skills" | Out-Null
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git "$env:TEMP\pp-skills"
Copy-Item -Recurse "$env:TEMP\pp-skills\power-apps-code-apps" ".claude\skills\"
Remove-Item -Recurse -Force "$env:TEMP\pp-skills"
```

**Windows (Git Bash / WSL)** — use the macOS & Linux commands above.

Commit `.claude/skills/power-apps-code-apps/` to your repo — everyone who clones it gets the skill automatically.

### Keeping it up to date

**macOS & Linux**
```bash
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git /tmp/pp-skills && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/pp-skills/power-apps-code-apps ~/.claude/skills/ && \
rm -rf /tmp/pp-skills
```

**Windows (PowerShell)**
```powershell
git clone https://github.com/maxhenkentech/claude-code-power-platform-skills.git "$env:TEMP\pp-skills"
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse "$env:TEMP\pp-skills\power-apps-code-apps" "$env:USERPROFILE\.claude\skills\"
Remove-Item -Recurse -Force "$env:TEMP\pp-skills"
```

---

## How it works

Claude Code skills are loaded automatically when Claude detects relevance from the skill's description. Once installed, Claude will load this skill when you ask about:

- Building or initializing a Power Apps Code App
- Adding data sources or running `pac code` commands
- Uploading or downloading files to/from Dataverse
- Rendering PDFs, DOCX, or XLSX files inside a Code App
- Debugging Dataverse errors, CSP issues, or PAC CLI problems

You can also invoke it directly: `/codeapps`

The skill uses **progressive disclosure** — the core `SKILL.md` (~900 words) loads first, and Claude pulls in the detailed reference files (`dataverse-gotchas.md`, `file-operations.md`, `document-rendering.md`) only when needed for the specific topic.

---

## Contributing

PRs welcome — especially additional gotchas from real projects. The reference files are intentionally structured so new sections can be added without touching `SKILL.md`.
