# MapAnalysisDemo — Technical Specification

> **Version:** 2.0  
> **Last Updated:** May 2026  
> **Type:** Single-Page Analytics Dashboard with Map Visualization

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Feature Catalog](#3-feature-catalog)
4. [Function Catalog](#4-function-catalog)
5. [Data Structures & State](#5-data-structures--state)
6. [UI Components](#6-ui-components)
7. [CSS Design System](#7-css-design-system)
8. [Leaflet Integration](#8-leaflet-integration)
9. [Scenario Flows](#9-scenario-flows)
10. [Implementation Status](#10-implementation-status)

---

## 1. Project Overview

**MapAnalysisDemo** is a self-contained, single-file analytics dashboard with interactive map visualization. Built entirely in `index.html` (~2243 lines), it uses **vanilla JavaScript**, **Leaflet.js** for mapping, and **SheetJS** for spreadsheet parsing — no build tools, bundlers, or frameworks.

### Primary Use Case

Import CSV/XLSX files with geographic coordinates, visualize them on an interactive map, explore data in a sortable/filterable table, analyze with a distance measurement tool, and customize presentation (icons, labels, colors) per-dataset and per-point.

### Key Design Principles

| Principle | Description |
|---|---|
| **Single-file** | Entire application in one HTML file — zero deployment friction |
| **No build step** | Vanilla JS with CDN-loaded libraries |
| **Immediate startup** | Ships with 3 sample datasets loaded on page load |
| **Self-contained data** | 4 built-in POI datasets generated synthetically (no API calls) |
| **Stateful modals** | Each operation uses a consistent modal pattern with target state variables |

### External Dependencies

| Library | Version | Source | Purpose |
|---|---|---|---|
| Leaflet.js | 1.9.4 | `unpkg.com/leaflet@1.9.4` | Interactive map rendering |
| SheetJS | 0.20.3 | `cdn.sheetjs.com/xlsx-0.20.3` | Excel/CSV file parsing |
| CartoDB | — | `basemaps.cartocdn.com` | Map tile layers (dark + light) |

---

## 2. Architecture

### 2.1 Layout Structure

```
┌─────────────────────────────────────────────────────────┐
│  Header (48px)                                          │
│  ┌ Title ──── MOCKUP badge ───────── Theme ── Counts ┐  │
├─────────────────────────────────────────────────────────┤
│  Sidebar (280px)  │         Center (flex: 1)            │
│  ┌──────────────┐ │  ┌──────────────────────────────┐  │
│  │ My Sets      │ │  │  Tab Bar: [Table] [Map]      │  │
│  │  ── file 1   │ │  ├──────────────────────────────┤  │
│  │  ── file 2   │ │  │  Table View or Map View      │  │
│  │ Built-in     │ │  │  (Leaflet.js)                │  │
│  │  ── Embassies│ │  │                              │  │
│  │  ── Banks    │ │  │                              │  │
│  │  ── Gov't    │ │  │                              │  │
│  │  ── Poland   │ │  │                              │  │
│  └──────────────┘ │  └──────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  Details Panel (slide-in from right, 340px)              │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐    │
│  │  Point data key/value rows                        │    │
│  │  Toolbar: [Label] [Icon]                          │    │
│  │  Inline icon picker                               │    │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘    │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
│  User Event  │────▶│  Event Handler│────▶│  State Mutation  │
│  (click,     │     │  (function)   │     │  (FILES, globals)│
│   input,     │     │               │     │                  │
│   drag)      │     │               │     │                  │
└─────────────┘     └──────────────┘     └──────────┬───────┘
                                                     │
┌──────────────────┐     ┌──────────────────┐        │
│  UI Update       │◀────│  Render Pipeline  │◀───────┘
│  (DOM, Leaflet,  │     │  renderSidebar()  │
│   badges)        │     │  renderTable()    │
└──────────────────┘     │  renderMap()      │
                         │  updateCounts()   │
                         └──────────────────┘
```

### 2.3 Module Organization (within single file)

| Section | Lines (approx.) | Content |
|---|---|---|
| **CSS** | 1–380 | Custom properties, reset, layout, sidebar, table, map, details panel, modals, themes, animations |
| **HTML** | 381–680 | Static DOM structure: header, sidebar, main panes, details panel, modal dialogs |
| **JS - Data** | 681–900 | Built-in set definitions, icon picker options, sample data generators |
| **JS - State** | 901–950 | Global variables and initialization |
| **JS - Core** | 951–1400 | Rendering functions, event handlers, dialog logic |
| **JS - Map** | 1401–1700 | Leaflet integration, measurement tool, blink animation |
| **JS - Utility** | 1701–1800 | `parseCoord()`, `detectCoordFormat()`, `formatDist()`, `getLabelText()` |
| **JS - Init** | 1801–end | Start-up sequence, initial render calls |

---

## 3. Feature Catalog

### 3.1 Data Management

| ID | Feature | Description | Trigger | Status |
|---|---|---|---|---|
| **F1** | **Import CSV/XLSX** | Open file dialog for `.xlsx`, `.xls`, `.csv` files. Parses with SheetJS, shows preview with column mapping | `+ Import` button | ✅ |
| **F2** | **Coordinate auto-detection** | Automatically identifies lat/lng columns by regex (`/lat/i`, `/lng\|lon/i`) | On import dialog open | ✅ |
| **F3** | **Multi-format coordinate parsing** | Parses DD (decimal), DDM (deg min), DMS (deg min sec) with directional suffixes (N/S/E/W) | On confirm import | ✅ |
| **F4** | **Merge datasets** | Union-merge selected files' data — collects all unique keys, fills missing values with `''` | `Merge` button | ✅ |
| **F5** | **Remove dataset** | Confirmation dialog; re-selects nearest neighbor if active file removed | `×` on file item | ✅ |
| **F6** | **Rename dataset** | Inline text input with auto-focus + select-all; Enter key to confirm | `✎` on file item | ✅ |
| **F7** | **Dataset visibility** | Show/hide dataset on map; visual red strikethrough on eye icon when hidden | Eye button on file item | ✅ |
| **F8** | **Built-in POI sets** | 4 synthetic datasets: Embassies in Europe, Banks in Europe, Government Locations, Embassies in Poland | "Built-in" sidebar tab | ✅ |
| **F9** | **Add new point** | Modal with dynamic fields matching active dataset schema + map-click coordinate picking | `+ Point` button | 🔜 |
| **F10** | **Dataset-level icon** | 20-emoji picker applied to all points in a dataset | 🎨 on file item | ✅ |

### 3.2 Map Visualization

| ID | Feature | Description | Status |
|---|---|---|---|
| **M1** | **Point markers** | Circle markers (color-coded per dataset, radius 7, white stroke) or emoji divIcon overlays | ✅ |
| **M2** | **Per-point icon override** | Inline emoji picker in details panel sets `row._icon` for an individual point | ✅ |
| **M3** | **Track mode** | Polyline connecting points in row order; start marker (green) + end marker (red) with popups | ✅ |
| **M4** | **Combined mode** | Both point markers and track polyline visible simultaneously | ✅ |
| **M5** | **Dataset labels** | Toggle labels for all points in a dataset via `file.showLabels` | ✅ |
| **M6** | **Per-point labels** | Toggle label for individual point via `row._showLabel` (overrides dataset setting) | ✅ |
| **M7** | **Auto label extraction** | `getLabelText()` checks `name`, `city`, `store`, `respondent` keys; falls back to `#id` or first non-null value | ✅ |
| **M8** | **Per-point custom label** | Text input in details panel to set arbitrary label text via `row._label` | 🔜 |
| **M9** | **Map click → details** | Click any marker to open the details panel with all row data | ✅ |
| **M10** | **Zoom to dataset** | Clicking a sidebar dataset fits map to its bounding box | ✅ |
| **M11** | **Blink animation** | Selected dataset markers flash red → fade back to original over 2s (1s each direction) | ✅ |
| **M12** | **Measurement tool** | Click map to place draggable waypoints; dashed polylines with segment distances + cumulative total | ✅ |
| **M13** | **Theme-aware tiles** | Dark CartoDB tiles in dark mode, light tiles in light mode, swapped via `tileLayer.setUrl()` | ✅ |
| **M14** | **Map click coordinate pick** | Map enters crosshair mode, single click fills both lat + lng fields (for Add Point) | 🔜 |
| **M15** | **Popup on click** | Shows all non-lat/lng fields as `<b>key:</b> value` pairs | ✅ |

### 3.3 Table View

| ID | Feature | Description | Status |
|---|---|---|---|
| **T1** | **Column sorting** | Click column header to sort ascending/descending; visual arrow indicator (▲/▼) | ✅ |
| **T2** | **Global text search** | Free-text filter across all columns simultaneously | ✅ |
| **T3** | **Per-column filters** | Individual text input under each column header for column-specific filtering | ✅ |
| **T4** | **Row selection → details** | Click any row to open details panel and center map on that point (zoom 10) | ✅ |
| **T5** | **Row count display** | Shows "X of Y rows" with filtered count / total count | ✅ |
| **T6** | **Clear filter button** | × button appears when filters are active, clears global search | ✅ |

### 3.4 Details Panel

| ID | Feature | Description | Status |
|---|---|---|---|
| **D1** | **Slide-in panel** | Animated 340px panel slides from right; 0.3s cubic-bezier transition | ✅ |
| **D2** | **Key/value display** | All row fields rendered as styled key/value rows with bottom borders | ✅ |
| **D3** | **Per-point label toggle** | Override dataset-level label visibility for individual point | ✅ |
| **D4** | **Per-point icon picker** | Inline 6-column emoji grid; sets `row._icon` override | ✅ |
| **D5** | **Per-point label editing** | Text input field to set custom label text via `row._label` | 🔜 |

### 3.5 Theme & UX

| ID | Feature | Description | Status |
|---|---|---|---|
| **U1** | **Dark/Light theme** | Moon toggle in header swaps all CSS custom properties and map tiles | ✅ |
| **U2** | **Main tab switching** | Table / Map tabs with active state and count badges | ✅ |
| **U3** | **Total record counter** | Header displays sum of `file.rows` across all datasets | ✅ |
| **U4** | **Sidebar tab system** | "My Sets" (imported) and "Built-in" (synthetic) views with active state | ✅ |
| **U5** | **Loading indicators** | `status = 'loading'` disables built-in POI items during data generation | ✅ |
| **U6** | **Empty states** | Placeholder content when no datasets exist or import dialog is empty | ✅ |
| **U7** | **Measurement distance** | Gold-styled distance markers at segment midpoints; "Total: X.XX km" | ✅ |

---

## 4. Function Catalog

### 4.1 Data Generation Functions

| Function | Signature | Returns | Description |
|---|---|---|---|
| `generateSalesData` | `()` → `Array<Object>` | 15 rows | City sales data: id, city, lat, lng, region, revenue, units, growth |
| `generateStoreData` | `()` → `Array<Object>` | 12 rows | NYC store data: id, store, lat, lng, type, sqft, employees |
| `generateSurveyData` | `()` → `Array<Object>` | 20 rows | Survey responses: id, respondent, lat, lng, region, satisfaction, age |
| `generateEmbassiesData` | `()` → `Array<Object>` | 36–126 rows | European embassies across 18 capitals; 2–6 per city, jittered ±0.02° |
| `generateBanksData` | `()` → `Array<Object>` | 30–70 rows | European banks across 10 financial hubs; 3–7 per city, deduped, jittered ±0.015° |
| `generateGovernmentData` | `()` → `Array<Object>` | 36–84 rows | European govt buildings across 12 capitals; 3–6 per city, deduped, jittered ±0.0125° |
| `generatePolandEmbassiesData` | `()` → `Array<Object>` | 70 rows | Embassies in Warsaw; 70 countries positioned radially around city center |

### 4.2 Rendering Functions

| Function | Signature | Description |
|---|---|---|
| `renderBuiltinSets` | `()` → `void` | Renders `BUILTIN_SETS` array into `#builtinList` DOM with icons, names, and status text |
| `renderSidebar` | `()` → `void` | Renders `FILES` array into `#fileList` DOM with per-file action buttons (visibility, labels, icon, rename, remove) |
| `renderTable` | `()` → `void` | Orchestrates full table render: calls `renderTableHead()` then `renderTableBody()` |
| `renderTableHead` | `()` → `void` | Builds `<thead>` with sortable column headers (sort arrows + click handlers) and per-column filter inputs |
| `renderTableBody` | `()` → `void` | Filters via `getFilteredData()`, sorts via `sortData()`, renders `<tbody>` rows; updates `#currentFileLabel`, `#rowInfo`, `#clearFilter` visibility |
| `renderMap` | `(focusFileId?: string)` → `void` | Full map re-render: clears all non-tile layers, iterates visible files, draws points/tracks, fit-bounds to focus file or all visible coords |
| `renderMeasurements` | `()` → `void` | Re-renders measure tool: draggable gold circle markers, dashed polylines, midpoint distance labels, total distance display |
| `renderImportPreview` | `()` → `void` | Renders first 5 rows of `pendingImport` into the import dialog's preview table |

### 4.3 Event Handlers / UI Callbacks

| Function | Signature | Description |
|---|---|---|
| `switchSidebarTab` | `(tab: 'my' \| 'builtin')` → `void` | Toggles between My Sets and Built-in sidebar views |
| `selectFile` | `(id: string)` → `void` | Sets file active, resets sort, re-renders everything, triggers blink animation |
| `blinkDataset` | `(id: string)` → `void` | Animates dataset markers: red flash (sepia + blur filter) for 1s, restored over 1s |
| `toggleVisibility` | `(id: string)` → `void` | Toggles `file.visible`, re-renders sidebar (strikethrough eye) and map |
| `toggleDatasetLabels` | `(id: string)` → `void` | Toggles `file.showLabels`, clears all per-point `_showLabel` overrides, re-renders |
| `getLabelText` | `(row: object)` → `string` | Extracts display label: checks `name`, `city`, `store`, `respondent`, `_label`, then `#id`, then first non-null value |
| `showIconPicker` | `(id: string)` → `void` | Opens dataset icon picker modal with 20 emoji options |
| `selectIcon` | `(ic: string \| null)` → `void` | Sets `file.icon`, closes dialog, re-renders sidebar + map |
| `cancelIconPicker` | `()` → `void` | Clears `iconPickerTargetId`, closes modal |
| `selectTableRow` | `(idx: number)` → `void` | Selects row, opens details panel, centers map on point at zoom 10 |
| `setMapMode` | `(mode: 'points' \| 'track' \| 'both')` → `void` | Updates toggle buttons, re-renders map in selected mode |
| `switchTab` | `(name: 'table' \| 'map')` → `void` | Switches main content tab; invalidates map size on map tab switch |
| `updateCounts` | `()` → `void` | Updates tab badges (`#tableCount`, `#mapCount`) and header `#recordCount` (sum of all file rows) |
| `toggleTheme` | `()` → `void` | Toggles `.light` class on `<body>`, swaps map tile URL via `tileLayer.setUrl()` |
| `importFile` | `()` → `void` | Triggers click on hidden `<input #fileInput>` to open OS file dialog |
| `handleFileImport` | `(event: InputEvent)` → `void` | Reads file via `FileReader`, parses with SheetJS (array buffer for xlsx, text for csv), stores in `pendingImport`, calls `showImportDialog()` |
| `showImportDialog` | `()` → `void` | Populates lat/lng column selects with auto-detection + coordinate format detection, shows preview |
| `confirmImport` | `()` → `void` | Creates new `FILES` entry from `pendingImport`, parses coordinates via `parseCoord()`, re-renders everything |
| `cancelImport` | `()` → `void` | Clears `pendingImport`, closes import dialog |
| `showMergeDialog` | `()` → `void` | Opens merge dialog with checkboxes for all existing files (active file pre-checked) |
| `confirmMerge` | `()` → `void` | Validates ≥2 files selected, union-merges data (all keys, missing values filled with `''`), creates new file entry, re-renders |
| `cancelMerge` | `()` → `void` | Closes merge dialog |
| `showRemoveDialog` | `(id: string)` → `void` | Opens remove confirmation with file name displayed |
| `confirmRemove` | `()` → `void` | Removes file from `FILES`, selects nearest neighbor if active removed, re-renders |
| `cancelRemove` | `()` → `void` | Clears `removeTargetId`, closes dialog |
| `showRenameDialog` | `(id: string)` → `void` | Opens rename dialog pre-filled with current name, auto-focuses + selects all text |
| `confirmRename` | `()` → `void` | Validates non-empty name, updates `file.name`, re-renders sidebar + table |
| `cancelRename` | `()` → `void` | Clears `renameTargetId`, closes dialog |
| `onFilterChange` | `()` → `void` | Re-renders table body and map (for filtered data) when any filter input changes |
| `clearGlobalFilter` | `()` → `void` | Clears `#globalFilter` value, calls `onFilterChange()` |
| `sortBy` | `(key: string)` → `void` | Toggles sort direction if same key, or sets new sort key ascending; updates `sortKey`/`sortDir` globals, re-renders |
| `toggleMeasure` | `()` → `void` | Toggles measurement mode: attaches/removes map click handler, changes cursor to crosshair |
| `addMeasurePoint` | `(e: LeafletMouseEvent)` → `void` | Pushes `{lat, lng}` to `measurePoints`, shows clear button, calls `renderMeasurements()` |
| `clearMeasurements` | `()` → `void` | Clears `measurePoints`, removes measure layers, hides measure UI |
| `loadBuiltinSet` | `(id: string)` → `void` | Sets status to `'loading'`, calls appropriate generator, creates file entry, deactivates others, switches to My Sets tab, re-renders |
| `openDetails` | `(row: object)` → `void` | Opens details panel: sets title, renders toolbar (Label/Icon toggles), inline icon picker, all key/value rows |
| `togglePointIcon` | `()` → `void` | Shows/hides `#pointIconPicker` inline grid in details panel |
| `selectPointIcon` | `(ic: string \| null)` → `void` | Sets `row._icon`, re-opens details panel, re-renders map |
| `togglePointLabel` | `()` → `void` | Toggles `row._showLabel` boolean, re-opens details panel, re-renders map |
| `closeDetails` | `()` → `void` | Closes details panel, clears `currentDetailRow` |
| `showAddPointDialog` | `()` → `void` | (Planned) Opens Add Point modal with dynamic fields from active dataset schema |
| `enableMapPick` | `()` → `void` | (Planned) Puts map in crosshair selection mode for coordinate picking |
| `onMapPick` | `(e: LeafletMouseEvent)` → `void` | (Planned) Fills lat + lng fields after map click in pick mode |
| `confirmAddPoint` | `()` → `void` | (Planned) Appends new row to `file.data`, re-renders everything |

### 4.4 Utility Functions

| Function | Signature | Returns | Description |
|---|---|---|---|
| `getCurrentFile` | `()` → `FileEntry \| undefined` | Active `FILES` entry matching `currentFileId` |
| `getLatLngColumns` | `(file: FileEntry)` → `{latKey: string, lngKey: string}` | Identifies lat/lng column names: checks saved keys first, falls back to regex detection |
| `getFilteredData` | `()` → `Array<Object>` | Applies global filter + per-column filters to current file's data; returns filtered array |
| `sortData` | `(arr: Array<Object>)` → `Array<Object>` | Returns sorted shallow copy based on `sortKey`/`sortDir` globals; safe for non-existent keys |
| `formatDist` | `(m: number)` → `string` | Formats distance: ≥1000m → `"X.XX km"`, <1000m → `"X.X m"` |
| `detectCoordFormat` | `(values: string[])` → `'DD' \| 'DDM' \| 'DMS'` | Heuristic: checks for `°` with `"` → DMS, `°` with `'` → DDM, else DD |
| `parseCoord` | `(value: string \| number, format: 'DD' \| 'DDM' \| 'DMS')` → `number \| NaN` | Parses coordinate string to decimal degrees; handles directional suffixes (N/S/E/W), trims whitespace |

---

## 5. Data Structures & State

### 5.1 `FILES` — Master Dataset Registry (Global Array)

```typescript
interface FileEntry {
  id: string;              // Auto-incrementing: 'f1', 'f2', ...
  name: string;            // Display name (original filename or user-renamed)
  type: 'xlsx' | 'csv';   // Source file type
  size: string;            // Human-readable: '2.4 MB', '47 points'
  rows: number;            // Number of data rows
  cols: number;            // Number of columns
  active: boolean;         // Currently selected/highlighted in sidebar
  visible: boolean;        // Shown on map
  icon: string | null;     // Dataset-level emoji override (from picker)
  showLabels: boolean;     // Show labels on all points
  latKey: string;          // Latitude column name
  lngKey: string;          // Longitude column name
  coordFormat: 'DD' | 'DDM' | 'DMS';  // Coordinate format
  data: Array<RowObject>;  // Row data with optional per-point overrides
}
```

### 5.2 Row Object (with Per-Point Overrides)

```typescript
interface RowObject {
  [originalKey: string]: string | number | boolean | undefined;

  // Per-point overrides (not original data — injected at runtime):
  _showLabel?: boolean;     // Show label for this point (overrides file.showLabels)
  _icon?: string | null;    // Emoji icon override (overrides file.icon)
  _label?: string;          // Custom label text (checked first by getLabelText)
}
```

### 5.3 `BUILTIN_SETS` — Sidebar POI Definitions (Global Array, Read-Only)

```typescript
interface BuiltinSet {
  id: string;         // 'poi_embassies' | 'poi_banks' | 'poi_government' | 'poi_pl_embassies'
  name: string;       // Display name in sidebar
  icon: string;       // Emoji character
  status: string;     // '' (not loaded) | 'loading' | 'N points loaded'
}
```

### 5.4 `datasetMarkers` — Marker References for Blink Animation

```typescript
Record<string, L.Marker[]>  // Indexed by fileId
```

Each file's Leaflet marker references are stored here so `blinkDataset()` can iterate and animate them. Markers are also inside `L.layerGroup` instances on the map for rendering.

### 5.5 `measurePoints` — Measurement Waypoints

```typescript
Array<{ lat: number; lng: number }>  // Ordered list of click points
```

### 5.6 `pendingImport` — Temporary Import Buffer

```typescript
interface PendingImport {
  rows: Array<Object>;    // Parsed rows from SheetJS
  keys: Array<string>;    // Column names
  name: string;           // Original filename
  type: 'xlsx' | 'csv';   // File type
  size: string;           // Formatted file size
}
```

Holds data between file pick and import confirmation. Cleared on cancel or after successful import.

### 5.7 Global State Variables

| Variable | Type | Initial | Description |
|---|---|---|---|
| `FILES` | `Array<FileEntry>` | 3 sample entries | All datasets (imported + generated) |
| `currentFileId` | `string \| null` | `'f1'` | Currently selected file ID |
| `fileIdCounter` | `number` | `4` | Auto-incrementing ID counter |
| `sortKey` | `string \| null` | `null` | Current sort column |
| `sortDir` | `'asc' \| 'desc'` | `'asc'` | Sort direction |
| `mapMode` | `'points' \| 'track' \| 'both'` | `'points'` | Map rendering mode |
| `mapInstance` | `L.Map \| null` | `null` | Leaflet map (lazy initialized) |
| `tileLayer` | `L.TileLayer \| null` | `null` | Leaflet tile layer (for theme swap) |
| `measuring` | `boolean` | `false` | Measure tool active state |
| `measurePoints` | `Array<{lat,lng}>` | `[]` | Measurement waypoints |
| `pendingImport` | `PendingImport \| null` | `null` | Data awaiting import confirmation |
| `sidebarTab` | `'my' \| 'builtin'` | `'my'` | Active sidebar tab |
| `iconPickerTargetId` | `string \| null` | `null` | File ID for open icon picker |
| `removeTargetId` | `string \| null` | `null` | File ID pending removal |
| `renameTargetId` | `string \| null` | `null` | File ID pending rename |
| `currentDetailRow` | `object \| null` | `null` | Row displayed in details panel |
| `datasetMarkers` | `Record<string, L.Marker[]>` | `{}` | Marker refs for blink |
| `BUILTIN_SETS` | `BuiltinSet[]` | 4 entries | Read-only POI definitions |
| `ICON_PICKER_OPTIONS` | `string[]` | 20 emoji | Available icon choices |
| `DATASET_COLORS` | `string[]` | 10 hex colors | Cycled for dataset markers |

---

## 6. UI Components

### 6.1 Static HTML Elements by ID

| ID | Element Type | Role | Dynamic Content |
|---|---|---|---|
| `#themeSwitch` | `<input type="checkbox">` | Theme toggle | — |
| `#recordCount` | `<span>` | Header total record count | `updateCounts()` |
| `#fileInput` | `<input type="file">` | Hidden file picker (accepts `.xlsx,.xls,.csv`) | — |
| `#fileList` | `<div class="file-list">` | My Sets container | `renderSidebar()` |
| `#builtinList` | `<div class="file-list">` | Built-in POI container | `renderBuiltinSets()` |
| `#tableCount` | `<span class="tab-count">` | Table tab badge | `updateCounts()` |
| `#mapCount` | `<span class="tab-count">` | Map tab badge | `updateCounts()` |
| `#paneTable` | `<div class="content-pane">` | Table view pane | `switchTab()` toggles |
| `#paneMap` | `<div class="content-pane">` | Map view pane | `switchTab()` toggles |
| `#currentFileLabel` | `<h3>` | Active file name | `renderTableBody()` |
| `#globalFilter` | `<input type="text">` | Global search input | — |
| `#clearFilter` | `<button class="clear-btn">` | Clear search `×` | `.visible` class toggled |
| `#rowInfo` | `<span>` | "X of Y rows" text | `renderTableBody()` |
| `#dataTable` | `<table>` | Main data table | `renderTable()` |
| `#tableHead` | `<thead>` | Table header row | `renderTableHead()` |
| `#tableBody` | `<tbody>` | Table body | `renderTableBody()` |
| `#map` | `<div>` | Leaflet container | `renderMap()` |
| `#measureBtn` | `<button>` | Measure toggle | `.active` class toggled |
| `#clearMeasureBtn` | `<button>` | Clear measurements | `.hidden` toggled |
| `#totalDistDisplay` | `<span class="dist-total">` | Distance total container | `.hidden` toggled |
| `#totalDistVal` | `<span>` | Distance value | `renderMeasurements()` |
| `#detailsPanel` | `<div class="details-panel">` | Slide-in panel | `.open` class toggled |
| `#detailTitle` | `<h3>` | Panel title | `openDetails()` |
| `#detailBody` | `<div class="panel-body">` | Panel content | `openDetails()` |
| `#pointIconPicker` | `<div class="point-icon-picker">` | Inline icon grid | `openDetails()` / `togglePointIcon()` |

### 6.2 Modal Dialogs

| ID | Width | State Variable | Confirm | Cancel | Purpose |
|---|---|---|---|---|---|
| `#importDialog` | 680px | `pendingImport` | `confirmImport()` | `cancelImport()` | Column mapping, format selection, preview |
| `#mergeDialog` | 460px | — | `confirmMerge()` | `cancelMerge()` | File selection checklist |
| `#removeDialog` | 400px | `removeTargetId` | `confirmRemove()` | `cancelRemove()` | Delete confirmation |
| `#renameDialog` | 400px | `renameTargetId` | `confirmRename()` | `cancelRename()` | Inline name editing |
| `#iconPickerDialog` | 420px | `iconPickerTargetId` | `selectIcon(ic)` | `cancelIconPicker()` | Emoji grid selection |
| `#addPointDialog` | 480px | (pending) | `confirmAddPoint()` | `cancelAddPoint()` | Dynamic field form (🔜) |

### 6.3 CSS Class Architecture

| Class | Role | Key Properties |
|---|---|---|
| `.app` | Root flex column | `height: 100vh`, `display: flex`, `flex-direction: column` |
| `.header` | Top bar | `height: 48px`, `display: flex`, `align-items: center`, bordered |
| `.sidebar` | Left panel | `width: 280px`, `display: flex`, `flex-direction: column`, bordered |
| `.file-item` | Dataset row | `display: flex`, `align-items: center`, `padding: 10px 12px`, hover effects |
| `.file-item.active` | Selected state | `border-left: 3px solid var(--accent)` |
| `.details-panel` | Slide-in panel | `position: absolute`, `right: calc(-1 * var(--panel-w))`, `transition: right 0.3s cubic-bezier(0.4, 0, 0.2, 1)` |
| `.details-panel.open` | Visible state | `right: 0` |
| `.modal-overlay` | Dialog backdrop | `position: fixed`, `inset: 0`, `background: rgba(0,0,0,0.6)`, `z-index: 2000`, flex centering |
| `.modal` | Dialog card | `background: var(--surface)`, `border-radius: 12px`, `box-shadow: 0 16px 48px rgba(0,0,0,0.4)` |
| `.marker-icon` | Leaflet icon container | `background: none`, `border: none`, `text-align: center` |
| `.marker-emoji` | Emoji display | `font-size: 24px`, `filter: drop-shadow(0 1px 2px rgba(0,0,0,0.5))` |

---

## 7. CSS Design System

### 7.1 Custom Properties

```css
:root {
  --header-h: 48px;
  --sidebar-w: 280px;
  --panel-w: 340px;
  --bg: #0f1117;
  --surface: #1a1d27;
  --surface2: #232733;
  --border: #2a2e3a;
  --text: #e1e4eb;
  --text2: #8b90a0;
  --accent: #4f8cff;
  --accent2: #6c5ce7;
  --green: #00b894;
}
```

### 7.2 Dark Theme (Default)

| Variable | Value | Usage |
|---|---|---|
| `--bg` | `#0f1117` | Page background |
| `--surface` | `#1a1d27` | Cards, sidebar, panels |
| `--surface2` | `#232733` | Secondary surfaces, hover states |
| `--border` | `#2a2e3a` | Borders, dividers |
| `--text` | `#e1e4eb` | Primary text |
| `--text2` | `#8b90a0` | Muted/secondary text |

### 7.3 Light Theme (`.light` on `<body>`)

| Variable | Value | Usage |
|---|---|---|
| `--bg` | `#f0f2f5` | Page background |
| `--surface` | `#ffffff` | Cards, sidebar, panels |
| `--surface2` | `#e8eaef` | Secondary surfaces, hover states |
| `--border` | `#d1d5db` | Borders, dividers |
| `--text` | `#1a1d27` | Primary text |
| `--text2` | `#6b7280` | Muted/secondary text |

### 7.4 Key Design Patterns

| Pattern | Implementation |
|---|---|
| **Layout** | Full-viewport flexbox column (`.app` `height: 100vh`) with header + body flex row |
| **Sidebar** | Fixed 280px, scrollable `.file-list` with `overflow-y: auto`, tabbed navigation |
| **Content area** | `.center` flex column with tab bar + `.content` (`position: relative`, flex: 1) |
| **Details panel** | Absolute-positioned, slides via `right: calc(-1 * 340px)` → `right: 0`, 0.3s cubic-bezier |
| **Modals** | Fixed overlay `rgba(0,0,0,0.6)`, centered `.modal` card (400–680px), 12px radius |
| **Scrollbars** | Custom 6px Webkit scrollbar: transparent track, `var(--border)` thumb |
| **Hover reveals** | File item action buttons start at `opacity: 0`, transition to `opacity: 1` on `.file-item:hover` |
| **Segmented toggle** | Toggle groups with `overflow: hidden` + `border-radius: 8px` for border collapse |
| **Icon grids** | CSS Grid: `repeat(5, 1fr)` in modal dialogs, `repeat(6, 1fr)` in details panel |
| **Sort arrows** | Inline `span.sort-arrow` with `.active` class for accent coloring |
| **Leaflet overrides** | `!important` on tooltip/popup backgrounds, borders to match theme |
| **Transition** | Blink uses CSS `transition: filter 1s ease-in-out` (sepia + blur → normal) |

---

## 8. Leaflet Integration

### 8.1 Map Setup

| Property | Value |
|---|---|
| **Element** | `<div id="map">` (flex: 1 in `.map-view`) |
| **Initialization** | Lazy: first `renderMap()` call creates `L.map('map', { zoomControl: false })` |
| **Default center** | `[39.8283, -98.5795]` (geographic center of USA) |
| **Default zoom** | 4 |
| **Zoom control** | `L.control.zoom({ position: 'bottomright' })` |

### 8.2 Tile Layers

| Theme | URL |
|---|---|
| **Dark** | `https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png` |
| **Light** | `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png` |
| **Swap method** | `tileLayer.setUrl(newUrl)` — no layer re-creation |

### 8.3 Layer Management

| Layer Type | Grouping | Lifecycle |
|---|---|---|
| Dataset points | `L.layerGroup` per dataset | Removed by `mapInstance.eachLayer()` before re-render |
| Dataset tracks | `L.polyline` + start/end markers | Removed by `mapInstance.eachLayer()` before re-render |
| Measure points | Individual `L.marker` with `_isMeasure = true` | Removed by `clearMeasurements()` or map cleanup |
| Measure lines | Individual `L.polyline` with `_isMeasure = true` | Removed by `clearMeasurements()` or map cleanup |
| Measure labels | Individual `L.marker` with `_isMeasure = true` | Removed by `clearMeasurements()` or map cleanup |
| Dataset markers | Stored in `datasetMarkers[fileId]` array | References kept for `blinkDataset()` |

### 8.4 Point Rendering

| Type | Leaflet Component | When Used |
|---|---|---|
| **Circle marker** | `L.circleMarker([lat, lng], { radius: 7, color: datasetColor, fillColor: datasetColor, fillOpacity: 0.8, weight: 2 })` | Default — no icon override |
| **Emoji icon** | `L.marker([lat, lng], { icon: L.divIcon({ className: 'marker-icon', html: '<div class="marker-emoji">EMOJI</div>', iconSize: [30, 30], iconAnchor: [15, 15] }) })` | When `file.icon` or `row._icon` is set |
| **Tooltip (label)** | `marker.bindTooltip(labelText, { permanent: true, direction: 'top', offset: [0, -14 or -18], className: 'point-label' })` | When `file.showLabels` or `row._showLabel` is true |
| **Popup** | `marker.bindPopup(html)` — all non-lat/lng keys as `<b>key:</b> value` | On click |
| **Click handler** | `marker.on('click', () => openDetails(row))` | Always attached |

### 8.5 Track Rendering

| Element | Component | Style |
|---|---|---|
| **Polyline** | `L.polyline(coords, options)` | `color: datasetColor`, `weight: 3`, `opacity: 0.8`, `smoothFactor: 1` |
| **Start marker** | `L.circleMarker(firstCoord, options)` | `radius: 8`, `fillColor: '#00b894'` (green), `color: '#fff'` |
| **End marker** | `L.circleMarker(lastCoord, options)` | `radius: 8`, `fillColor: '#e17055'` (red), `color: '#fff'` |

### 8.6 Measurement Tool

| Element | Component | Style |
|---|---|---|
| **Waypoints** | `L.marker` with divIcon | Gold circle (14px), `border: 3px solid #fdcb6e`, draggable |
| **Drag state** | Class `.measure-point-icon-dragging` | Bigger (20px), orange tint |
| **Segments** | `L.polyline` | `color: '#fdcb6e'`, `weight: 2`, `dashArray: '6, 8'` |
| **Distance labels** | `L.marker` at segment midpoint | Gold border, `background: #2d2d2d`, `color: #fdcb6e` |
| **Total display** | HTML `#totalDistVal` | Right-aligned, gold text |
| **Distance calc** | `L.latLng(a).distanceTo(b)` | Leaflet's built-in Haversine |

### 8.7 Fit Bounds

```javascript
mapInstance.fitBounds(L.latLngBounds(bounds), { padding: [30, 30] });
// bounds = all coords from focus file, or all visible files' coords
```

### 8.8 Blink Animation

```javascript
// CSS transition on the marker element
transition: filter 1s ease-in-out;

// Apply blink
marker._icon.style.filter = 'sepia(1) blur(0.5px) hue-rotate(-30deg)';
// After 1s, remove — CSS transitions back over 1s
setTimeout(() => { marker._icon.style.filter = ''; }, 1000);
```

Each of the dataset's markers blinks sequentially with 100ms stagger between them for a cascading wave effect.

---

## 9. Scenario Flows

### 9.1 Import a CSV/XLSX File

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks `+ Import` button | Hidden `<input #fileInput>` clicked programmatically | `importFile()` |
| 2 | Selects `.xlsx` / `.xls` / `.csv` | File begins reading via `FileReader` | `handleFileImport(event)` |
| 3 | — | SheetJS parses data: `XLSX.read()` for xlsx, `TextDecoder + XLSX.read()` for csv | SheetJS API |
| 4 | — | Import dialog appears: column mapping selects, preview table, row count | `showImportDialog()` |
| 5 | — | Lat/lng columns auto-detected, coordinate format auto-detected | `getLatLngColumns()`, `detectCoordFormat()` |
| 6 | Adjusts lat/lng columns or format (optional) | Preview updates | `updateImportFormat()` |
| 7 | Clicks `Import` | Coordinates parsed, new `FILES` entry created, all views re-rendered | `confirmImport()` |
| 8 | — | Import dialog closes; data visible in table, map, and sidebar | Render chain |
| 9 | Clicks `Cancel` or `×` | `pendingImport` cleared, modal dismissed | `cancelImport()` |

### 9.2 Load a Built-in POI Set

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks `Built-in` sidebar tab | Built-in list shown | `switchSidebarTab('builtin')` |
| 2 | Clicks e.g. "Embassies in Poland" | Item shows "loading..." state, disabled style | `loadBuiltinSet('poi_pl_embassies')` |
| 3 | — | Synthetic data generated (70 Warsaw embassy points) | `generatePolandEmbassiesData()` |
| 4 | — | All existing files deactivated | Loop: `f.active = false` |
| 5 | — | New `FILES` entry created (type: `'csv'`, cols: 5, auto-icon) | File constructor |
| 6 | — | Sidebar switches to "My Sets" tab | `switchSidebarTab('my')` |
| 7 | — | Built-in list updated with "70 points loaded" | `renderBuiltinSets()` |
| 8 | — | Full re-render: sidebar, table, map, counts | Render chain |
| 9 | — | Map centers on Warsaw, markers blink wave | `renderMap()`, `blinkDataset()` |

### 9.3 Explore Points on Map

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks a dataset in sidebar | Active file set, map zooms to bounds | `selectFile(id)` → `renderMap(id)` |
| 2 | Hovers over point marker | Tooltip appears (if labels enabled for dataset or point) | Leaflet `bindTooltip()` |
| 3 | Clicks point marker | Details panel slides in from right edge | `openDetails(row)` |
| 4 | — | Panel shows point title, toolbar (Label + Icon), key/value rows | Panel render |
| 5 | Clicks `🎙 Label` button | `row._showLabel` toggled, panel + map re-rendered | `togglePointLabel()` |
| 6 | Clicks `🎨 Icon` button | Inline emoji grid shown/hidden in panel | `togglePointIcon()` |
| 7 | Selects an emoji from grid | `row._icon` set, panel + map re-rendered with new icon | `selectPointIcon('🏛')` |
| 8 | Clicks `×` close button | Panel slides away, `currentDetailRow` cleared | `closeDetails()` |

### 9.4 Sort and Filter Table

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks "Revenue" column header | Table sorted ascending by revenue; ▲ shown | `sortBy('revenue')` |
| 2 | Clicks "Revenue" again | Sort direction toggles to descending; ▼ shown | `sortBy('revenue')` |
| 3 | Types "Paris" in global search | Table filters to rows where any column matches "Paris"; "X of Y" updated | `onFilterChange()` |
| 4 | Types ">1000" in Revenue column filter | Per-column filter applied, combined with global; further narrowed | `onFilterChange()` |
| 5 | Clicks `×` clear button | Global search cleared, per-column filters remain | `clearGlobalFilter()` |
| 6 | Clicks a table row | Details panel opens, map centers on point at zoom 10 | `selectTableRow(idx)` |

### 9.5 Use Measurement Tool

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks measure button in map toolbar | Button active (`.active` class), cursor changes to crosshair | `toggleMeasure()` |
| 2 | Clicks a location on map | Gold circle placed; clear button appears | `addMeasurePoint(e)` |
| 3 | Clicks second location | Dashed line drawn between points; distance label at midpoint | `renderMeasurements()` |
| 4 | Clicks more points | Accumulating polyline with segment distances; total distance updated | `renderMeasurements()` |
| 5 | Drags a waypoint | Line + all distances update in real-time as marker moves | Drag handlers |
| 6 | Clicks `× Clear` | All measurements removed, UI reset | `clearMeasurements()` |

### 9.6 Merge Two Datasets

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks `Merge` button in sidebar header | Merge dialog opens with checkbox list of all datasets | `showMergeDialog()` |
| 2 | Checks ≥2 datasets | Selection state tracked | Checkbox change |
| 3 | Clicks `Merge` | Validates ≥2 checked; union of all keys computed | `confirmMerge()` |
| 4 | — | For each row: missing keys filled with `''` | Data merge loop |
| 5 | — | New `FILES` entry created with name "Merged Dataset (N files)" | File constructor |
| 6 | — | All original files deactivated; new file active | State update |
| 7 | — | Sidebar, table, map, counts re-rendered | Render chain |
| 8 | — | Merge dialog closes | Modal close |

### 9.7 Toggle Dark/Light Theme

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks 🌙/☀️ toggle in header | Checkbox state toggles | `toggleTheme()` |
| 2 | — | `.light` class toggled on `<body>` | `classList.toggle('light')` |
| 3 | — | All CSS custom properties swap via cascade | CSS engine |
| 4 | — | Map tile layer URL swapped: `dark_all` ↔ `light_all` | `tileLayer.setUrl()` |
| 5 | — | Full visual theme transition | CSS / tile load |

### 9.8 Customize Dataset Icon

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks 🎨 on a file item (sidebar) | Icon picker modal opens with 20 emoji options | `showIconPicker(fileId)` |
| 2 | Hovers over emoji options | Visual hover highlight on each emoji cell | CSS `:hover` |
| 3 | Selects an emoji | `file.icon = selectedEmoji`, modal closes | `selectIcon('🏦')` |
| 4 | — | Sidebar re-rendered (icon shown in file item) | `renderSidebar()` |
| 5 | — | Map re-rendered: all points now use emoji markers | `renderMap()` |
| 6 | Clicks "Clear icon" (or `✕` option) | `file.icon = null`, reverts to default circle markers | `selectIcon(null)` |

### 9.9 Rename a Dataset

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks ✎ on a file item | Rename dialog opens with current name in input (auto-focused + selected) | `showRenameDialog(id)` |
| 2 | Types new name | Input value changes | — |
| 3 | Presses Enter or clicks `Rename` | Validates non-empty, updates `file.name`, re-renders sidebar + table | `confirmRename()` |
| 4 | Clicks `Cancel` or `×` | Input discarded, state cleared | `cancelRename()` |

### 9.10 Remove a Dataset

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks `×` on a file item | Remove confirmation dialog opens with file name | `showRemoveDialog(id)` |
| 2 | Clicks `Remove` (red button) | File removed from `FILES` array | `confirmRemove()` |
| 3 | — | If active file removed: nearest neighbor selected (or null) | State cleanup |
| 4 | — | Sidebar, table, map, counts re-rendered | Render chain |
| 5 | Clicks `Cancel` or `×` | `removeTargetId` cleared, dialog dismissed | `cancelRemove()` |

### 9.11 Customize Per-Point Label (🔜 Planned)

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Opens details panel for a point | Label text input shown in toolbar area | `openDetails(row)` |
| 2 | Current label text displayed (auto-extracted or `_label`) | Input pre-filled | `getLabelText(row)` |
| 3 | Types new label text | Input updates | `onLabelInput()` |
| 4 | — | Map marker label updates in real-time | `reapplyLabel()` |
| 5 | Label persists when re-opening panel | `row._label` stored on row object | State persistence |

### 9.12 Add New Point via Map Click (🔜 Planned)

| # | Actor Action | System Response | Function Called |
|---|---|---|---|
| 1 | Clicks `+ Point` in sidebar header | Add Point dialog opens | `showAddPointDialog()` |
| 2 | — | Dynamic form fields generated from active dataset schema | Form generation |
| 3 | Clicks 📍 button next to lat field | Map enters crosshair selection mode, cursor changes | `enableMapPick()` |
| 4 | Clicks a location on map | Lat + lng fields auto-populated, picker mode ends | `onMapPick(e)` |
| 5 | Fills remaining fields (optional) | Form data ready | User input |
| 6 | Clicks `Add Point` | New row appended at end of `file.data` | `confirmAddPoint()` |
| 7 | — | Table, map, sidebar re-rendered | Render chain |
| 8 | — | New point visible on map with dataset's icon/color | `renderMap()` |

---

## 10. Implementation Status

### 10.1 Legend

| Icon | Meaning |
|---|---|
| ✅ | Completed and tested |
| 🔜 | Planned, approved, not yet implemented |
| ❌ | Not planned / future consideration |

### 10.2 Feature Completeness Matrix

| Category | Feature | ID | Status |
|---|---|---|---|
| **Data** | Import CSV/XLSX | F1 | ✅ |
| **Data** | Coordinate auto-detection | F2 | ✅ |
| **Data** | Multi-format coordinate parsing | F3 | ✅ |
| **Data** | Merge datasets | F4 | ✅ |
| **Data** | Remove dataset | F5 | ✅ |
| **Data** | Rename dataset | F6 | ✅ |
| **Data** | Dataset visibility toggle | F7 | ✅ |
| **Data** | Built-in POI sets | F8 | ✅ |
| **Data** | Add new point | F9 | 🔜 |
| **Data** | Dataset-level icon | F10 | ✅ |
| **Map** | Point markers | M1 | ✅ |
| **Map** | Per-point icon override | M2 | ✅ |
| **Map** | Track mode | M3 | ✅ |
| **Map** | Combined mode | M4 | ✅ |
| **Map** | Dataset labels | M5 | ✅ |
| **Map** | Per-point labels | M6 | ✅ |
| **Map** | Auto label extraction | M7 | ✅ |
| **Map** | Per-point custom label | M8 | 🔜 |
| **Map** | Map click → details | M9 | ✅ |
| **Map** | Zoom to dataset | M10 | ✅ |
| **Map** | Blink animation | M11 | ✅ |
| **Map** | Measurement tool | M12 | ✅ |
| **Map** | Theme-aware tiles | M13 | ✅ |
| **Map** | Map click coordinate pick | M14 | 🔜 |
| **Map** | Popup on click | M15 | ✅ |
| **Table** | Column sorting | T1 | ✅ |
| **Table** | Global text search | T2 | ✅ |
| **Table** | Per-column filters | T3 | ✅ |
| **Table** | Row selection → details | T4 | ✅ |
| **Table** | Row count display | T5 | ✅ |
| **Table** | Clear filter button | T6 | ✅ |
| **Details** | Slide-in panel | D1 | ✅ |
| **Details** | Key/value display | D2 | ✅ |
| **Details** | Per-point label toggle | D3 | ✅ |
| **Details** | Per-point icon picker | D4 | ✅ |
| **Details** | Per-point label editing | D5 | 🔜 |
| **UX** | Dark/Light theme | U1 | ✅ |
| **UX** | Main tab switching | U2 | ✅ |
| **UX** | Total record counter | U3 | ✅ |
| **UX** | Sidebar tab system | U4 | ✅ |
| **UX** | Loading indicators | U5 | ✅ |
| **UX** | Empty states | U6 | ✅ |
| **UX** | Measurement distance | U7 | ✅ |

### 10.3 Upcoming Work (Approved)

| Feature | ID | Scope | Complexity |
|---|---|---|---|
| Per-point custom label editing | D5 / M8 | Add text input in details panel below toolbar; `getLabelText()` checks `row._label` first; map updates in real-time | **Low** — ~20 lines |
| Add new point with map click | F9 / M14 | `+ Point` button in sidebar header; modal with dynamic fields; map-click coordinate picker; confirm appends + re-renders | **Medium** — ~80 lines |

---

## Appendices

### A. Coordinate Parsing Details

**`parseCoord(value, format)`** supports three formats:

| Format | Example | Parsing Logic |
|---|---|---|
| **DD** (Decimal Degrees) | `40.7128` | `parseFloat()` |
| **DDM** (Degrees Decimal Minutes) | `40°42.768' N` | `deg + min/60`, apply sign from N/S/E/W |
| **DMS** (Degrees Minutes Seconds) | `40°42'46" N` | `deg + min/60 + sec/3600`, apply sign from N/S/E/W |

**`detectCoordFormat(values)`** heuristic:
1. Check if values contain `°` (degree symbol)
2. If yes: check for `"` (double prime) → `'DMS'`; check for `'` (prime) → `'DDM'`
3. Otherwise → `'DD'`

### B. Color Palette (DATASET_COLORS)

```javascript
['#4f8cff', '#6c5ce7', '#00b894', '#fdcb6e', '#e17055',
 '#00cec9', '#a29bfe', '#fd79a8', '#74b9ff', '#55efc4']
```

Cycled through modulo `i % 10` for up to 10 datasets before repeating.

### C. Icon Picker Options (ICON_PICKER_OPTIONS)

```javascript
['📍', '🏛', '🏢', '🏣', '🏦', '🏨', '🏪', '🏫',
 '🏭', '🏯', '🗽', '⛪', '🕌', '🕍', '⛩', '🎯',
 '⚡', '🔥', '💧', '⭐']
```

20 emoji choices in both dataset-level (modal, 5-column grid) and per-point (inline, 6-column grid) pickers.

### D. Built-in Dataset Generators — Coverage

| Generator | Cities | Range | Jitter | Dedup |
|---|---|---|---|---|
| `generateEmbassiesData()` | 18 capitals | 2–6 per city | ±0.02° | No (countries unique per city) |
| `generateBanksData()` | 10 hubs | 3–7 per city | ±0.015° | Yes (Set-based) |
| `generateGovernmentData()` | 12 capitals | 3–6 per city | ±0.0125° | Yes (Set-based) |
| `generatePolandEmbassiesData()` | 1 (Warsaw) | 70 total | 0.01–0.03° radial | No (all unique countries) |

---

> **Document generated:** May 2026  
> **Source:** `index.html` (~2243 lines)  
> **Next planned features:** Per-point label editing, Add new point with map-click coordinate picker
