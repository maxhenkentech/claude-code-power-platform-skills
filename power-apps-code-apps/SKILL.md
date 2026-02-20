---
name: codeapps
description: This skill should be used when the user asks to "build a Power Apps Code App", "add a data source to a Code App", "deploy with PAC CLI", "upload files to Dataverse", "download files from Dataverse", "render PDFs in Power Apps", "render documents in Code Apps", "configure CSP for Code Apps", "set up Dataverse CRUD", "fix a PAC CLI error", "initialize a code app project", or mentions pac code, Power Apps Code Apps, or Power Apps SDK. Provides expert guidance for Power Apps Code Apps development using React, TypeScript, and Power Platform connectors.
disable-model-invocation: true
user-invocable: true
argument-hint: "[task description]"
---

# Power Apps Code Apps

Power Apps Code Apps are custom web applications hosted inside Power Platform, built with React/TypeScript and the Power Apps SDK. They connect to Power Platform data sources via generated services and communicate with Dataverse through the SDK's internal postMessage bridge — **not** direct HTTP — due to a strict CSP (`connect-src 'none'`) that blocks all `fetch()` and XHR.

Always read the relevant reference files when working on a topic — they contain complete implementations, critical gotchas, and battle-tested patterns.

---

## Project Structure

```
my-app/
├── src/
│   ├── App.tsx
│   └── main.tsx
├── generated/
│   └── services/
│       ├── *Service.ts
│       └── *Model.ts
├── power.config.json
├── package.json
└── vite.config.ts
```

---

## Common PAC CLI Commands

**Authentication:**
```bash
pac auth create                          # Authenticate with Power Platform
pac auth list                            # List authentication profiles
pac env select --environment <id>        # Select environment
pac env who                              # Show current environment
```

**Data Sources:**
```bash
pac code add-data-source -a dataverse -t <table-logical-name>          # Add Dataverse table
pac code add-data-source -a <apiName> -c <connectionId>                # Add nontabular source
pac code add-data-source -a <apiName> -c <connectionId> -t <tableId> -d <datasetName>  # Tabular
pac code delete-data-source -a <apiName> -ds <dataSourceName>          # Remove data source
pac connection list                                                     # List connections
```

**Deployment:**
```bash
pac code init --displayname "App Name"   # Initialize code app
pac code push                            # Deploy to Power Platform
pac code push --solutionName "MySolution" # Deploy to specific solution
npm run dev                              # Local development
```

---

## SDK Patterns

**Get user/app context:**
```typescript
import { getContext } from '@microsoft/power-apps/app'

const ctx = await getContext()
console.log(ctx.user.fullName)
console.log(ctx.app.environmentId)
```

**Dataverse CRUD:**
```typescript
import { AccountsService } from './generated/services/AccountsService'

// Create — always check result.error (does NOT throw on failure)
const result = await AccountsService.create({ name: 'New Account' })
if (result?.error) throw new Error(result.error?.message ?? JSON.stringify(result.error))

// Read with query options
const accounts = await AccountsService.getAll({
  select: ['name', 'accountnumber', '_primarycontact_value'],  // include lookup GUIDs explicitly
  filter: "address1_country eq 'USA'",                         // avoid _*_value virtual props in filter
  orderBy: ['name asc'],
  top: 50,
})

// Update — strip _*_value read-only properties before writing
const { _ownerid_value, ...payload } = record
await AccountsService.update(accountId, payload)

// Delete
await AccountsService.delete(accountId)
```

> **Critical:** `IOperationResult` does **not** throw on failure. Always check `result.error`. See `references/dataverse-gotchas.md` for full details including lookup column patterns and filter limitations.

---

## ALM & Configuration

- Configure preferred solutions — never use the default solution
- Use `--solutionName` flag for targeted deployment
- Use connection references for environment portability (not hardcoded connection IDs)
- `power.config.json` holds the `environmentId` — verify it matches `pac env who` output

---

## Troubleshooting Checklist

**Data Source Addition Failures:**
1. Verify environment: `pac env who`
2. Check `power.config.json` environmentId matches
3. Ensure region is `"prod"` (unless intentionally different)
4. Reset authentication: `pac auth create`

**"Data source not found: No Dataverse data source" Runtime Error:**
- Caused by PAC CLI <2.x generating wrong `dataSourceName` in service files
- See `references/dataverse-gotchas.md` for diagnosis and fix

**Zscaler SSL Issues:**
1. Confirm Node.js v22+
2. Export Zscaler root CA to PEM format
3. Set `NODE_EXTRA_CA_CERTS` environment variable
4. Restart terminal

**Build/Deploy Issues:**
1. Check auth: `pac auth list`
2. Verify environment: `pac env who`
3. Confirm build succeeded: `npm run build`
4. Verify Code Apps are enabled in the environment

---

## Current Limitations

- Excel Online connectors not supported
- Dataverse polymorphic lookups not supported
- SharePoint document processing APIs not supported
- Cannot create new connections via PAC CLI (use Power Apps portal)
- No mobile app support
- Browser localhost restrictions (Chrome/Edge December 2025+)

## Migration: SDK v0.3.21 → v1.0

- Remove all `initialize()` imports and calls
- Delete initialization state management
- No longer need to wait for SDK initialization — make data calls immediately

---

## Best Practices

- Use `select` to retrieve only needed columns — improves performance
- Implement pagination (`top`/`skip`) for large datasets
- Never store sensitive data in app code; use environment variables
- Use TypeScript and leverage generated models for IntelliSense
- Always implement proper error handling on `IOperationResult`
- Reference official docs: [Power Apps Code Apps](https://learn.microsoft.com/en-us/power-apps/developer/code-apps/) | [GitHub](https://github.com/microsoft/PowerAppsCodeApps)

---

## Reference Files

Load these when working on specific topics:

- **`references/dataverse-gotchas.md`** — PAC CLI version bugs, IOperationResult error handling, lookup column read/write patterns, virtual property filter limitations
- **`references/file-operations.md`** — CSP constraints explained, complete file upload implementation (3-step block API), complete file download implementation (2-step block API), all critical gotchas
- **`references/document-rendering.md`** — PDF.js main-thread mode for Code Apps, DOCX with mammoth, XLSX with SheetJS, format support matrix, server-side conversion for unsupported formats
