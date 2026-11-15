# Build brief: Pilar column-layout grid for Dataverse (HTML web resource)

You are building a **model-driven Power Apps HTML web resource** that shows Dataverse records in a polished
data table, where each user can resize, reorder, pin and hide columns and change row density, and their layout
is saved per user in Dataverse. Build it exactly as specified here. **Appendix A has the complete source of every
project file**: reproduce those files verbatim unless a section below tells you to change something. The narrative
sections explain why each piece exists, so you can adapt it safely.

---

## 1. What "done" looks like

- A page that loads records from the **Dataverse Web API** (`/api/data/v9.2/`) using the signed-in user's session:
  no tokens, no CORS, and the user's security roles apply.
- **Default target:** the standard `account` table, with columns Account, Number, Status (chip), Primary contact, City,
  Phone, Annual revenue, Employees, Owner, Modified. It's switchable through one config file.
- **Column features**, provided by the third-party component **Pilar**:
  - drag a header edge to resize (double-click to fit)
  - drag a header to reorder
  - pin to the left (sticky)
  - show or hide from a "Columns" menu
  - density: Comfortable / Normal / Compact
  - Reset
  - full keyboard support
- **Per-user layout persistence** in a user-owned Dataverse table. It's restored on load, with no animation or flicker
  before the table appears.
- **Double-click a row** to open the record (`Xrm.Navigation.openForm`, falling back to `main.aspx` in a new tab).
- **Clear error states:** for example, a missing read privilege shows a message and a **Try again** button.
- **Graceful fallback:** if the layout table doesn't exist, layouts save to `localStorage` and a notice says so.
- **English UI throughout.** Pilar ships in Spanish, and a build-time patch translates it.
- **A local mock of the Web API**, so everything can be built and tested without an environment.
- **15 automated tests passing** and a clean TypeScript type-check.

## 2. Environment and tooling

- **Browser code** (`src/`): TypeScript bundled into a single classic-script IIFE (`app.js`). No framework, no CDN, no web
  fonts; tenant CSPs often block external hosts. Target: current Edge/Chrome.
- **Tooling:** **Bun** (runtime, bundler, test runner) for the build script, mock server and tests.
  - If Bun isn't approved, port the tooling only: `Bun.serve` → `node:http`, `Bun.build` → `esbuild --format=iife`,
    `bun:test` → `vitest`, `Bun.file`/`Bun.write` → `node:fs`. The browser code needs no changes.
- **Dev dependencies:** `typescript`, `@types/bun`. **Runtime dependencies:** none.

## 3. Third-party component: Pilar by Moxen

- **What:** a free, dependency-free HTML/CSS/JS column manager for data tables (`pilar.js`, `pilar.css`, demo
  `index.html`, `README.md`, `LICENCIA.txt`).
- **Get it:** a person must download it from `https://moxen.cl/recursos` (search "Pilar" → Descargar; the site shows a
  free public coupon on the download dialog). Put the five files in `vendor/moxen-pilar/` and **never edit them**. All
  changes happen at build time (section 4).
- **Licence (LICENCIA.txt):** use and modify freely in your own and your clients' projects. **Do not** redistribute the
  source as a resource, template or library, or publish it in a public repo. Never commit `vendor/moxen-pilar/` to a
  public repository; keep any repo that contains it internal.
- **API** (as used here):
  ```js
  const p = Pilar(rootElement, {
    columnas: [{ id, nombre, tipo /* 'id'|'estado'|'texto'|'num'|'mono' */, ancho, min?, max? }],
    filas: [[/* one string per column, in column order */]],
    densidad: 'normal',            // 'comoda' | 'normal' | 'compacta'; what Reset returns to
    empujaAlFijar: true,           // small scroll "nudge" after pinning
    alCambiar: (estado) => {},     // called whenever the layout settles
    estados: { [value]: { texto, tono } }  // added by our patch (status chips)
  });
  p.mueve(id, pos); p.fija(id, bool); p.oculta(id, bool); p.ancho(id, px);
  p.densidad(d); p.ajusta(id); p.restablece();
  p.estado(); // -> { orden: string[], anchos: {id: px}, fijas: string[], ocultas: string[], densidad }
  ```
- **Required DOM hooks** (Pilar looks these up inside the root): `[data-pilar]` (root), `[data-columnas]`, `[data-menu]`,
  `[data-menu-lista]`, `[data-menu-cuenta]`, `[data-todas]`, `[data-densidad]` with `button[data-d=comoda|normal|compacta]`,
  `[data-restablece]` containing `.pi-rest-a` and `.pi-rest-b`, `[data-scroll]`, `[data-tabla]`, `[data-estado]`,
  `[data-cuenta]`, `[data-vivo]`. Use `src/index.html` from Appendix A as is.
- **Behaviours the restore logic depends on** (verified by reading pilar.js):
  1. `fija(id, true)` does nothing if the column is hidden, so **pin before hiding**.
  2. Pinning moves the column to the **end of the pinned group**.
  3. `mueve(id, pos)` counts `pos` **within the column's own group** (pinned or unpinned), excluding the column itself.
  4. Every layout call fires `alCambiar`, so ignore it while restoring.
  5. `quieto()` (Pilar's "skip animations" check) is `matchMedia('(prefers-reduced-motion: reduce)')`; widths animate via a
     spring loop on `requestAnimationFrame`.
  6. Each body row gets the inline CSS variable `--i` = its index in `filas`. Use that to map a row back to its record id.
  7. Pilar only manages columns: no sorting, filtering, searching, paging or editing.

## 4. Build-time patch of pilar.js (`scripts/patch-pilar.ts`)

The script applies exact string replacements to `vendor/moxen-pilar/pilar.js`. **Each replacement must match the expected
number of times or the build fails**, so a new Pilar version can't half-patch silently. It makes three changes:

1. **Quiet mode:** `quieto()` also returns true when `<html data-pilar-quieto>` is present, so a layout restore applies
   instantly with no animation.
2. **Configurable status chips:** the hard-coded Spanish `ESTADOS` lookup becomes `op.estados || ESTADOS`, with object
   values `{ texto, tono }`, and `tono` is written to `data-e` for styling.
3. **English text:** every user-facing and screen-reader string. The full replacement table is in Appendix A.

## 5. Configuration (`src/config.ts`)

- **One exported `config` object:**
  - `gridId` (the layout key), `title`, `subtitle`
  - `entityName`, `entitySet`, `idField`
  - `columns: { field, label, kind, width }[]`, where `kind` is one of
    `text|id|number|money|date|choice|lookup|status`
  - optional `filter` / `orderBy` (OData), `pageSize` (100), `maxRows` (500)
  - `statusTones` (option value → `filled|outline|muted|dark`)
  - `layoutStore`: `{ kind: "dataverse", entitySet, idField, keyField, layoutField }` or `{ kind: "local" }`
- **For lookups,** `field` is the lookup's logical name. It's selected and read as `_<field>_value`.
- **The layout-table names must use the solution publisher's prefix** (the default `new_` is a placeholder).

## 6. Modules (all in Appendix A)

| File | Responsibility |
|------|----------------|
| `src/mapping.ts` | Builds the query (`$select` = id + columns, lookups as `_x_value`; URL-encoded `$filter`/`$orderby`). Maps records to Pilar input: formatted value (`<prop>@OData.Community.Display.V1.FormattedValue`) for every kind except `text`/`id`; `—` for empty; for `status`, the raw option value plus an `estados` map built from the formatted labels and `statusTones`. |
| `src/dataverse.ts` | `fetch` wrapper with OData headers (`Accept`, `OData-MaxVersion`, `OData-Version`) and `credentials: same-origin`; parses error bodies into `DataverseError(status, code, message)`. `retrieveAll` sends `Prefer: odata.maxpagesize=N,odata.include-annotations="OData.Community.Display.V1.FormattedValue"` and follows absolute `@odata.nextLink`s up to `maxRows`. Also `whoAmI()`, `create()` (id from the `OData-EntityId` header), `update()` (PATCH with `If-Match: *`) and `isMissingResource()` (404, `0x80060888`, "Resource not found"). |
| `src/layout.ts` | `planRestore(saved, defaults)` is a pure function that returns ops in this order: pins (saved pin order) → `mueve` for each group left to right → widths that differ → hides (never all) → density. Unknown saved columns are dropped and new columns go to the end. Also `applyRestore`, `LocalLayoutStore` (try/catch around `localStorage`) and `DataverseLayoutStore` (one record per user per grid, filtered by `keyField eq '<key>' and _ownerid_value eq <userId>`; single quotes doubled, GUID unquoted; POST first, then PATCH). |
| `src/host.ts` | `getXrm()` (`window.Xrm` or `parent.Xrm`, wrapped in try/catch), `isLocalDev()`, `isEmbedded()`, and `openRecord()` (Xrm, then a mock-mode toast, then `main.aspx?etn=&id=&pagetype=entityrecord` in a new tab). |
| `src/app.ts` | Bootstrap, described below. |
| `src/index.html` | Pilar's markup hooks with English labels, plus a header status line (`[data-mode]`, `[data-count]`, `[data-saved]`), a loading/error panel (`[data-panel]`, `[data-panel-text]`, `[data-retry]`) and a toast (`[data-toast]`). Loads `pilar.css`, `grid.css`, `pilar.js` and `app.js` by **relative** paths. |
| `src/grid.css` | Segoe UI font stack, the four chip tones, an edge-to-edge layout when inside an iframe (`[data-embebido]`), the header status line, the panel/spinner, `[data-restaurando] .pi-scroll { visibility: hidden }`, and the toast. |
| `scripts/build.ts` | `--prefix <publisher>` (2–8 lowercase letters or digits) and `--folder pilargrid`. Writes `dist/<prefix>_/<folder>/` with patched `pilar.js`, `pilar.css`, `grid.css`, `index.html` and the IIFE `app.js`, then prints the web resource names and types. |
| `mock/server.ts`, `mock/data.ts` | The local Web API stand-in (section 8). |

**What `app.ts` does on start-up:**
1. Marks the page as embedded if it's inside an iframe, sets the titles and the mode badge ("Mock Web API" on localhost,
   otherwise "Dataverse"), and shows the loading panel.
2. Runs in parallel:
   - `retrieveAll(buildQuery(config))`
   - opening the layout store: `whoAmI` + `load`; on a missing table, or any error, fall back to `LocalLayoutStore`
     with a toast explaining why
3. On a data error, shows the panel with a human message (401/403 → "You don't have permission to read …"), plus
   **Try again** (reloads the page). With zero rows, shows an empty message.
4. Creates Pilar with `columnas`, `filas`, `estados` and `alCambiar` → a debounced (700 ms), serialised saver: one save
   at a time and the newest state wins, which prevents duplicate creates. The header shows saving, saved or failed.
5. If there's a saved layout:
   1. set `<html data-pilar-quieto>` and `[data-restaurando]`, then `applyRestore(planRestore(saved, pilar.estado()))`
   2. wait for **two animation frames or 150 ms, whichever comes first**, then reveal the table
   3. remove quiet mode after two frames once the page is visible (use `visibilitychange` if it's hidden)
6. Turns restoring off and hides the panel.
7. Double-clicking a body row reads `--i`, looks up the id and calls `openRecord`.

## 7. Dataverse contract

- **Requests the page makes** (all relative to the org):
  - `GET accounts?$select=…&$orderby=modifiedon%20desc`, plus follow-up `@odata.nextLink` URLs
  - `GET WhoAmI` → `{ UserId }`
  - `GET <prefix>_gridlayouts?$select=<prefix>_gridlayoutid,<prefix>_layout&$filter=<prefix>_name eq '<gridId>' and _ownerid_value eq <UserId>&$top=1`
  - `POST <prefix>_gridlayouts` with `{ <prefix>_name, <prefix>_layout }` → `204` plus header
    `OData-EntityId: …/<prefix>_gridlayouts(<guid>)`
  - `PATCH <prefix>_gridlayouts(<guid>)` with `{ <prefix>_layout }`
- **Layout table to create** (in the same solution):
  - display name **Grid Layout**, **Record ownership: User or team**
  - primary column **Name** (`<prefix>_name`)
  - column **Layout**: *Multiple lines of text*, max 100,000 (`<prefix>_layout`)
  - every security role that uses the grid needs **Create / Read / Write at User level** on it
- **Stored layout format:** the JSON of `pilar.estado()`.

## 8. Mock server (`mock/server.ts`)

- **Serves `dist/` under `/WebResources/…`**, so relative paths behave exactly as in Dataverse. `/` redirects to the built
  `index.html`.
- **Imitates the Web API under `/api/data/v9.2/`** with 250 ms latency:
  - `WhoAmI`
  - `accounts`: 240 deterministic sample rows with formatted-value annotations; supports `$select`, `$orderby`,
    `<field> eq <number>` filters, `Prefer: odata.maxpagesize` and `$skiptoken` paging with an absolute `@odata.nextLink`
  - `new_gridlayouts`: GET with the name/owner filter, POST (with `OData-EntityId`), PATCH; persisted to `.mock-store.json`
  - OData-style error bodies
- **Test switches:**
  - `POST /__mock/fail?status=403` (status `0` turns it off)
  - `POST /__mock/layouts-missing?on=1`
  - `POST /__mock/reset`
  - `GET /__mock/host`: a fake model-driven form that hosts the grid in an iframe and logs `parent.Xrm.Navigation.openForm`
    calls
- **`startMockServer({ port, latencyMs, persist })`** is exported for the tests.

## 9. Tests (`bun test`, 15 cases in Appendix A)

- **Patch** (3):
  - every patch applies
  - no Spanish UI strings remain in the code (comments and unused sample data excepted)
  - a missing anchor throws
- **Mapping** (2):
  - query shape: id first, lookups as `_x_value`, encoded filter and orderby
  - rows: formatted values except text/id, `—` for empty, and the status `estados` map
- **Layout** (4), against a fake Pilar that copies Pilar's ordering rules:
  - a full round trip reproduces the saved state exactly
  - a pinned + hidden column is pinned first
  - added and removed columns are tolerated
  - it never hides every column
- **Web API client vs mock** (6):
  - paging: 240 rows in 3 pages; `maxRows` truncation
  - formatted values and `$select`
  - 403 → a `DataverseError` with the message
  - the layout store creates once, then updates
  - a missing layout table is recognised
  - `whoAmI`

## 10. Acceptance checks (run these yourself)

1. `bun install && bun test` → 15 pass. `bun run typecheck` → no errors.
2. `bun run dev`, then open `http://localhost:5181`. Expect:
   - 240 records, the "Mock Web API" badge and English labels
   - formatted money, numbers and dates, and Active/Inactive chips
   - no console errors
3. Pin Account, hide Phone, move Owner to third, widen City and choose Compact. The header shows "Layout saved to
   Dataverse", and the mock holds exactly **one** layout record.
4. Reload. `pilar.estado()` equals the state from step 3 exactly; the header says "Layout restored from Dataverse"; the
   table appears with no animation.
5. Double-click a row → a toast: "Would open account … (mock mode)".
6. `POST /__mock/fail?status=403`, then reload → the error panel with the permission message and **Try again**.
7. Turn that off, `POST /__mock/layouts-missing?on=1`, reload → the grid loads, a toast says the table wasn't found,
   and changes save to "this browser".
8. Open `/__mock/host`. The grid fills the iframe edge to edge (no card), and double-clicking logs
   `Xrm.Navigation.openForm {"entityName":"account","entityId":"…"}`.

## 11. Deploy to Dataverse

1. Build with the publisher prefix: `bun scripts/build.ts --prefix <prefix>` (also set `layoutStore` in `config.ts` to
   `<prefix>_gridlayouts` / `<prefix>_gridlayoutid` / `<prefix>_name` / `<prefix>_layout`).
2. Create the Grid Layout table (section 7). It's optional; without it, layouts are per browser.
3. Solution → **New → More → Web resource**, once for each of the five files. The name after the prefix must match the
   build output exactly (`<prefix>_/pilargrid/index.html`, `…/pilar.js`, `…/app.js`, `…/pilar.css`, `…/grid.css`).
   Types: HTML → Webpage (HTML), JS → Script (JScript), CSS → Style Sheet (CSS).
4. **Publish all customizations.**
5. Smoke test by opening `https://<org>.crm.dynamics.com/WebResources/<prefix>_/pilargrid/index.html`. Expect the
   "Dataverse" badge and real record counts, and a layout change should survive a reload.
6. Surface it as a model-driven **app page** (Add page → Web resource), a form **HTML web resource** component, or a
   dashboard component.
7. Later config changes only need a rebuild and a re-upload of `app.js`.

## 12. Pitfalls already solved (don't reintroduce them)

- **Animation-frame stalls:** browsers pause `requestAnimationFrame` in background pages, and a web resource on a collapsed
  form tab counts. Never block the reveal on rAF alone; race it against a timeout.
- **Restore order matters:** pin before hiding, then reorder each group with in-group positions.
- **`alCambiar` fires during restore:** ignore it then, or you'll save half-restored states.
- **Duplicate layout records:** serialise saves, so two quick changes can't both POST.
- **Relative paths:** web resource names must mirror the folder layout, or `pilar.css` / `app.js` won't resolve.
- **No Google Fonts:** Pilar's demo loads them; this page doesn't (CSP). Use Segoe UI.
- **Lookups** are `_<name>_value` in both `$select` and the response; their label is the formatted-value annotation.
- **OData filters:** GUIDs go unquoted; strings go in single quotes with any `'` doubled.
- **`parent.Xrm`** works today, but Microsoft prefers passing `Xrm` via `getContentWindow`, so keep the `main.aspx` fallback.
- **Size limit:** every row is drawn. Keep `maxRows` in the hundreds, or add paging or a narrower filter.

## 13. Out of scope (possible next steps)

- Inline editing: a PATCH to `/<entitySet>(<id>)` with `If-Match: <etag>`.
- Server-side sorting, filtering and search.
- Virtualised rendering for thousands of rows.
- Packaging as a PCF dataset control, for canvas apps.
- A managed-solution export (it's a manual upload today).

---

## Appendix A: complete source

The project's own files are below, exactly as built and tested. Pilar itself (`vendor/moxen-pilar/*`) isn't included,
because of its licence; download it as described in section 3.

File tree:

```text
package.json
tsconfig.json
.gitignore
src/config.ts
src/mapping.ts
src/dataverse.ts
src/layout.ts
src/host.ts
src/app.ts
src/index.html
src/grid.css
scripts/patch-pilar.ts
scripts/build.ts
mock/data.ts
mock/server.ts
tests/patch.test.ts
tests/mapping.test.ts
tests/layout.test.ts
tests/api.test.ts
vendor/moxen-pilar/   (download: pilar.js, pilar.css, index.html, README.md, LICENCIA.txt)
```

### `package.json`

```json
{
  "name": "pilar-dataverse",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "bun scripts/build.ts",
    "dev": "bun scripts/build.ts && bun mock/server.ts",
    "test": "bun test",
    "typecheck": "bunx tsc --noEmit -p .",
    "brief": "bun scripts/make-brief.ts"
  },
  "devDependencies": {
    "@types/bun": "^1.4.4",
    "typescript": "^7.0.2"
  }
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["bun"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "skipLibCheck": true,
    "allowImportingTsExtensions": true
  },
  "include": ["src", "mock", "scripts", "tests"]
}
```

### `.gitignore`

```text
node_modules/
dist/
.mock-store.json
```

### `src/config.ts`

```ts
// What the grid shows. Edit this file, then `bun run build`.
// The default points at the standard Account table, which every Dataverse environment has.

export type ColumnKind =
  | "text"    // plain text
  | "id"      // record number / code, monospace blue
  | "number"  // right-aligned, uses Dataverse's formatted value ("1,250")
  | "money"   // right-aligned, formatted with currency ("$1,250.00")
  | "date"    // monospace, formatted in the user's format and time zone
  | "choice"  // option set label
  | "lookup"  // name of the related record (queried as _field_value)
  | "status"; // option set shown as a coloured chip (see statusTones)

export type Tone = "filled" | "outline" | "muted" | "dark";

export interface GridColumn {
  field: string;   // logical name (for lookups, the lookup's logical name, e.g. "primarycontactid")
  label: string;
  kind: ColumnKind;
  width?: number;  // starting width in px
}

export type LayoutStoreConfig =
  | { kind: "dataverse"; entitySet: string; idField: string; keyField: string; layoutField: string }
  | { kind: "local" };

export interface GridConfig {
  gridId: string;           // key the saved layout is stored under; change it if you change the columns a lot
  title: string;
  subtitle?: string;
  entityName: string;       // logical name, used to open a record
  entitySet: string;        // Web API entity set name
  idField: string;          // primary key column
  columns: GridColumn[];
  filter?: string;          // OData $filter
  orderBy?: string;         // OData $orderby
  pageSize: number;         // rows per Web API page
  maxRows: number;          // stop after this many rows (every row is drawn, so keep it in the hundreds)
  statusTones: Record<string, Tone>; // option value -> chip style; unlisted values are "muted"
  layoutStore: LayoutStoreConfig;
}

export const config: GridConfig = {
  gridId: "accounts-grid",
  title: "Accounts",
  subtitle: "Dataverse",
  entityName: "account",
  entitySet: "accounts",
  idField: "accountid",
  columns: [
    { field: "name", label: "Account", kind: "text", width: 220 },
    { field: "accountnumber", label: "Number", kind: "id", width: 120 },
    { field: "statuscode", label: "Status", kind: "status", width: 120 },
    { field: "primarycontactid", label: "Primary contact", kind: "lookup", width: 170 },
    { field: "address1_city", label: "City", kind: "text", width: 130 },
    { field: "telephone1", label: "Phone", kind: "text", width: 140 },
    { field: "revenue", label: "Annual revenue", kind: "money", width: 150 },
    { field: "numberofemployees", label: "Employees", kind: "number", width: 120 },
    { field: "ownerid", label: "Owner", kind: "lookup", width: 150 },
    { field: "modifiedon", label: "Modified", kind: "date", width: 160 },
  ],
  orderBy: "modifiedon desc",
  pageSize: 100,
  maxRows: 500,
  statusTones: { "1": "filled", "2": "muted" }, // Account: 1 = Active, 2 = Inactive
  // Per-user saved layouts. Create this table (see README) or switch to { kind: "local" }.
  // If the table doesn't exist the app falls back to this browser's storage and says so.
  layoutStore: {
    kind: "dataverse",
    entitySet: "new_gridlayouts",
    idField: "new_gridlayoutid",
    keyField: "new_name",
    layoutField: "new_layout",
  },
};
```

### `src/mapping.ts`

```ts
// Dataverse records -> Pilar's input (columnas, filas, estados), plus the Web API query that fetches them.
import type { ColumnKind, GridConfig, Tone } from "./config";

const FORMATTED = "@OData.Community.Display.V1.FormattedValue";
const EMPTY = "—";

export type PilarColumn = { id: string; nombre: string; tipo: "id" | "estado" | "texto" | "num" | "mono"; ancho: number };
export type PilarStatus = { texto: string; tono: Tone };
export type DataverseRecord = Record<string, unknown>;

const TIPO: Record<ColumnKind, PilarColumn["tipo"]> = {
  text: "texto", lookup: "texto", choice: "texto", id: "id", number: "num", money: "num", date: "mono", status: "estado",
};

/** The JSON property a column's value comes back under. Lookups come back as _field_value. */
export const propertyOf = (field: string, kind: ColumnKind) => (kind === "lookup" ? `_${field}_value` : field);

export function buildQuery(cfg: GridConfig): string {
  const select = [cfg.idField, ...cfg.columns.map((c) => propertyOf(c.field, c.kind))];
  const parts = [`$select=${[...new Set(select)].join(",")}`];
  if (cfg.filter) parts.push(`$filter=${encodeURIComponent(cfg.filter)}`);
  if (cfg.orderBy) parts.push(`$orderby=${encodeURIComponent(cfg.orderBy)}`);
  return `${cfg.entitySet}?${parts.join("&")}`;
}

function cellText(record: DataverseRecord, prop: string, kind: ColumnKind): string {
  const raw = record[prop];
  if (raw === null || raw === undefined || raw === "") return EMPTY;
  // Formatted values carry the user's locale, currency and time zone; prefer them for everything but text.
  const formatted = record[prop + FORMATTED];
  if (kind !== "text" && kind !== "id" && typeof formatted === "string" && formatted !== "") return formatted;
  return String(raw);
}

export function toPilar(cfg: GridConfig, records: DataverseRecord[]) {
  const columnas: PilarColumn[] = cfg.columns.map((c) => ({
    id: c.field, nombre: c.label, tipo: TIPO[c.kind], ancho: c.width ?? 140,
  }));
  const estados: Record<string, PilarStatus> = {};
  const ids: string[] = [];
  const filas = records.map((r) => {
    ids.push(String(r[cfg.idField] ?? ""));
    return cfg.columns.map((c) => {
      const prop = propertyOf(c.field, c.kind);
      if (c.kind !== "status") return cellText(r, prop, c.kind);
      const raw = r[prop];
      if (raw === null || raw === undefined) return EMPTY;
      const key = String(raw);
      if (!estados[key]) {
        const label = r[prop + FORMATTED];
        estados[key] = { texto: typeof label === "string" && label ? label : key, tono: cfg.statusTones[key] ?? "muted" };
      }
      return key;
    });
  });
  return { columnas, filas, estados, ids };
}
```

### `src/dataverse.ts`

```ts
// Minimal Dataverse Web API client. Inside Dataverse the page is a web resource on the org's own domain,
// so relative /api/data calls ride on the signed-in user's session: no tokens, no CORS, their security roles apply.

export class DataverseError extends Error {
  constructor(message: string, readonly status: number, readonly code?: string) {
    super(message);
  }
}

const HEADERS: Record<string, string> = {
  Accept: "application/json",
  "OData-MaxVersion": "4.0",
  "OData-Version": "4.0",
};

export class Dataverse {
  /** base: the Web API root. Relative by default; tests pass an absolute URL. */
  constructor(readonly base = "/api/data/v9.2/") {}

  private url(path: string) {
    return /^https?:\/\//.test(path) ? path : this.base + path;
  }

  async request(path: string, init: RequestInit & { prefer?: string[] } = {}): Promise<Response> {
    const headers: Record<string, string> = { ...HEADERS, ...(init.headers as Record<string, string>) };
    if (init.body) headers["Content-Type"] = "application/json; charset=utf-8";
    if (init.prefer?.length) headers.Prefer = init.prefer.join(",");
    const res = await fetch(this.url(path), { ...init, headers, credentials: "same-origin" });
    if (!res.ok) {
      let message = `${res.status} ${res.statusText}`, code: string | undefined;
      try {
        const body = (await res.json()) as { error?: { code?: string; message?: string } };
        if (body.error?.message) message = body.error.message;
        code = body.error?.code;
      } catch { /* not JSON */ }
      throw new DataverseError(message, res.status, code);
    }
    return res;
  }

  async json<T>(path: string, init?: RequestInit & { prefer?: string[] }): Promise<T> {
    return (await this.request(path, init)).json() as Promise<T>;
  }

  /** Follows @odata.nextLink until maxRows. Asks for formatted values (labels, currency, local dates). */
  async retrieveAll<T = Record<string, unknown>>(query: string, opts: { pageSize: number; maxRows: number }) {
    const prefer = [`odata.maxpagesize=${opts.pageSize}`, 'odata.include-annotations="OData.Community.Display.V1.FormattedValue"'];
    const records: T[] = [];
    let next: string | undefined = query, pages = 0;
    while (next && records.length < opts.maxRows) {
      const page: { value: T[]; "@odata.nextLink"?: string } = await this.json(next, { prefer });
      records.push(...page.value);
      next = page["@odata.nextLink"];
      pages++;
    }
    return { records: records.slice(0, opts.maxRows), truncated: Boolean(next) || records.length > opts.maxRows, pages };
  }

  async whoAmI(): Promise<string> {
    return (await this.json<{ UserId: string }>("WhoAmI")).UserId;
  }

  /** POST a record; returns its id from the OData-EntityId header. */
  async create(entitySet: string, data: Record<string, unknown>): Promise<string> {
    const res = await this.request(entitySet, { method: "POST", body: JSON.stringify(data) });
    const id = res.headers.get("OData-EntityId")?.match(/\(([0-9a-f-]{36})\)/i)?.[1];
    if (!id) throw new DataverseError("Create succeeded but returned no record id", res.status);
    return id;
  }

  async update(entitySet: string, id: string, data: Record<string, unknown>): Promise<void> {
    await this.request(`${entitySet}(${id})`, { method: "PATCH", body: JSON.stringify(data), headers: { "If-Match": "*" } });
  }
}

/** True when an error means "that table or column doesn't exist" rather than a real failure. */
export const isMissingResource = (e: unknown) =>
  e instanceof DataverseError && (e.status === 404 || /0x80060888|Resource not found|Could not find a property/i.test(`${e.code} ${e.message}`));
```

### `src/layout.ts`

```ts
// Saving and restoring each user's column layout (order, widths, pinned, hidden, density).
import type { LayoutStoreConfig } from "./config";
import { Dataverse } from "./dataverse";

export type PilarState = {
  orden: string[];
  anchos: Record<string, number>;
  fijas: string[];
  ocultas: string[];
  densidad: string;
};

export type PilarApi = {
  mueve(id: string, pos: number): void;
  fija(id: string, v: boolean): void;
  oculta(id: string, v: boolean): void;
  ancho(id: string, px: number): void;
  densidad(d: string): void;
  estado(): PilarState;
};

export type RestoreOp =
  | { op: "fija"; id: string }
  | { op: "mueve"; id: string; pos: number }
  | { op: "ancho"; id: string; px: number }
  | { op: "oculta"; id: string }
  | { op: "densidad"; d: string };

const DENSITIES = ["comoda", "normal", "compacta"];

/**
 * Steps that take a fresh grid (`defaults`) to a saved layout. Columns the saved layout doesn't know
 * (added to the config since) go to the end; saved columns no longer in the config are ignored.
 * Order matters: pin before hiding (Pilar won't pin a hidden column), and Pilar's `mueve` position
 * counts within the column's own group (pinned or not), so each group is placed left to right.
 */
export function planRestore(saved: PilarState, defaults: PilarState): RestoreOp[] {
  const known = new Set(defaults.orden);
  const order = [...saved.orden.filter((id) => known.has(id)), ...defaults.orden.filter((id) => !saved.orden.includes(id))];
  const pinned = order.filter((id) => saved.fijas.includes(id));
  const ops: RestoreOp[] = pinned.map((id) => ({ op: "fija", id }));
  for (const group of [pinned, order.filter((id) => !pinned.includes(id))]) {
    group.forEach((id, pos) => ops.push({ op: "mueve", id, pos }));
  }
  for (const id of order) {
    const px = saved.anchos?.[id];
    if (typeof px === "number" && Math.round(px) !== Math.round(defaults.anchos[id] ?? -1)) ops.push({ op: "ancho", id, px: Math.round(px) });
  }
  const hidden = order.filter((id) => saved.ocultas.includes(id));
  if (hidden.length >= order.length) hidden.pop(); // always leave one column visible
  hidden.forEach((id) => ops.push({ op: "oculta", id }));
  if (saved.densidad !== defaults.densidad && DENSITIES.includes(saved.densidad)) ops.push({ op: "densidad", d: saved.densidad });
  return ops;
}

export function applyRestore(pilar: PilarApi, ops: RestoreOp[]) {
  for (const o of ops) {
    if (o.op === "fija") pilar.fija(o.id, true);
    else if (o.op === "mueve") pilar.mueve(o.id, o.pos);
    else if (o.op === "ancho") pilar.ancho(o.id, o.px);
    else if (o.op === "oculta") pilar.oculta(o.id, true);
    else pilar.densidad(o.d);
  }
}

export interface LayoutStore {
  readonly where: string; // shown to the user: where layouts are being saved
  load(): Promise<PilarState | null>;
  save(state: PilarState): Promise<void>;
}

export class LocalLayoutStore implements LayoutStore {
  readonly where = "this browser";
  constructor(private key: string) {}
  async load() {
    try { const s = localStorage.getItem(this.key); return s ? (JSON.parse(s) as PilarState) : null; } catch { return null; }
  }
  async save(state: PilarState) {
    try { localStorage.setItem(this.key, JSON.stringify(state)); } catch { /* storage blocked: nothing to do */ }
  }
}

/** One record per user per grid, in a user-owned table, so each person only sees their own layouts. */
export class DataverseLayoutStore implements LayoutStore {
  readonly where = "Dataverse";
  private recordId: string | null = null;
  constructor(private api: Dataverse, private cfg: Extract<LayoutStoreConfig, { kind: "dataverse" }>, private key: string, private userId: string) {}

  async load(): Promise<PilarState | null> {
    const { entitySet, idField, keyField, layoutField } = this.cfg;
    const filter = encodeURIComponent(`${keyField} eq '${this.key.replace(/'/g, "''")}' and _ownerid_value eq ${this.userId}`);
    const res = await this.api.json<{ value: Record<string, string>[] }>(
      `${entitySet}?$select=${idField},${layoutField}&$filter=${filter}&$top=1`);
    const row = res.value[0];
    if (!row) return null;
    this.recordId = row[idField] ?? null;
    try { return JSON.parse(row[layoutField] ?? "null") as PilarState | null; } catch { return null; }
  }

  async save(state: PilarState) {
    const { entitySet, keyField, layoutField } = this.cfg;
    const body = { [layoutField]: JSON.stringify(state) };
    if (this.recordId) await this.api.update(entitySet, this.recordId, body);
    else this.recordId = await this.api.create(entitySet, { ...body, [keyField]: this.key });
  }
}
```

### `src/host.ts`

```ts
// Where the page is running: inside a model-driven app (Xrm available from the parent frame),
// as a standalone web resource tab, or on the local mock server.

type Xrm = { Navigation: { openForm(o: { entityName: string; entityId: string }): Promise<unknown> } };

export function getXrm(): Xrm | null {
  const w = window as unknown as { Xrm?: Xrm; parent?: { Xrm?: Xrm } };
  if (w.Xrm?.Navigation) return w.Xrm;
  try { if (w.parent?.Xrm?.Navigation) return w.parent.Xrm; } catch { /* cross-origin parent */ }
  return null;
}

export const isLocalDev = () => ["localhost", "127.0.0.1", "[::1]"].includes(location.hostname);
export const isEmbedded = () => { try { return window.self !== window.top; } catch { return true; } };

/** Opens a record the native way when possible; otherwise in a new tab via main.aspx. */
export function openRecord(entityName: string, id: string, notify: (msg: string) => void) {
  const xrm = getXrm();
  if (xrm) { xrm.Navigation.openForm({ entityName, entityId: id }); return; }
  if (isLocalDev()) { notify(`Would open ${entityName} ${id.slice(0, 8)}… (mock mode)`); return; }
  window.open(`/main.aspx?etn=${encodeURIComponent(entityName)}&id=${encodeURIComponent(id)}&pagetype=entityrecord`, "_blank", "noopener");
}
```

### `src/app.ts`

```ts
// Entry point: load records from Dataverse, hand them to Pilar, restore and save the user's layout,
// open a record on double-click.
import { config } from "./config";
import { Dataverse, DataverseError, isMissingResource } from "./dataverse";
import { buildQuery, toPilar } from "./mapping";
import {
  applyRestore, DataverseLayoutStore, LocalLayoutStore, planRestore,
  type LayoutStore, type PilarApi, type PilarState,
} from "./layout";
import { isEmbedded, isLocalDev, openRecord } from "./host";

declare const Pilar: (root: Element, options: object) => PilarApi;

const root = document.querySelector<HTMLElement>("[data-pilar]")!;
const $ = <T extends HTMLElement = HTMLElement>(sel: string) => root.querySelector<T>(sel)!;
const api = new Dataverse();

function setStatus(sel: string, text: string, tone?: "ok" | "warn" | "busy") {
  const el = $(sel);
  el.textContent = text;
  if (tone) el.dataset.tone = tone; else delete el.dataset.tone;
}

let toastTimer: ReturnType<typeof setTimeout> | undefined;
function toast(message: string) {
  const el = $("[data-toast]");
  el.textContent = message;
  el.hidden = false;
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => (el.hidden = true), 3200);
}

function showPanel(kind: "loading" | "error" | "empty" | null, message = "") {
  const panel = $("[data-panel]");
  panel.hidden = kind === null;
  panel.dataset.kind = kind ?? "";
  $("[data-panel-text]").textContent = message;
  $<HTMLButtonElement>("[data-retry]").hidden = kind !== "error";
}

function explain(e: unknown): string {
  if (e instanceof DataverseError) {
    if (e.status === 401 || e.status === 403) return `You don't have permission to read ${config.title.toLowerCase()} (${e.status}). ${e.message}`;
    return `Dataverse returned an error (${e.status}): ${e.message}`;
  }
  return `Couldn't load the data: ${e instanceof Error ? e.message : String(e)}`;
}

/** Picks where layouts are saved. Falls back to this browser if the layout table isn't there. */
async function openLayoutStore(): Promise<{ store: LayoutStore; saved: PilarState | null; note?: string }> {
  const key = `pilar-layout:${config.gridId}`;
  if (config.layoutStore.kind === "local") return { store: new LocalLayoutStore(key), saved: await new LocalLayoutStore(key).load() };
  try {
    const store = new DataverseLayoutStore(api, config.layoutStore, config.gridId, await api.whoAmI());
    return { store, saved: await store.load() };
  } catch (e) {
    const local = new LocalLayoutStore(key);
    const note = isMissingResource(e)
      ? `Layout table "${config.layoutStore.entitySet}" not found, so layouts are saved in this browser only.`
      : `Couldn't reach the layout table (${e instanceof Error ? e.message : e}); saving in this browser for now.`;
    return { store: local, saved: await local.load(), note };
  }
}

/** Saves at most one layout at a time; the newest pending state wins. */
function makeSaver(store: LayoutStore) {
  let pending: PilarState | null = null, busy = false, timer: ReturnType<typeof setTimeout> | undefined;
  const flush = async () => {
    if (busy || !pending) return;
    busy = true;
    const state = pending; pending = null;
    setStatus("[data-saved]", "Saving layout…", "busy");
    try {
      await store.save(state);
      setStatus("[data-saved]", `Layout saved to ${store.where}`, "ok");
    } catch (e) {
      setStatus("[data-saved]", "Layout not saved", "warn");
      toast(`Couldn't save your layout: ${e instanceof Error ? e.message : e}`);
    } finally {
      busy = false;
      if (pending) flush();
    }
  };
  return (state: PilarState) => { pending = state; clearTimeout(timer); timer = setTimeout(flush, 700); };
}

async function main() {
  document.documentElement.toggleAttribute("data-embebido", isEmbedded());
  document.title = config.title;
  $("[data-title]").textContent = config.title;
  $("[data-subtitle]").textContent = config.subtitle ?? "";
  setStatus("[data-mode]", isLocalDev() ? "Mock Web API" : "Dataverse", isLocalDev() ? "warn" : "ok");
  showPanel("loading", `Loading ${config.title.toLowerCase()}…`);

  let data;
  try {
    const [{ records, truncated }, layout] = await Promise.all([
      api.retrieveAll(buildQuery(config), { pageSize: config.pageSize, maxRows: config.maxRows }),
      openLayoutStore(),
    ]);
    data = { ...toPilar(config, records), truncated, layout };
  } catch (e) {
    showPanel("error", explain(e));
    return;
  }

  const { columnas, filas, estados, ids, truncated, layout } = data;
  setStatus("[data-count]", `${filas.length.toLocaleString()}${truncated ? "+" : ""} records${truncated ? ` (first ${config.maxRows})` : ""}`);
  if (layout.note) toast(layout.note);
  setStatus("[data-saved]", layout.saved ? `Layout restored from ${layout.store.where}` : `Layout saves to ${layout.store.where}`);
  if (!filas.length) { showPanel("empty", `No ${config.title.toLowerCase()} to show.`); return; }

  const save = makeSaver(layout.store);
  let restoring = true;
  const pilar = Pilar(root, {
    columnas, filas, estados,
    alCambiar: (state: PilarState) => { if (!restoring) save(state); },
  });
  (window as unknown as { pilar: PilarApi }).pilar = pilar;

  // Replay the saved layout with animations off (data-pilar-quieto) while the table is hidden. Pilar's
  // spring loop runs on animation frames, which browsers pause for background pages (and a web resource on a
  // collapsed form tab), so: reveal after two frames or 150 ms, whichever comes first, and keep animations
  // off until the page has actually painted two frames while visible.
  if (layout.saved) {
    const html = document.documentElement;
    const twoFrames = () => new Promise<void>((r) => requestAnimationFrame(() => requestAnimationFrame(() => r())));
    const quietOff = () => twoFrames().then(() => html.removeAttribute("data-pilar-quieto"));
    html.setAttribute("data-pilar-quieto", "");
    root.dataset.restaurando = "";
    applyRestore(pilar, planRestore(layout.saved, pilar.estado()));
    await Promise.race([twoFrames(), new Promise((r) => setTimeout(r, 150))]);
    delete root.dataset.restaurando;
    if (document.hidden) document.addEventListener("visibilitychange", quietOff, { once: true });
    else quietOff();
  }
  restoring = false;
  showPanel(null);

  // Rows carry their index in --i (set by Pilar); map it back to the record id.
  const rowId = (target: EventTarget | null) => {
    const row = (target as HTMLElement | null)?.closest<HTMLElement>(".pi-fila:not(.pi-fila--cab)");
    const i = row ? Number(row.style.getPropertyValue("--i")) : NaN;
    return Number.isInteger(i) ? ids[i] : undefined;
  };
  const table = $("[data-tabla]");
  table.addEventListener("dblclick", (e) => { const id = rowId(e.target); if (id) openRecord(config.entityName, id, toast); });
  table.addEventListener("pointermove", (e) => { table.title = rowId(e.target) ? "Double-click to open" : ""; });
}

$("[data-retry]").addEventListener("click", () => location.reload());
main();
```

### `src/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Grid</title>
<link rel="stylesheet" href="pilar.css">
<link rel="stylesheet" href="grid.css">
</head>
<body>
  <!-- Pilar (pilar.js, © Moxen) builds the table into [data-tabla]; app.js loads the rows from the Dataverse Web API.
       The data-* hooks below are the ones Pilar looks for; keep them if you restyle this page. -->
  <main class="pi-escena" data-pilar>
    <div class="pi-fondo" aria-hidden="true"></div>

    <section class="pi-app" aria-labelledby="pi-titulo">
      <header class="pi-cab">
        <div class="pi-titulos">
          <p class="pi-etq"><span data-subtitle></span></p>
          <h1 id="pi-titulo" data-title>Grid</h1>
          <p class="pv-meta">
            <span class="pv-badge" data-mode></span>
            <span data-count></span>
            <span class="pv-saved" data-saved></span>
          </p>
        </div>

        <div class="pi-barra" role="toolbar" aria-label="Table settings">
          <div class="pi-menu-env">
            <button class="pi-btn pi-btn-col" type="button" data-columnas aria-haspopup="menu" aria-expanded="false" aria-controls="pi-menu">
              <svg viewBox="0 0 24 24" aria-hidden="true"><rect x="4" y="5.5" width="16" height="13" rx="3"/><path d="M9.4 5.5v13M14.6 5.5v13"/></svg>
              <span>Columns</span>
              <b class="pi-cuenta" data-cuenta aria-hidden="true"></b>
              <svg class="pi-caret" viewBox="0 0 24 24" aria-hidden="true"><path d="M7 10l5 5 5-5"/></svg>
            </button>
            <div class="pi-menu" id="pi-menu" data-menu role="menu" aria-label="Visible columns" hidden>
              <ul class="pi-menu-lista" data-menu-lista></ul>
              <div class="pi-menu-pie">
                <span data-menu-cuenta></span>
                <button class="pi-menu-todas" type="button" data-todas>Show all</button>
              </div>
            </div>
          </div>

          <div class="pi-seg" role="radiogroup" aria-label="Row density" data-densidad>
            <i class="pi-seg-pulgar" aria-hidden="true"></i>
            <button type="button" role="radio" aria-checked="false" data-d="comoda">Comfortable</button>
            <button type="button" role="radio" aria-checked="true" data-d="normal">Normal</button>
            <button type="button" role="radio" aria-checked="false" data-d="compacta">Compact</button>
          </div>

          <button class="pi-btn pi-rest" type="button" data-restablece data-e="reposo" disabled>
            <span class="pi-rest-a"><svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5.5 11.2a6.6 6.6 0 1 1 1.4 5.3M5.5 6.4v4.8h4.8"/></svg><span>Reset</span><i class="pi-punto" aria-hidden="true"></i></span>
            <span class="pi-rest-b" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M6.6 12.6l3.8 3.8 7.2-8.2" pathLength="1"/></svg><span>Reset done</span></span>
          </button>
        </div>
      </header>

      <div class="pi-cuerpo">
        <div class="pi-scroll" data-scroll role="region" aria-label="Records table, scrolls horizontally" tabindex="0">
          <div class="pi-tabla" data-tabla role="table" aria-labelledby="pi-titulo"></div>
        </div>
        <div class="pv-panel" data-panel role="status">
          <div class="pv-spinner" aria-hidden="true"></div>
          <p data-panel-text>Loading…</p>
          <button class="pi-btn" type="button" data-retry hidden>Try again</button>
        </div>
      </div>

      <footer class="pi-pie">
        <p class="pi-estado" data-estado></p>
        <p class="pi-teclas"><span>Focus a header:</span> <kbd>←</kbd><kbd>→</kbd> move focus <i>·</i> <kbd>Alt</kbd>+<kbd>←</kbd><kbd>→</kbd> reorder <i>·</i> <kbd>Shift</kbd>+<kbd>←</kbd><kbd>→</kbd> width <i>·</i> <kbd>P</kbd> pin <i>·</i> <kbd>H</kbd> hide <i>·</i> <kbd>Enter</kbd> fit <i>·</i> double-click a row to open it</p>
      </footer>
    </section>

    <p class="pv-toast" data-toast role="status" aria-live="polite" hidden></p>
    <p class="pi-vivo" role="status" aria-live="polite" data-vivo></p>
  </main>

  <script src="pilar.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

### `src/grid.css`

```css
/* Overrides on top of Moxen's pilar.css: Dataverse fonts, generic status tones, embedded layout,
   header status line, loading/error panel and toast. */

:root {
  /* Segoe UI is what model-driven apps use; no web fonts are loaded (tenant CSPs often block them). */
  --pi-sans: "Segoe UI", system-ui, -apple-system, sans-serif;
  --pi-mono: "Cascadia Mono", Consolas, ui-monospace, "SF Mono", Menlo, monospace;
}

/* Status chip tones (config.statusTones), mirroring Pilar's four original looks. */
.pi-chip[data-e="filled"] { background: var(--pi-azul-suave); color: var(--pi-azul-tx); }
.pi-chip[data-e="outline"] { box-shadow: 0 0 0 1px rgba(var(--pi-azul-rgb), .5) inset; color: var(--pi-azul-tx); }
.pi-chip[data-e="outline"] i { background: none; box-shadow: 0 0 0 1.6px currentColor inset; }
.pi-chip[data-e="muted"] { background: var(--pi-gris); color: var(--pi-suave); }
.pi-chip[data-e="dark"] { background: var(--pi-grafito); color: var(--pi-blanco); }

/* Inside a form, dashboard or app page: fill the frame, no card or backdrop. */
[data-embebido] .pi-escena { padding: 0; min-height: 100vh; display: block; }
[data-embebido] .pi-fondo { display: none; }
[data-embebido] .pi-app { width: 100%; height: 100vh; border-radius: 0; border: 0; box-shadow: none; }
[data-embebido] .pi-cab { padding: 14px 20px 12px; }
[data-embebido] .pi-pie { padding-left: 20px; padding-right: 20px; }

/* Header status line under the title. */
.pv-meta { display: flex; flex-wrap: wrap; align-items: center; gap: 6px 12px; margin: 6px 0 0; font-size: 12.5px; color: var(--pi-suave); }
.pv-meta:empty { display: none; }
.pv-badge { padding: 2px 8px; border-radius: 999px; background: var(--pi-gris); font-weight: 600; font-size: 11.5px; letter-spacing: .02em; }
.pv-badge[data-tone="ok"] { background: var(--pi-azul-suave); color: var(--pi-azul-tx); }
.pv-badge[data-tone="warn"] { background: #fff2d6; color: #8a5a00; }
.pv-saved[data-tone="busy"] { color: var(--pi-azul-tx); }
.pv-saved[data-tone="warn"] { color: #b42318; }

/* Loading / error / empty panel over the table body. */
.pv-panel { position: absolute; inset: 0; z-index: 8; display: grid; place-content: center; justify-items: center; gap: 12px;
  padding: 24px; text-align: center; background: var(--pi-blanco); color: var(--pi-suave); font-size: 14px; }
.pv-panel[hidden] { display: none; }
.pv-panel p { margin: 0; max-width: 520px; line-height: 1.45; }
.pv-panel[data-kind="error"] p { color: #b42318; }
.pv-spinner { width: 26px; height: 26px; border-radius: 50%; border: 2.5px solid var(--pi-gris-2); border-top-color: var(--pi-azul, #2f6fed);
  animation: pv-giro .8s linear infinite; }
.pv-panel:not([data-kind="loading"]) .pv-spinner { display: none; }
@keyframes pv-giro { to { transform: rotate(360deg); } }

/* The table stays hidden while a saved layout is replayed. */
[data-restaurando] .pi-scroll { visibility: hidden; }

.pi-fila:not(.pi-fila--cab) { cursor: default; }

.pv-toast { position: fixed; left: 50%; bottom: 22px; z-index: 20; transform: translateX(-50%); max-width: min(560px, calc(100vw - 32px));
  margin: 0; padding: 10px 16px; border-radius: 12px; background: var(--pi-grafito); color: var(--pi-blanco); font-size: 13.5px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, .18); }
.pv-toast[hidden] { display: none; }

@media (prefers-reduced-motion: reduce) { .pv-spinner { animation-duration: 2.4s; } }
```

### `scripts/patch-pilar.ts`

```ts
// Turns Moxen's original pilar.js (vendor/moxen-pilar, kept untouched) into the copy this app ships:
//   1. status chips read an `estados` option ({ value: { texto, tono } }) instead of a hard-coded Spanish list
//   2. every user-facing and screen-reader string is translated to English
//   3. <html data-pilar-quieto> switches animations off (used while restoring a saved layout)
// Each replacement must match the expected number of times, so a new Moxen version fails loudly
// instead of silently half-patching.
type Patch = { from: string; to: string; count?: number };

export const PATCHES: Patch[] = [
  { from: `const quieto = () => mq.matches;`, to: `const quieto = () => mq.matches || document.documentElement.hasAttribute('data-pilar-quieto');` },
  {
    from: `if (c.tipo === 'estado') return '<span class="pi-chip" data-e="' + esc(v) + '"><i></i>' + esc(ESTADOS[v] || v) + '</span>';`,
    to: `if (c.tipo === 'estado') { const e = (op.estados || ESTADOS)[v]; const obj = e && typeof e === 'object'; return '<span class="pi-chip" data-e="' + esc(obj ? e.tono : v) + '"><i></i>' + esc(obj ? e.texto : (e || v)) + '</span>'; }`,
  },
  { from: `{ comoda: 'cómodas', normal: 'normales', compacta: 'compactas' }`, to: `{ comoda: 'comfortable', normal: 'normal', compacta: 'compact' }` },
  { from: `(c.fija ? ', fijada a la izquierda' : '')`, to: `(c.fija ? ', pinned left' : '')` },
  { from: `c.fija ? 'Soltar' : 'Fijar a la izquierda'`, to: `c.fija ? 'Unpin' : 'Pin left'` },
  { from: `vis + ' de ' + cols.length + ' visibles'`, to: `vis + ' of ' + cols.length + ' visible'` },
  {
    from: `'<b>' + vis + '</b> de ' + cols.length + ' columnas · ' + (fijas ? '<b>' + fijas + '</b> ' + (fijas === 1 ? 'fija' : 'fijas') : 'ninguna fija') + ' · filas ' + DENSIDADES[densidad]`,
    to: `'<b>' + vis + '</b> of ' + cols.length + ' columns · ' + (fijas ? '<b>' + fijas + '</b> pinned' : 'none pinned') + ' · ' + DENSIDADES[densidad] + ' rows'`,
  },
  { from: `Math.round(c.w) + ' px' : 'oculta'`, to: `Math.round(c.w) + ' px' : 'hidden'` },
  { from: `': ajustada al contenido, '`, to: `': fitted to content, '` },
  { from: `' píxeles'`, to: `' pixels'`, count: 3 },
  { from: `' · mín.' : tope === 'max' ? ' · máx.'`, to: `' · min' : tope === 'max' ? ' · max'` },
  { from: `' levantada. Mueve el cursor para cambiar su sitio.'`, to: `' lifted. Move the pointer to choose its new position.'` },
  { from: `': posición '`, to: `': position '`, count: 2 },
  { from: `' en la posición '`, to: `' moved to position '` },
  { from: `' de ' + visibles().length`, to: `' of ' + visibles().length`, count: 3 },
  { from: `'Movimiento cancelado'`, to: `'Move cancelled'` },
  { from: `(v ? ': fijada a la izquierda' : ': suelta')`, to: `(v ? ': pinned left' : ': unpinned')` },
  { from: `(v ? ': oculta' : ': visible')`, to: `(v ? ': hidden' : ': visible')` },
  { from: `': ya está en el extremo' + (c.fija ? ' de lo fijo' : '')`, to: `': already at the end' + (c.fija ? ' of the pinned group' : '')` },
  { from: `'Tiene que quedar al menos una columna visible'`, to: `'At least one column must stay visible'`, count: 2 },
  { from: `anuncia('Filas ' + DENSIDADES[d])`, to: `anuncia('Rows: ' + DENSIDADES[d])` },
  { from: `'Tabla restablecida: cada columna en su sitio'`, to: `'Table reset: every column back in place'` },
];

export function patchPilar(source: string): string {
  let out = source;
  for (const p of PATCHES) {
    const found = out.split(p.from).length - 1;
    const want = p.count ?? 1;
    if (found !== want) throw new Error(`patch-pilar: expected ${want} match(es), found ${found}: ${p.from.slice(0, 80)}`);
    out = out.split(p.from).join(p.to);
  }
  return `/* Patched by scripts/patch-pilar.ts: English strings, configurable status chips, quiet mode. Original: vendor/moxen-pilar/pilar.js (© Moxen, see LICENCIA.txt). */\n` + out;
}
```

### `scripts/build.ts`

```ts
// Builds the web resources into dist/<prefix>_/<folder>/, named exactly as they must be in Dataverse
// so the page's relative links (pilar.css, app.js, …) keep working after upload.
//   bun scripts/build.ts [--prefix new] [--folder pilargrid]
import { mkdir, rm } from "node:fs/promises";
import { join } from "node:path";
import { patchPilar } from "./patch-pilar";

const ROOT = join(import.meta.dir, "..");
const argv = process.argv.slice(2);
const arg = (name: string, fallback: string) => {
  const i = argv.indexOf(`--${name}`);
  return i >= 0 && argv[i + 1] ? argv[i + 1]! : fallback;
};
const prefix = arg("prefix", process.env.WR_PREFIX ?? "new");
const folder = arg("folder", "pilargrid");
if (!/^[a-z][a-z0-9]{1,7}$/.test(prefix)) throw new Error(`--prefix should be your publisher prefix (2-8 lowercase letters/digits), got "${prefix}"`);

const base = `${prefix}_/${folder}`;
const out = join(ROOT, "dist", base);
await rm(join(ROOT, "dist"), { recursive: true, force: true });
await mkdir(out, { recursive: true });

await Bun.write(join(out, "pilar.js"), patchPilar(await Bun.file(join(ROOT, "vendor/moxen-pilar/pilar.js")).text()));
await Bun.write(join(out, "pilar.css"), Bun.file(join(ROOT, "vendor/moxen-pilar/pilar.css")));
await Bun.write(join(out, "grid.css"), Bun.file(join(ROOT, "src/grid.css")));
await Bun.write(join(out, "index.html"), Bun.file(join(ROOT, "src/index.html")));

const bundle = await Bun.build({ entrypoints: [join(ROOT, "src/app.ts")], target: "browser", format: "iife", minify: false });
if (!bundle.success) { for (const log of bundle.logs) console.error(log); process.exit(1); }
await Bun.write(join(out, "app.js"), `/* Built from src/*.ts by scripts/build.ts. Edit the sources, not this file. */\n` + (await bundle.outputs[0]!.text()));

const TYPES: Record<string, string> = { html: "Webpage (HTML)", js: "Script (JScript)", css: "Style Sheet (CSS)" };
console.log(`Web resources in dist/${base}/ (upload each with exactly this name):\n`);
for (const f of ["index.html", "pilar.js", "app.js", "pilar.css", "grid.css"]) {
  const size = Bun.file(join(out, f)).size;
  console.log(`  ${`${base}/${f}`.padEnd(32)} ${TYPES[f.split(".").pop()!]!.padEnd(18)} ${(size / 1024).toFixed(1)} KB`);
}
```

### `mock/data.ts`

```ts
// Deterministic sample accounts shaped like Dataverse Web API rows, with formatted-value annotations.
const F = "@OData.Community.Display.V1.FormattedValue";

const A = ["Northwind", "Contoso", "Fabrikam", "Adventure", "Litware", "Tailspin", "Proseware", "Wingtip", "Alpine", "Coho",
  "Lamna", "Margie's", "Trey", "Blue Yonder", "Woodgrove", "Humongous", "Wide World", "Fourth Coffee", "Relecloud", "Graphic"];
const B = ["Traders", "Logistics", "Energy", "Systems", "Foods", "Robotics", "Health", "Motors", "Studios", "Supply", "Labs", "Partners"];
const CITIES = ["Austin", "Fremont", "Reno", "Berlin", "Shanghai", "Toronto", "Seattle", "Denver", "Monterrey", "Oslo", "Lathrop", "Buffalo"];
const PEOPLE = ["Ana Ruiz", "Ben Carter", "Chloe Park", "Dev Patel", "Elena Rossi", "Femi Adeyemi", "Grace Liu", "Hugo Martin",
  "Isla Novak", "Jonas Berg", "Kira Sato", "Leo Duarte", "Maya Cohen", "Nils Eriksen", "Omar Haddad", "Priya Nair"];
const OWNERS = ["Riley Chen", "Alex Kim", "Sam Rivera", "Jordan Lee", "Taylor Brooks"];

// Small seeded PRNG so every run serves the same data.
function rng(seed: number) {
  return () => { seed = (seed * 1664525 + 1013904223) % 4294967296; return seed / 4294967296; };
}
const guid = (n: number, salt: number) => {
  const h = ((n + 1) * 2654435761 + salt * 97) % 4294967296;
  const hex = h.toString(16).padStart(8, "0");
  return `${hex}-${(n % 65536).toString(16).padStart(4, "0")}-4${hex.slice(0, 3)}-a${hex.slice(3, 6)}-${hex}${(salt % 65536).toString(16).padStart(4, "0")}`;
};

const usd = new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" });
const int = new Intl.NumberFormat("en-US");
const when = (d: Date) => d.toLocaleString("en-US", { month: "numeric", day: "numeric", year: "numeric", hour: "numeric", minute: "2-digit", timeZone: "UTC" });

export function makeAccounts(count = 240): Record<string, unknown>[] {
  const r = rng(42);
  const pick = <T,>(xs: T[]) => xs[Math.floor(r() * xs.length)]!;
  const start = Date.UTC(2026, 10, 14, 17, 0);
  return Array.from({ length: count }, (_, i) => {
    const active = r() > 0.18;
    const revenue = r() > 0.1 ? Math.round(r() * 48_000_000 + 250_000) : null;
    const employees = r() > 0.08 ? Math.round(r() * 9_000 + 12) : null;
    const contact = r() > 0.12 ? pick(PEOPLE) : null;
    const owner = pick(OWNERS);
    const modified = new Date(start - i * 3_600_000 * (2 + r() * 9));
    return {
      "@odata.etag": `W/"${100000 + i}"`,
      accountid: guid(i, 1),
      name: `${pick(A)} ${pick(B)}`,
      accountnumber: `ACC-${String(10001 + i)}`,
      statuscode: active ? 1 : 2, [`statuscode${F}`]: active ? "Active" : "Inactive",
      _primarycontactid_value: contact ? guid(i, 2) : null, ...(contact ? { [`_primarycontactid_value${F}`]: contact } : {}),
      address1_city: r() > 0.05 ? pick(CITIES) : null,
      telephone1: r() > 0.1 ? `+1 (${200 + Math.floor(r() * 700)}) 555-${String(Math.floor(r() * 10000)).padStart(4, "0")}` : null,
      revenue, ...(revenue !== null ? { [`revenue${F}`]: usd.format(revenue) } : {}),
      numberofemployees: employees, ...(employees !== null ? { [`numberofemployees${F}`]: int.format(employees) } : {}),
      _ownerid_value: guid(OWNERS.indexOf(owner), 3), [`_ownerid_value${F}`]: owner,
      modifiedon: modified.toISOString().replace(/\.\d+Z$/, "Z"), [`modifiedon${F}`]: when(modified),
    };
  });
}
```

### `mock/server.ts`

```ts
// Local stand-in for Dataverse: serves the built web resources under /WebResources/ (the same relative
// layout as in Dataverse) and answers the Web API calls the app makes under /api/data/v9.2/.
//   bun mock/server.ts            → http://localhost:5181
// Test switches:  POST /__mock/fail?status=403   (accounts request fails; status=0 turns it off)
//                 POST /__mock/layouts-missing?on=1   (layout table returns 404, like an uncreated table)
//                 POST /__mock/reset             (clears saved layouts)
//                 GET  /__mock/host              (the grid inside a fake form iframe with a fake parent Xrm)
import { join, normalize } from "node:path";
import { makeAccounts } from "./data";

const ROOT = join(import.meta.dir, "..");
const DIST = join(ROOT, "dist");
const STORE = join(ROOT, ".mock-store.json");
export const MOCK_USER_ID = "7d1c2b3a-0000-4000-a000-00000000c0de";
const API = "/api/data/v9.2/";
const LAYOUT_SET = "new_gridlayouts";

type Layout = { new_gridlayoutid: string; new_name: string; new_layout: string; _ownerid_value: string };

export function startMockServer({ port = 5181, latencyMs = 250, persist = true } = {}) {
  const accounts = makeAccounts();
  let layouts: Layout[] = [];
  let failStatus = 0, layoutsMissing = false;

  const loadStore = async () => {
    if (!persist) return;
    try { layouts = JSON.parse(await Bun.file(STORE).text()); } catch { layouts = []; }
  };
  const saveStore = () => (persist ? Bun.write(STORE, JSON.stringify(layouts, null, 1)) : Promise.resolve());
  const ready = loadStore();

  const json = (body: unknown, status = 200, headers: Record<string, string> = {}) =>
    new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json; odata.metadata=minimal", "OData-Version": "4.0", ...headers } });
  const odataError = (status: number, code: string, message: string) => json({ error: { code, message } }, status);
  const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

  function listAccounts(url: URL, req: Request) {
    if (failStatus) return odataError(failStatus, "0x80040220", `Principal user is missing prvReadAccount privilege (mock ${failStatus}).`);
    const q = url.searchParams;
    const select = q.get("$select")?.split(",").map((s) => s.trim()).filter(Boolean);
    let rows = accounts.slice();
    const orderBy = q.get("$orderby");
    if (orderBy) {
      const [field, dir] = orderBy.split(/\s+/);
      rows.sort((a, b) => String(a[field!] ?? "").localeCompare(String(b[field!] ?? "")) * (dir === "desc" ? -1 : 1));
    }
    const filter = q.get("$filter");
    const eq = filter?.match(/^(\w+) eq (\d+)$/);
    if (eq) rows = rows.filter((r) => String(r[eq[1]!]) === eq[2]);
    else if (filter) return odataError(400, "0x80060888", `Mock only supports "<field> eq <number>" filters, got: ${filter}`);

    const pageSize = Number(req.headers.get("Prefer")?.match(/odata\.maxpagesize=(\d+)/)?.[1] ?? 5000);
    const skip = Number(q.get("$skiptoken") ?? 0);
    const page = rows.slice(skip, skip + pageSize).map((r) => {
      if (!select) return r;
      const out: Record<string, unknown> = { "@odata.etag": r["@odata.etag"] };
      for (const [k, v] of Object.entries(r)) if (select.includes(k.split("@")[0]!)) out[k] = v;
      return out;
    });
    const body: Record<string, unknown> = { "@odata.context": `${url.origin}${API}$metadata#accounts`, value: page };
    if (skip + pageSize < rows.length) {
      const next = new URL(url); next.searchParams.set("$skiptoken", String(skip + pageSize));
      body["@odata.nextLink"] = next.toString();
    }
    return json(body);
  }

  async function layoutsApi(req: Request, url: URL, path: string) {
    if (layoutsMissing) return odataError(404, "0x80060888", `Resource not found for the segment '${LAYOUT_SET}'.`);
    const byId = path.match(/^new_gridlayouts\(([0-9a-f-]{36})\)$/i);
    if (req.method === "GET" && !byId) {
      const f = url.searchParams.get("$filter") ?? "";
      const name = f.match(/new_name eq '((?:[^']|'')*)'/)?.[1]?.replace(/''/g, "'");
      const owner = f.match(/_ownerid_value eq ([0-9a-f-]{36})/i)?.[1];
      const value = layouts.filter((l) => (!name || l.new_name === name) && (!owner || l._ownerid_value === owner));
      return json({ value: value.slice(0, Number(url.searchParams.get("$top") ?? 5000)) });
    }
    if (req.method === "POST" && !byId) {
      const body = (await req.json()) as Partial<Layout>;
      const id = crypto.randomUUID();
      layouts.push({ new_gridlayoutid: id, new_name: body.new_name ?? "", new_layout: body.new_layout ?? "", _ownerid_value: MOCK_USER_ID });
      await saveStore();
      return new Response(null, { status: 204, headers: { "OData-EntityId": `${url.origin}${API}${LAYOUT_SET}(${id})` } });
    }
    if (req.method === "PATCH" && byId) {
      const row = layouts.find((l) => l.new_gridlayoutid === byId[1]);
      if (!row) return odataError(404, "0x80040217", "new_gridlayout With Id = " + byId[1] + " Does Not Exist");
      Object.assign(row, (await req.json()) as Partial<Layout>);
      await saveStore();
      return new Response(null, { status: 204 });
    }
    return odataError(405, "0x8006088a", `${req.method} not supported by the mock on ${path}`);
  }

  const server = Bun.serve({
    port,
    async fetch(req) {
      await ready;
      const url = new URL(req.url);
      const p = decodeURIComponent(url.pathname);

      if (p === "/__mock/host") {
        // Imitates a model-driven form hosting the web resource in an iframe, with a fake parent Xrm.
        const first = [...new Bun.Glob("*_/*/index.html").scanSync(DIST)][0] ?? "";
        return new Response(`<!doctype html><meta charset="utf-8"><title>Fake form</title>
<style>body{margin:0;font:14px "Segoe UI",system-ui;background:#f3f2f1}header{padding:10px 16px;background:#1f1f1f;color:#fff}
#log{padding:8px 16px;font:12px ui-monospace,monospace;color:#333}iframe{display:block;width:calc(100% - 32px);height:560px;margin:0 16px;border:1px solid #ccc;background:#fff}</style>
<header>Fake model-driven form · web resource below</header><div id="log">openForm calls appear here</div>
<iframe src="/WebResources/${first}" title="grid"></iframe>
<script>window.Xrm={Navigation:{openForm:function(o){document.getElementById('log').textContent='Xrm.Navigation.openForm '+JSON.stringify(o);return Promise.resolve();}}};</script>`,
          { headers: { "Content-Type": "text/html; charset=utf-8" } });
      }

      if (p.startsWith("/__mock/")) {
        if (p === "/__mock/fail") failStatus = Number(url.searchParams.get("status") ?? 0);
        else if (p === "/__mock/layouts-missing") layoutsMissing = url.searchParams.get("on") === "1";
        else if (p === "/__mock/reset") { layouts = []; await saveStore(); }
        return json({ failStatus, layoutsMissing, layouts: layouts.length });
      }

      if (p.startsWith(API)) {
        await sleep(latencyMs);
        const path = p.slice(API.length);
        if (path === "WhoAmI") return json({ UserId: MOCK_USER_ID, BusinessUnitId: crypto.randomUUID(), OrganizationId: crypto.randomUUID() });
        if (path === "accounts") return listAccounts(url, req);
        if (path.startsWith(LAYOUT_SET)) return layoutsApi(req, url, path);
        return odataError(404, "0x80060888", `Resource not found for the segment '${path}'.`);
      }

      if (p === "/") {
        const first = [...new Bun.Glob("*_/*/index.html").scanSync(DIST)][0];
        return first ? Response.redirect(`/WebResources/${first}`, 302) : new Response("Run `bun run build` first.", { status: 503 });
      }
      if (p.startsWith("/WebResources/")) {
        const file = normalize(join(DIST, p.slice("/WebResources/".length)));
        if (!file.startsWith(DIST)) return new Response("Forbidden", { status: 403 });
        const f = Bun.file(file);
        if (await f.exists()) return new Response(f, { headers: { "Cache-Control": "no-store" } });
      }
      return new Response("Not found", { status: 404 });
    },
  });
  return server;
}

if (import.meta.main) {
  const server = startMockServer({ port: Number(process.env.PORT ?? 5181) });
  console.log(`Mock Dataverse on http://localhost:${server.port}  (user ${MOCK_USER_ID})`);
}
```

### `tests/patch.test.ts`

```ts
import { expect, test } from "bun:test";
import { patchPilar } from "../scripts/patch-pilar";

const original = await Bun.file(new URL("../vendor/moxen-pilar/pilar.js", import.meta.url)).text();
const patched = patchPilar(original);

test("every patch applies to the vendored pilar.js", () => {
  expect(patched).toContain("pinned left");
  expect(patched).toContain("data-pilar-quieto");
  expect(patched).toContain("op.estados || ESTADOS");
});

test("no Spanish UI strings are left in code (comments and sample data excepted)", () => {
  const code = patched
    .replace(/\/\*[\s\S]*?\*\//g, "")
    .split("\n")
    .filter((l) => !/^\s*\/\//.test(l) && !/^\s*\[?'PD-\d+/.test(l)) // drop line comments and the sample FILAS rows
    .join("\n");
  const literals = [...code.matchAll(/'([^'\\\n]|\\.)*'/g)].map((m) => m[0]);
  const spanish = literals.filter((s) => /[áéíóúñ¿¡]|\b(de|columna|fija|oculta|posición|Tabla|Filas|visibles|ninguna|Soltar|Fijar)\b/i.test(s)
    && !/^'(es-fija|oculta|fija)'$/.test(s) && !/^'\[data-[\w-]+\]'$/.test(s)); // class names and selectors aren't UI text
  // What's allowed to remain: the sample data constants (COLUMNAS names, ESTADOS) that this app never shows.
  const allowed = /^'(Pedido|Estado|Canal|Región|Producto|Unidades|Importe|Descuento|Fecha|Entrega|Pagado|Enviado|Pendiente|Devuelto)'$/;
  expect(spanish.filter((s) => !allowed.test(s))).toEqual([]);
});

test("a missing anchor fails loudly instead of half-patching", () => {
  expect(() => patchPilar(original.replace("'Movimiento cancelado'", "'x'"))).toThrow(/expected 1 match/);
});
```

### `tests/mapping.test.ts`

```ts
import { expect, test } from "bun:test";
import { config, type GridConfig } from "../src/config";
import { buildQuery, toPilar } from "../src/mapping";

const F = "@OData.Community.Display.V1.FormattedValue";

test("query selects the id, plain fields and lookups as _x_value, with filter and order", () => {
  const q = buildQuery({ ...config, filter: "statecode eq 0" });
  expect(q.startsWith("accounts?$select=accountid,name,accountnumber,statuscode,_primarycontactid_value,")).toBe(true);
  expect(q).toContain("_ownerid_value");
  expect(q).toContain("$filter=statecode%20eq%200");
  expect(q).toContain("$orderby=modifiedon%20desc");
});

test("rows use formatted values except for text/id, and dashes for empty cells", () => {
  const cfg: GridConfig = {
    ...config,
    columns: [
      { field: "name", label: "Name", kind: "text" },
      { field: "revenue", label: "Revenue", kind: "money" },
      { field: "primarycontactid", label: "Contact", kind: "lookup" },
      { field: "statuscode", label: "Status", kind: "status" },
      { field: "telephone1", label: "Phone", kind: "text" },
    ],
  };
  const { columnas, filas, estados, ids } = toPilar(cfg, [
    { accountid: "a1", name: "Contoso", revenue: 1250, [`revenue${F}`]: "$1,250.00", _primarycontactid_value: "c1",
      [`_primarycontactid_value${F}`]: "Ana Ruiz", statuscode: 1, [`statuscode${F}`]: "Active", telephone1: null },
    { accountid: "a2", name: "Fabrikam", revenue: null, _primarycontactid_value: null, statuscode: 2, [`statuscode${F}`]: "Inactive" },
  ]);
  expect(columnas.map((c) => c.tipo)).toEqual(["texto", "num", "texto", "estado", "texto"]);
  expect(filas).toEqual([
    ["Contoso", "$1,250.00", "Ana Ruiz", "1", "—"],
    ["Fabrikam", "—", "—", "2", "—"],
  ]);
  expect(estados).toEqual({ "1": { texto: "Active", tono: "filled" }, "2": { texto: "Inactive", tono: "muted" } });
  expect(ids).toEqual(["a1", "a2"]);
});
```

### `tests/layout.test.ts`

```ts
import { expect, test } from "bun:test";
import { applyRestore, planRestore, type PilarApi, type PilarState } from "../src/layout";

/** A stand-in with the same ordering rules as pilar.js (fija appends to the pinned group; mueve's position
 *  is counted within the column's own group, excluding itself; pinning a hidden column does nothing). */
function fakePilar(ids: string[], widths: Record<string, number>): PilarApi {
  let orden = ids.slice();
  const fija = new Set<string>(), oculta = new Set<string>(), anchos = { ...widths };
  let densidad = "normal";
  return {
    fija(id, v) {
      if (fija.has(id) === v || oculta.has(id)) return;
      v ? fija.add(id) : fija.delete(id);
      const sin = orden.filter((k) => k !== id);
      sin.splice(sin.filter((k) => fija.has(k)).length, 0, id);
      orden = sin;
    },
    mueve(id, pos) {
      const grupo = orden.filter((k) => fija.has(k) === fija.has(id) && k !== id);
      const i = Math.max(0, Math.min(grupo.length, pos));
      const sin = orden.filter((k) => k !== id);
      const idx = i < grupo.length ? sin.indexOf(grupo[i]!) : sin.indexOf(grupo[grupo.length - 1]!) + 1;
      sin.splice(idx, 0, id);
      orden = sin;
    },
    oculta(id, v) { v ? oculta.add(id) : oculta.delete(id); },
    ancho(id, px) { anchos[id] = px; },
    densidad(d) { densidad = d; },
    estado: () => ({ orden: orden.slice(), anchos: { ...anchos }, fijas: orden.filter((k) => fija.has(k)), ocultas: orden.filter((k) => oculta.has(k)), densidad }),
  };
}

const IDS = ["a", "b", "c", "d", "e", "f"];
const W = Object.fromEntries(IDS.map((id) => [id, 120]));

test("restoring reproduces order, pins, widths, hidden columns and density", () => {
  const saved: PilarState = { orden: ["e", "b", "f", "a", "d", "c"], fijas: ["e", "b"], ocultas: ["d"], anchos: { ...W, f: 200 }, densidad: "compacta" };
  const pilar = fakePilar(IDS, W);
  applyRestore(pilar, planRestore(saved, pilar.estado()));
  expect(pilar.estado()).toEqual(saved);
});

test("a hidden pinned column is pinned before it is hidden", () => {
  const saved: PilarState = { orden: ["c", "a", "b", "d", "e", "f"], fijas: ["c"], ocultas: ["c"], anchos: W, densidad: "normal" };
  const pilar = fakePilar(IDS, W);
  applyRestore(pilar, planRestore(saved, pilar.estado()));
  expect(pilar.estado().fijas).toEqual(["c"]);
  expect(pilar.estado().ocultas).toEqual(["c"]);
});

test("columns added or removed since the layout was saved don't break the restore", () => {
  const saved: PilarState = { orden: ["old", "c", "a", "b"], fijas: ["old", "c"], ocultas: ["old"], anchos: { old: 300, a: 160 }, densidad: "comoda" };
  const pilar = fakePilar(IDS, W);
  applyRestore(pilar, planRestore(saved, pilar.estado()));
  const s = pilar.estado();
  expect(s.orden).toEqual(["c", "a", "b", "d", "e", "f"]); // new columns d, e, f go to the end
  expect(s.fijas).toEqual(["c"]);
  expect(s.anchos.a).toBe(160);
  expect(s.densidad).toBe("comoda");
});

test("never hides every column", () => {
  const saved: PilarState = { orden: IDS, fijas: [], ocultas: IDS, anchos: W, densidad: "normal" };
  const ops = planRestore(saved, fakePilar(IDS, W).estado());
  expect(ops.filter((o) => o.op === "oculta").length).toBe(IDS.length - 1);
});
```

### `tests/api.test.ts`

```ts
// The Web API client and layout store against the mock Dataverse server.
import { afterAll, beforeEach, expect, test } from "bun:test";
import { startMockServer, MOCK_USER_ID } from "../mock/server";
import { config } from "../src/config";
import { Dataverse, DataverseError, isMissingResource } from "../src/dataverse";
import { DataverseLayoutStore, type PilarState } from "../src/layout";
import { buildQuery } from "../src/mapping";

const server = startMockServer({ port: 0, latencyMs: 0, persist: false });
const origin = `http://localhost:${server.port}`;
const api = new Dataverse(`${origin}/api/data/v9.2/`);
const mock = (path: string) => fetch(origin + path, { method: "POST" });
const store = () => new DataverseLayoutStore(api, config.layoutStore as Extract<typeof config.layoutStore, { kind: "dataverse" }>, "test-grid", MOCK_USER_ID);

afterAll(() => server.stop(true));
beforeEach(async () => { await mock("/__mock/reset"); await mock("/__mock/fail?status=0"); await mock("/__mock/layouts-missing?on=0"); });

test("retrieveAll follows nextLink pages and stops at maxRows", async () => {
  const all = await api.retrieveAll(buildQuery(config), { pageSize: 100, maxRows: 500 });
  expect(all.records.length).toBe(240);
  expect(all.pages).toBe(3);
  expect(all.truncated).toBe(false);
  const capped = await api.retrieveAll(buildQuery(config), { pageSize: 100, maxRows: 150 });
  expect(capped.records.length).toBe(150);
  expect(capped.truncated).toBe(true);
});

test("records come back with formatted values and only the selected columns", async () => {
  const { records } = await api.retrieveAll(buildQuery(config), { pageSize: 5, maxRows: 5 });
  const r = records[0]!;
  expect(r["statuscode@OData.Community.Display.V1.FormattedValue"]).toMatch(/Active|Inactive/);
  expect(Object.keys(r).some((k) => k.startsWith("_ownerid_value"))).toBe(true);
  expect(Object.keys(r)).not.toContain("address1_line1"); // not selected, not returned
});

test("errors carry the status and Dataverse message", async () => {
  await mock("/__mock/fail?status=403");
  const err = await api.retrieveAll(buildQuery(config), { pageSize: 10, maxRows: 10 }).catch((e) => e);
  expect(err).toBeInstanceOf(DataverseError);
  expect(err.status).toBe(403);
  expect(err.message).toContain("prvReadAccount");
});

test("layout store creates once, then updates the same record", async () => {
  const s = store();
  expect(await s.load()).toBeNull();
  const a: PilarState = { orden: ["name"], anchos: { name: 200 }, fijas: [], ocultas: [], densidad: "compacta" };
  await s.save(a);
  await s.save({ ...a, densidad: "comoda" });
  const status = (await (await mock("/__mock/status")).json()) as { layouts: number };
  expect(status.layouts).toBe(1);
  expect((await store().load())?.densidad).toBe("comoda");
});

test("a missing layout table is recognised so the app can fall back", async () => {
  await mock("/__mock/layouts-missing?on=1");
  const err = await store().load().catch((e) => e);
  expect(isMissingResource(err)).toBe(true);
});

test("whoAmI returns the user id", async () => {
  expect(await api.whoAmI()).toBe(MOCK_USER_ID);
});
```
