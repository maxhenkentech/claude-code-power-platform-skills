# Dataverse Gotchas & Known Bugs

## Known Bug: "Data source not found: No Dataverse data source" Runtime Error

**Symptom:** `Hek_*Service.getAll()` (or any Dataverse call) returns:
```
{success: false, error: Error: Retrieve multiple records operation failure: Data source not found: No Dataverse data source…}
```
even when authenticated and the environment ID matches.

**Root cause:** PAC CLI versions before ~2.x generated service files where `dataSourceName` used the friendly camelCase name (e.g., `'llModels'`) instead of the Dataverse entity set name (e.g., `'hek_llmodels'`). The SDK runtime keys its data source dictionary by `entitySetName`, so the lookup fails.

**Diagnosis:** Open any generated `src/generated/services/*Service.ts` and check:
```typescript
private static readonly dataSourceName = 'llModels'; // BUG: friendly name
private static readonly dataSourceName = 'hek_llmodels'; // CORRECT: entitySetName
```

**Fix — Upgrade PAC CLI to 2.x (recommended):**
1. Check current version: `pac --version`
2. If below 2.x, upgrade. PAC CLI 2.x requires .NET 10:
   ```bash
   # Install .NET 10 arm64 (Apple Silicon)
   curl -sSL https://dot.net/v1/dotnet-install.sh | bash -s -- --channel 10.0 --architecture arm64 --install-dir ~/.dotnet
   # Install latest PAC CLI
   ~/.dotnet/dotnet tool install --global Microsoft.PowerApps.CLI.Tool
   ```
3. Delete and re-add all Dataverse data sources so the generator re-creates service files with correct names:
   ```bash
   pac code delete-data-source -a dataverse -ds <dataSourceName>
   pac code add-data-source -a dataverse -t <table-logical-name>
   ```
4. Verify the regenerated files now use `hek_*` entitySetName values.

**Quick manual fix (if upgrade not immediately possible):**
Edit each affected `*Service.ts` and change `dataSourceName` to match the table's logical entity set name (e.g., `'hek_llmodels'` not `'llModels'`).

**Note:** After upgrading to PAC CLI 2.x the code generator also changes field types — money/integer/decimal fields become `string` (matching OData reality), navigation properties become `object`, and option sets get enum types. Update app code accordingly:
- Fields like `costin`, `costout`, token counts: store and compare as strings, convert with `parseFloat()`/`parseInt()` only for display
- Lookup GUIDs: use `_hek_model_value` (read) and `"hek_Model@odata.bind": /entity(guid)` (write)
- Boolean option sets: now typed as `0 | 1` numeric enum

---

## IOperationResult Does Not Throw on Failure

`create()`, `update()`, and `get()` return `IOperationResult<T>`. Dataverse errors land in `result.error` — they do **not** throw. A bare `await Service.create(...)` inside `try/catch` will silently appear to succeed even when Dataverse rejected the record.

Always capture and check the result:

```typescript
const result = await MyService.create(payload)
if (result?.error) {
  const msg = result.error?.message ?? JSON.stringify(result.error)
  // surface error to user — do NOT proceed
  return
}
// only now treat as success
```

---

## Lookup Columns: Read vs Write

Dataverse lookup columns produce two SDK properties:

| Property | Purpose | Writable? |
|---|---|---|
| `_tablename_value` | GUID of related record (read) | **No** |
| `"Table@odata.bind"` | Navigation property (write) | **Yes** |

Including a `_*_value` property in a create/update payload causes: `CRM do not support direct update of Entity Reference properties, Use Navigation properties instead` (error `0x80060888`).

**Pattern:** store both in state (the GUID drives dropdown display), but strip `_*_value` before writing:

```typescript
// In state
{ _hek_model_value: guid, 'hek_Model@odata.bind': `/hek_llmodels(${guid})` }

// Before create/update — strip the read-only property
const { _hek_model_value, ...payload } = record
await MyService.create(payload)
```

---

## Lookup GUIDs Are Not Returned Unless Explicitly Selected

When using a `select` array on `get()`, `_*_value` lookup GUIDs are **not** returned unless explicitly listed — even though they appear in the model type.

```typescript
// ❌ _related_value will be undefined
select: ['name', 'description']

// ✅
select: ['name', 'description', '_related_value']
```

---

## Do NOT Filter on `_*_value` Virtual Properties — Use Client-Side Filtering

**Symptom:** `getAll({ filter: '_hek_documentbundle_value eq <guid>' })` returns zero results despite records existing in Dataverse. No error is returned — `relsResult.error` is null and `relsResult.data` is `[]`.

**Root cause:** `_*_value` properties are OData virtual annotations on navigation properties. They appear in the generated TypeScript model interface but are **NOT declared in the entity's schema JSON**. The SDK's query builder validates filter fields against the schema and silently drops or ignores unrecognised fields, resulting in an unfiltered or broken query that returns nothing.

**OData GUID filter syntax is also easy to get wrong:**
- ✅ Correct OData v4: `_fieldname_value eq 3f2504e0-4f89-11d3-9a0c-0305e82c3301` (bare GUID, no quotes)
- ❌ Wrong (single quotes): `_fieldname_value eq '3f2504e0-...'` — causes syntax error or no results
- ❌ Wrong (old v8 style): `_fieldname_value eq guid'3f2504e0-...'` — not supported in v9

**Reliable fix — fetch all, filter client-side:**

```typescript
// ❌ Unreliable — SDK may silently ignore the $filter on virtual _*_value property
const relsResult = await RelationshipsService.getAll({
  filter: `_hek_documentbundle_value eq ${bundleId}`,  // may be stripped silently
})

// ✅ Reliable — fetch all (with a generous top), filter in JS
const relsResult = await RelationshipsService.getAll({
  orderBy: ['hek_order asc'],
  top: 5000,
  // NO filter — let Dataverse return everything, filter client-side
})
const rels = (relsResult.data ?? []).filter(
  r => r._hek_documentbundle_value === bundleId
)
```

Since no `$select` is specified, Dataverse returns ALL fields including `_hek_documentbundle_value` and `_hek_document_value` naturally.

**For secondary per-record lookups, prefer individual `get()` over batch filter:**

```typescript
// ❌ Batch filter on primary key — may fail if SDK validates filter against schema
const filterStr = docIds.map(id => `hek_lldocumentid eq ${id}`).join(' or ')
const docsResult = await DocsService.getAll({ filter: filterStr })

// ✅ Individual get() per record — primary-key retrieval is always reliable
const results = await Promise.all(
  docIds.map(id => DocsService.get(id, {
    select: ['hek_lldocumentid', 'hek_filename', 'hek_documenttype'],
  }))
)
```
