# Structable file format specification

A Structable Document is a fusion of CSV and Markdown: a typed table in `data.csv`, Markdown rowdocs for every row, each its own file under `Rowdocs/`, and saved views and metrics, in one folder that Finder shows as one file. This specification sets out the package's files, their rules and how values are stored.

## Document Structure

Each Structable Document is a macOS package:

```text
Project Tracker.structable/
├── README.md                   # optional document description
├── manifest.json
├── schema.json
├── data.csv
├── views.json                  # optional named filter/sort/group configurations
├── Rowdocs/                    # optional; one folder per rowdoc column
│   └── <column-name>/
│       └── <row-id>.md
├── Metrics/                    # optional; mixed managed/preserved namespace
│   └── <metric-id>/
│       ├── config.json             # Structable-owned
│       └── data.json               # optional Structable-owned derived cache
└── AGENTS.md                   # optional generated package guide
```

`README.md`, `manifest.json`, `schema.json`, `data.csv`, `views.json`, `Rowdocs/*`, and `Metrics/*/config.json` are the native package files. `AGENTS.md` is generated helper guidance for external agents and does not mirror live document structure. When the Structable tool is available, prefer it for invariant-sensitive reads and writes because it preserves package rules and returns structured JSON for automation.

**Preserved members are carried forward.** Structable determines ownership from each normalized package-relative path and member kind. Canonical files and reserved namespaces are Structable-managed; every member classified as preserved is carried through a save unchanged, including hidden members and preserved leaves inside the mixed `Metrics/` namespace. Three categories are deliberately discarded: the retired top-level `Skills` member or tree, regular `.DS_Store` files at any depth, and regular writer-scratch files matching `.<name>.<UUID>.tmp` at any depth. The two filename patterns apply only to regular files; a same-named directory is not discarded on that basis. Structable never interprets preserved members; it only refuses to destroy them.

### manifest.json

```json
{
  "version": "2.0.0",
  "appVersion": "1.0.0",
  "createdAt": "2025-01-22T10:30:00Z",
  "modifiedAt": "2025-01-22T14:45:00Z",
  "documentId": "F81D4FAE-7DEC-11D0-A765-00A0C91E6BF6",
  "rowArchetype": {
    "singular": "Task",
    "plural": "Tasks"
  },
  "lastWriter": {
    "kind": "app",
    "version": "1.0.0",
    "recap": "Added 3 Tasks to Project Tracker"
  }
}
```

`documentId` is required and must be a UUID; a package without a valid one does not open. It is the package's stable identity key: App Intents entities resolve documents by it. The app preserves it across in-place saves, renames, and moves, and regenerates it whenever an action creates a second live package: File > Duplicate, Save As or Save a Copy targeting a new location, and template instantiation. Save As or Save a Copy issued while a package's rows are still loading writes a faithful copy of the original bytes and keeps the original id — regeneration applies once the document is fully loaded. External byte-copies (Finder duplicates, iCloud/Dropbox conflict copies) can share an id. On open, Structable compares the materializing package with the currently open documents at positively distinct filesystem identities: an already-open document wins and the newcomer is re-keyed, with the fresh id persisting through the normal save flow. Two copies that are never open at the same time keep the shared id. Symlink aliases to the same filesystem package do not collide.

An App Intent entity's remembered package URL is a strict identity disambiguator, not a fallback hint. If headless materialization discovers a duplicate id, Structable durably saves the repair, rejects the stale entity without running its requested read or mutation, and returns the replacement id so the caller can refresh and retry. A later request carrying the old id and remembered URL remains a teaching error; it never falls through to a different package that still legitimately owns the old id.

`rowArchetype` is optional document configuration for what one table row represents. `rowArchetype.singular` stores the singular label; empty or absent values are treated as the default `Row`. The optional `rowArchetype.plural` stores an explicit plural label; when absent, the app and the Structable tool derive the plural from the singular at display time.

`lastWriter` records which surface wrote the package most recently. The optional `lastWriter.recap` stores a one-line natural-language summary of that write; the Structable tool persists it so optimistic-concurrency conflicts can report what the most recent write did.

### README.md

`README.md` is the optional user-authored Document Description for the package. It can describe document purpose, context, and any user-authored workflow or handoff rules. It is stored as UTF-8 markdown at the package root, normalized to LF line endings on read/write, and omitted when blank. `AGENTS.md` always tells external agents to read `README.md` first when it exists; it never copies README contents and never requires the file to exist.

### views.json

`views.json` optionally stores the document's ordered saved views. A saved view is a named table-lens configuration only: it can contain structured filters, one column sort, and one group-by column. It never contains row data, search text, column layout, the currently active view, or any other window/session state. Selecting a view in the app is therefore a free, non-persistent action; creating, changing, moving, duplicating, or deleting one is an undoable document mutation.

```json
{
  "views": [
    {
      "id": "m1abc123",
      "name": "Open Issues",
      "rowFilterCombination": "and",
      "rowFilters": [
        {
          "id": "AF0D7C49-D77F-4EF8-82A1-D6E9C8A4F8A2",
          "columnId": "Status",
          "condition": {
            "type": "singleSelectEquals",
            "value": "Open"
          }
        }
      ],
      "sort": {
        "column": "Priority",
        "ascending": false
      },
      "groupByColumn": "Status",
      "icon": "flag"
    }
  ]
}
```

- `views` array order is display order. There is no separate sort-order field.
- `id` is any non-blank text, unique case-insensitively within the file, and stable across renames. Structable-created IDs are lowercase base-36 values; agents should round-trip IDs returned by `structable_describe` instead of deriving them from names.
- `name` is trimmed, non-empty, and unique case-insensitively within the file. View names are user content and are never localized.
- `rowFilterCombination` is `and` or `or`; when absent it defaults to `and`. `rowFilters` uses the same flat persisted shape and condition vocabulary as metric filters. Filter IDs should also be round-tripped to avoid unnecessary Git diffs. IDs must be unique within each view or metric, but can be reused across different views and metrics. CLI/MCP writes reject duplicate filter IDs before persisting any operation in the batch; omitted IDs for new filters receive fresh UUIDs. Every filter ID must be a UUID; a view or metric with any other filter ID is skipped when the package loads.
- `sort` is optional `{column, ascending}`. Its absence means Manual Order. `groupByColumn` is optional and may name only a Single Select, Rating, or Checkbox column. A view cannot sort and group by the same column; normalization keeps grouping and clears that sort.
- `icon` is optional and names one glyph from the curated saved-view icon vocabulary: `list`, `checklist`, `calendar`, `clock`, `flag`, `star`, `tag`, `person`, `people`, `folder`, `tray`, `archive`, `warning`, `bolt`, `flame`, `bookmark`, `envelope`, `cart`, `dollar`, `chart`, `target`, `lightbulb`, `book`, `document`, `map`, `house`, `heart`, or `sparkles`. It is display only and carries no color. The key is written only when an icon is set, so pre-icon files round-trip unchanged. A missing key and an explicit `null` both mean no icon, and neither is reported. Any other value the vocabulary does not contain — an unknown string, or a value written with a non-string type — is removed during load normalization with a non-fatal load issue, and the cleaned file persists on the next save.
- Column references use schema column names. A column rename rewrites saved-view references. Select-option renames through the app and the Structable tool also rewrite matching saved-view and metric filter operands before normalization. Missing or incompatible references and invalid filter operands are removed or cleared during load normalization, with a non-fatal load issue, and the repaired file is persisted on the next save. A stale saved view never blocks document open or a schema edit.
- Blank or later duplicate IDs and blank, untrimmed, or case-insensitively duplicate names are repaired deterministically in array order. The first valid occurrence is preserved. Root-level and per-view unknown fields survive repair.
- The whole file uses the Metrics JSON date-coding regime and pretty-printed, sorted keys. It is omitted when there are no views and no preserved root-level unknown keys.
- A regular root file named `views.json` is Structable-managed. A pre-existing directory with that name is preserved and ignored as foreign content; it has no views fingerprint. Structable refuses a write that would have to destroy such a collision.
- `structable_query` can apply a saved view by ID. Its filters are used instead of an inline filter, its sort is inherited only when the request omits `sort`, and its grouping is ignored because query output has no grouped presentation.

### schema.json

```json
{
  "columns": [
    {
      "name": "Task Name",
      "description": "Short title shown to users and agents.",
      "type": "text",
      "width": 200
    },
    {
      "name": "Priority",
      "type": "single_select",
      "width": 120,
      "options": [
        { "id": "Low" },
        { "id": "Medium" },
        { "id": "High" },
        { "id": "Critical", "color": "Red" }
      ]
    },
    {
      "name": "Due Date",
      "type": "date",
      "width": 120,
      "dateFormat": {
        "dateStyle": "medium",
        "timeStyle": "none",
        "includeTime": false
      }
    },
    {
      "name": "Budget",
      "type": "number",
      "width": 100,
      "numberFormat": {
        "style": "currency",
        "currencyCode": "USD",
        "decimals": 2,
        "groupingSeparator": true,
        "allowNegative": true
      }
    },
    {
      "name": "Created",
      "type": "created_timestamp",
      "width": 150
    },
    {
      "name": "Updated",
      "type": "updated_timestamp",
      "width": 150
    }
  ]
}
```

Canonical write-shape notes:

- `schema.json` stays flat; type-specific configuration lives directly on each column object.
- A supplied `numberFormat` is a complete definition: `style`, `decimals`, `groupingSeparator` and `allowNegative` are required; `currencyCode` is optional. To change one setting, copy the full saved definition and change that setting. Partial configuration updates are not supported.
- select options are canonically written as objects using `id` and optional `color`
- Structable still reads legacy string options, legacy `label` objects, and legacy `checkboxMode` for compatibility, but it does not write them back out in that shape

### data.csv

```csv
Task Name,Priority,Due Date,Budget,Created,Updated,ID
Design mockups,High,2025-02-01,500.00,2025-01-20T09:00:00Z,2025-01-22T14:30:00Z,mngfz756
Implement login,Critical,2025-01-25,1200.00,2025-01-21T10:15:00Z,2025-01-22T16:45:00Z,mngfz757
Write documentation,Low,2025-02-15,200.00,2025-01-22T08:00:00Z,2025-01-22T12:00:00Z,mngfz758
```

When a column uses type `rowdoc`, the value persisted in `data.csv` is the package-relative path to its body file, for example `Rowdocs/Notes/mngfz756.md`. The markdown body itself lives in that `.md` file and is not inlined into the package CSV. The `Rowdocs/<Column>/` folder name is a storage-safe encoding of the column name, so rowdoc column names must remain unique after that normalization. Distinct names that collapse to the same folder are invalid.

An empty body file is a valid persisted state: the cell keeps its `.md` reference and the file is zero-length. It is distinct from an unpopulated cell, which stores no reference and has no file. Agent-surface set/write operations persist an empty body exactly as offered and never delete; only the explicit rowdoc-cell delete operations clear the reference and remove the file.

### Metrics/config.json

Metrics are stored outside `schema.json` under `Metrics/<metric-id>/` and use `config.json` as the canonical widget source:

`Metrics/` is a mixed-ownership namespace. Structable owns only regular files exactly at `Metrics/<metric-id>/config.json` and `Metrics/<metric-id>/data.json`. Subject to the global discarded-member rules above, every other member under `Metrics/` is preserved, including empty mixed-container directories. A save rewrites current widget configs, writes or removes each optional cache to match the save snapshot, and removes only the owned leaves for deleted widgets; it does not remove a mixed-container directory merely because no owned leaves remain. If a preserved member conflicts by kind, case, or Unicode spelling with a managed Metrics path required by the current widgets, the save fails before mutation rather than replacing that member.

```json
{
  "id": "total-budget",
  "title": "Total Budget",
  "widgetType": "stat",
  "colorTheme": "Primary",
  "sortOrder": 1000000,
  "rowFilterCombination": "and",
  "rowFilters": [
    {
      "id": "AF0D7C49-D77F-4EF8-82A1-D6E9C8A4F8A2",
      "columnId": "Due Date",
      "condition": {
        "type": "dateRelative",
        "range": "last_30_days"
      }
    }
  ],
  "statConfig": {
    "valueColumnId": "Budget",
    "function": "sum"
  },
  "trendConfig": null,
  "breakdownConfig": null
}
```

- `widgetType` determines which nested config object is populated: `statConfig`, `trendConfig`, or `breakdownConfig`
- metric IDs are normalized from titles using lowercase letters and digits with non-alphanumeric runs collapsed to `-`; if nothing remains after normalization, use `metric`
- metric titles must stay unique after that normalization; renaming a metric title also rewrites `config.json.id`, `data.json.widgetId`, and the `Metrics/<metric-id>/` folder on the next write
- widget column references such as `dateColumnId`, `valueColumnId`, `groupByColumnId`, and `rowFilters[].columnId` use schema column names
- `rowFilterCombination` controls whether widget row filters use `and` or `or`; older configs that omit it default to `and`
- widget row filters support checkbox, single-select, multi-select, rating, text, URL, email, phone, number, progress, date, `created_timestamp`, and `updated_timestamp` columns
- single-select filters use `singleSelectEquals`, `singleSelectNotEquals`, `singleSelectIn`, and `singleSelectNotIn`; negative conditions match only non-empty single-select cells, and continue matching newly added options unless those options are explicitly excluded
- text-like filters use `textContains`, `textDoesNotContain`, `textEquals`, `textNotEquals`, `textStartsWith`, `textEndsWith`, `textIsEmpty`, and `textIsNotEmpty`
- number and progress filters use `numberEquals`, `numberNotEquals`, `numberGreaterThan`, `numberGreaterThanOrEqual`, `numberLessThan`, `numberLessThanOrEqual`, `numberBetween`, `numberIsEmpty`, and `numberIsNotEmpty`
- multi-select filters use `multiSelectContainsAny`, `multiSelectContainsAll`, `multiSelectContainsNone`, `multiSelectIsEmpty`, and `multiSelectIsNotEmpty`
- date filter `value`, `start`, and `end` accept both cell encodings on read: a calendar date such as `2026-07-04` or a full ISO 8601 instant. Metrics writers emit the same canonical UTC instant string, preserving milliseconds, across directory saves, file-wrapper saves, fingerprints, and the Structable tool's metric/filter surfaces; surrounding JSON formatting is not part of that value-level guarantee
- `dateOn` matches the selected calendar day for date-only columns and the selected minute for date-time and timestamp columns
- invalid widget row filters are normalized on load: missing columns, incompatible condition types, empty text operands, stale select options, empty select lists after cleanup, and out-of-range rating or progress values are removed, while reversed number/date `between` bounds are reordered. Stale cached widget data for affected widgets is discarded before recomputation
- `Metrics/*/data.json` stores an optional last-known full-document derived cache and may be ignored or deleted safely; the app regenerates it. `resultChangedAt` records when that canonical full-document result last changed. Trend points and breakdown segments do not have persisted item IDs. Breakdown widget caches are display-ready and collapse more than 8 groups into the top 7 groups plus an `Other`. Other applies the selected function to the overflow categories' underlying values. Count counts category memberships; numeric functions ignore empty values.
- when a document window scopes inspector metrics to its applied table filter/search result, those values are a transient in-memory presentation overlay. They never enter `Metrics/*/data.json`, the canonical metrics cache, dirty state, save snapshots, or the Structable tool's output

### CSV Encoding Rules

- UTF-8 encoding always; invalid byte sequences are rejected with record and field diagnostics instead of replacement characters
- readers tolerate a single leading UTF-8 byte order mark; writers never emit one
- first row is the header, required even for an empty table; a zero-byte or BOM-only file is invalid
- records wider than the schema columns plus `ID` are rejected before rows become live; shorter records retain empty trailing values and deterministically repair an omitted ID
- CSV row order is the canonical manual order
- stable row identity is persisted through a trailing `ID` column in `data.csv`. An ID is lowercase letters and digits only, unique in the document, such as `mngfz756` or `task001`. Structable makes new IDs from the current time in milliseconds, written in base 36. Capitals are lowercased on load, but a document does not open if a row with capitals has a rowdoc; an ID with any other character is replaced by a new one (note 9)
- headers in `data.csv` must follow `schema.json` column order, then `ID`, and match column names case-insensitively
- the row-number column is UI-only, always shown at visual column 0, and is never persisted to `data.csv`
- optional `manifest.json.rowArchetype.singular` stores the singular row label, such as `Contact`; blank or absent means `Row`; optional `rowArchetype.plural` overrides the derived plural
- `ID` is a reserved public system field, not a schema column
- date columns whose format omits time store calendar dates as date-only ISO 8601 (`YYYY-MM-DD`); Structable supports Gregorian years `1583...9999`, beginning with the first full year after the native system calendar's historical cutover so package validation, rendering, editing, filtering, and metrics share one calendar interpretation. ISO-looking values outside that range remain text after normal surrounding-space trimming on package load and are rejected by strict agent writes. A supported calendar date is timezone-independent and canonically the UTC midnight of its day, so the same file names the same day on every machine
- date columns whose format includes time, and timestamp columns, canonically write UTC-normalizable ISO 8601 instants in the form `YYYY-MM-DDTHH:MM:SS[.fraction](Z|z|signHH|signHH:MM|signHHMM)`, where `sign` is `+` or `-`; strict agent writes reject date-only input for these columns, while native package reads accept legacy `YYYY-MM-DD` values as UTC midnight and canonicalize them to a full instant on the next write. Both the written civil year and normalized UTC year must remain in `1583...9999`
- instant fractions are persisted canonically at millisecond precision: one or more fractional digits are accepted, digits beyond the first three are intentionally truncated on intake, trailing zeroes are omitted on write, and integral-second values retain the `...:SSZ` form
- migration is lenient-read plus canonicalize-on-write: an exact UTC-midnight value in a day-granularity date column remains that canonical day; a full instant with an explicit numeric UTC offset preserves the `YYYY-MM-DD` day written in that offset; other legacy full instants (notably historical picker values serialized with `Z`) use the migration process's current timezone when the instant is local midnight there, otherwise they fall back to the nearest UTC day. The value is rewritten date-only on the next save. This recovers local-midnight picker writes across fractional offsets, DST, and UTC+14 in the writer's timezone while preserving the established result for most packages moved between timezones. A non-midnight `Z` instant carries no writer-timezone metadata, so moved legacy values beyond UTC+12 remain inherently ambiguous until canonicalized.
- toggling a date column's include-time setting from `true` to `false` snaps every existing row instant and every absolute `on` / `before` / `after` / `between` operand in metric row filters that reference the column to the nearest UTC calendar day under the established rounding policy; relative and empty-state filter modes do not carry absolute operands and remain unchanged. All row and metric-filter snaps are preflighted before changing the schema; if any resulting day would fall outside `1583...9999`, the entire edit is rejected without changing schema, rows, metrics, or package bytes. Toggling from `false` to `true` keeps the canonical midnight values and re-encodes them as full UTC instants on the next write
- booleans are stored as `true` / `false`
- numbers are stored as plain decimals, without grouping separators, currency symbols, or percent signs
- progress values are stored as raw decimals in `0...1`
- progress persistence/import parsing is locale-neutral and never strips commas; an ambiguous value such as `0,1` stays text instead of becoming `1`. Human-facing filter input separately accepts the current locale's decimal separator and percent symbol.
- native `data.csv` date parsing is locale-neutral: non-ISO localized text stays text instead of changing meaning between machines. The explicit user-facing foreign CSV import flow may additionally interpret dates using the current locale, but localized results must remain within both UTC persistence bounds and the displayed civil-year bounds. Human-facing instant pickers use the same intersection so extreme offsets cannot display year 1582 or 10000. Day-only pickers instead bind directly to canonical UTC-midnight dates across the full `1583-01-01...9999-12-31` range, including civil days skipped by a local timezone.
- rowdoc columns store package-relative `.md` paths such as `Rowdocs/Notes/<row-id>.md`; the file name must be the row's `ID` exactly
- multi-select values use `|` as the delimiter, escaping literal `\` as `\\` and literal `|` as `\|`
- standard CSV escaping rules apply: quote fields containing commas, quotes, or line breaks, and double embedded quotes

### Important Notes

1. **Row numbers are generated** - they are derived from physical CSV row order and are not stored in the package; stable row identity is instead persisted via the trailing `ID` column.
2. **Column names are identifiers** - Structable treats column names as stable schema identifiers, requires case-insensitive uniqueness, and for `rowdoc` columns also requires uniqueness after the storage-folder normalization used for `Rowdocs/<Column>/...`. Readers enforce these identity invariants at load: a package whose `schema.json` contains case-insensitive duplicate names, an empty name, a column claiming the reserved `ID` name, or colliding rowdoc storage folders is rejected with a teaching error identifying the offending column; unlike the value-level leniency in note 10, identity corruption is never repaired silently.
3. **Schema stays flat** - type-specific configuration stays in dedicated fields such as `numberFormat`, `dateFormat`, `options`, `checkboxIcon`, `checkboxColor`, `progressColor`, and `ratingColor`.
4. **Column descriptions are optional** - they power header help and can help external agents, but are not required.
5. **Generated guidance stays static** - `AGENTS.md` points external agents to native files and global rules, but it does not duplicate the current schema or widget list. The app's own saves do not rewrite it; they carry the existing file forward like any other member they did not author.
6. **Row archetypes are manifest-backed** - `manifest.json.rowArchetype.singular` optionally stores the singular row label; absent or blank falls back to `Row`; `rowArchetype.plural` optionally overrides the derived plural.
7. **Document descriptions are README-backed** - the optional root `README.md` stores user-authored package purpose/context and is removed when blank.
8. **Rowdoc bodies are file-backed** - populated `rowdoc` cells keep their markdown bodies in `Rowdocs/<Column>/<RowID>.md`, while `data.csv` stores only the relative reference path. A zero-length body file with a live reference is a valid state, distinct from an unpopulated cell.
9. **Row IDs and their rowdoc references are repaired within the native format** - if the `ID` column is present but a row ID is omitted, blank, duplicated, invalid, or requires normalization, Structable repairs it on load. Each repaired row's populated `rowdoc` cells with unambiguous package-path spelling are canonicalized to `Rowdocs/<Column>/<RowID>.md`; references in rows whose IDs did not change are left alone. The next write persists the row repair and copies the exact referenced source bytes to each canonical target, including when multiple repaired cells shared one source. Migration planning retains every raw Unicode spelling through collision validation and rejects a missing or invalid source, an already occupied target (including one owned by an unchanged row), or case- or Unicode-equivalent source, target, or final reference components with different spellings before persistence mutation, leaving package bytes unchanged for manual correction.
10. **Agent-surface writes validate strictly** - imports into an existing schema reject unknown headers, duplicate case-insensitive destinations (including `ID`), and surplus unmapped fields before mutation. Row-write objects reject duplicate case-insensitive keys. The Structable tool's row writes and delimited imports reject values that do not match the column's stated encoding (teaching errors instead of coercion), match select values case-insensitively against option IDs, and persist the canonical ID. JSON row writes (create, add and update) reject unknown select options before saving; the error lists valid options and may suggest a close match without applying it. To create an option intentionally, add it to the column first with the `add_option` column action. Delimited imports create unknown options with assigned colors and report them in `createdOptions`. Day-granularity date columns accept only calendar dates like `2026-07-04`; a time-bearing ISO 8601 value is rejected with a teaching error naming the calendar-date encoding, while date columns that include time and timestamp writes accept the strict instant grammar above. Load-time parsing of hand-edited files is locale-neutral: invalid or non-ISO date/timestamp values remain text after normal surrounding-space trimming instead of being coerced, and unknown select options in `data.csv` are cleared when the package loads, so direct file edits must add options to `schema.json` first. Number and progress cells read leniently but full-string: US-grouped (`1,234.56`), currency-prefixed, exponent, and percent forms are accepted whole — a percent string like `50%` reads as its semantic value `0.5` — and an unparseable value remains text after the same normalization instead of being truncated or dropped.

JSON row inputs also accept native numbers for number, progress and rating columns, and native booleans for checkbox columns. Native numbers must have at most 15 significant decimal digits, magnitude at most 9007199254740991, and an exact decimal round trip through the transport and decimal storage; scientific notation is normalized to plain decimal text. Values outside that policy must use quoted plain decimal strings. Column ranges still apply; numbers and booleans are not coerced into text, select or date columns. Strings follow the encoding rules above, null writes an empty cell, and `data.csv` stores the same text whichever form a value arrives in.

### Format Versioning & Compatibility

`manifest.json.version` carries the package format version as semantic versioning. The current writable format is **2.0.0**.

Readable band:

- Structable reads packages whose version has major `2` and minor `0`; the patch component is ignored, so the readable band is `2.0.x`
- packages outside the band are rejected at the read boundary with a teaching error naming the supported formats
- the reader does not migrate older packages; format 2.0 changes the stored column type to `rowdoc` and the body folder to `Rowdocs/`
- every save pins `manifest.json.version` back to the latest writable version, regardless of which 2.0.x patch the package carried on load

Version-bump rules:

- **patch bumps** (`2.0.0` → `2.0.1`) may introduce additive fields at the preserved levels below; readers in the band do not understand those fields but tolerate and preserve them through the unknown-key guarantee, so no reader update is required
- **minor bumps** (`2.0.x` → `2.1.0`) signal changes a reader must understand to interpret the package correctly; the readable band widens only after every reader in the band handles them
- **major bumps** (`1.x` → `2.0`) signal breaking changes; older readers must refuse the package
- precedent: the calendar-date encoding for day-granularity date columns (date-only `YYYY-MM-DD` cell values) shipped with no bump — every reader in the band already parsed date-only ISO 8601 date values, so canonicalizing the writer onto an already-readable shape introduces nothing a reader must newly understand
- `views.json` was an additive `1.2.x` member adoption. Older readers classify the then-unknown member as preserved and carry it byte-for-byte; saved-views-aware readers manage a regular file at that path while continuing to preserve a wrong-kind collision.

Unknown-key preservation:

- unknown keys at the **top level** of `manifest.json`, `schema.json`, `views.json`, each `schema.json` column entry, each `views.json.views[]` entry, and each `Metrics/*/config.json` are preserved verbatim across saves from the app, CLI, and MCP server — including through document edits, `documentId` regeneration, and the save-time version pinning above
- preserved means preserved, **not understood**: Structable never validates, interprets, or migrates unknown keys; they are carried as opaque JSON
- keys inside Structable's reserved vocabulary (the current coding keys of each file, including legacy compatibility keys such as `checkboxMode`) are Structable-owned: they may be normalized, rewritten, or dropped on save exactly as before
- if a future format version adopts a previously unknown key into its reserved vocabulary, that version's readers own the key from then on — a preserved third-party key that collides with a later official key is read (and may be rewritten) under the official semantics, so third parties should namespace custom keys (for example an `x-`/vendor prefix)
- unknown keys **nested inside known sub-objects** (for example inside `lastWriter`, `rowFilters` entries, select option objects, or `statConfig`) are **not** preserved; additive 2.0.x fields must be introduced at the preserved top levels
- `Metrics/*/data.json` is regenerated derived cache and carries no preservation guarantee
- template instantiation is app-authoring, not copying: unknown keys preserved in a template package do not follow instantiated documents
- unknown numeric values round-trip through a double-precision representation: integers beyond 2^53 (and other pathological numbers) may lose precision or re-serialize in a different textual form
- an unknown key whose value cannot be represented at all (for example a number beyond double range such as `1e999`) never fails the read: the key is dropped on load — reported as a non-fatal load issue where the read path surfaces one — and does not survive the next save

---

## Column Types

| Type | Storage Format |
|------|----------------|
| `text` | Plain string |
| `rowdoc` | Relative `.md` path in `data.csv`; body in `Rowdocs/<Column>/<RowID>.md` |
| `number` | Decimal string |
| `progress` | Decimal string in `0...1` |
| `date` | Date-only ISO 8601 calendar date (`YYYY-MM-DD`); full ISO 8601 instant when the format includes time |
| `checkbox` | `true` / `false` |
| `single_select` | Option string |
| `multi_select` | Pipe-delimited string |
| `rating` | Integer string |
| `url` | URL string |
| `email` | Email string |
| `phone` | Phone string |
| `created_timestamp` | ISO 8601 string |
| `updated_timestamp` | ISO 8601 string |

### Timestamp Rules

- only one `created_timestamp` column and one `updated_timestamp` column are allowed
- readers tolerate legacy or hand-made duplicates of a timestamp role; automatic bumps target the first column of that role in schema order, so any additional column of the role drifts stale but round-trips losslessly
- timestamp columns are read-only in direct cell editing
- timestamp columns cannot be renamed
- timestamp columns cannot be converted to or from non-timestamp types
- undo restores the previous `updated_timestamp` value when applicable

### Column Editing Rules

- users choose a column type when creating a new column
- existing columns keep their type after creation; changing type requires creating a new column instead

### Number Formatting

Number styling is display-only. CSV storage remains plain decimal text even when the user sees:

- decimal formatting
- currency formatting
- percent formatting
- scientific formatting
- measurement units

### Checkbox, Progress, Rating, and Select Configuration

- checkbox columns always persist `true` / `false` regardless of presentation
- progress columns persist raw decimal values and use `ColorTheme` only for display color
- select-option colors and rating/checkbox icon colors use shared `ColorTheme` vocabulary
- current `ColorTheme` values are `Indigo`, `Blue`, `Green`, `Red`, `Orange`, `Purple`, `Pink`, `Teal`, `Cyan`, `Mint`, `Brown`, `Violet`, `Plum`, `Primary`, and `Gray`
- icon values persist Structable icon tokens rather than SF Symbol names
- canonical enum-like values are the raw strings written in `schema.json` and `Metrics/*/config.json`
- legacy read compatibility still accepts older option encodings and checkbox presentation fields, but Structable writes only the current canonical shape

### Widget Configuration

- `Metrics/*/config.json` is the canonical widget file; `Metrics/*/data.json` is an optional last-known full-document derived cache with `resultChangedAt` and semantic metric values only
- `widgetType` uses `stat`, `trend`, or `breakdown`
- `sortOrder` is an integer; metrics are listed in ascending `sortOrder`
- `colorTheme`, `statConfig.function`, `trendConfig.function`, `trendConfig.resamplingPeriod`, `breakdownConfig.function`, `rowFilterCombination`, `rowFilters[].condition.type`, and `rowFilters[].condition.range` all persist app-defined raw enum strings
- widget configs stay document-specific, but they do not alter the `schema.json` contract

---

- **Format version:** 2.0.0
- **Date:** October 15, 2026
