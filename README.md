# AG Grid Custom Type System — Cheat Sheet

## TL;DR — Storing Types in DB

**Yes**, you can store plain string names in a colDef (in the DB) and attach all behavior at runtime. There are **two complementary mechanisms** for this, and they serve slightly different purposes. You'll likely want to use both together.

---

## The Two Name-Based Systems

### 1. `columnTypes` — Column Property Bundles

Stores reusable **groups of colDef properties** under a name. A column opts in via `type: 'myTypeName'`.

```ts
// Grid option — defined once, never stored in DB
const columnTypes = {
  currency: {
    width: 150,
    editable: true,
    valueFormatter: (p) => `£${p.value}`,
    valueParser: (p) => Number(p.newValue),
    cellEditor: 'agNumberCellEditor',
    filter: 'agNumberColumnFilter',
  },
  status: {
    cellRenderer: 'statusRenderer',  // string name — safe to use here too
    cellEditor: 'statusEditor',
    editable: true,
  }
}

// ColDef — this IS what you store in the DB
{ field: 'price', type: 'currency' }
{ field: 'price', type: ['currency', 'shaded'] }  // multiple types
```

**What `ColTypeDef` can contain:** Everything a `ColDef` can, *except* `type` and `cellDataType` themselves.

---

### 2. `dataTypeDefinitions` — Value Lifecycle Definitions

Defines how a **data type** flows through the whole value lifecycle. Referenced by `cellDataType: 'myTypeName'` in colDef.

```ts
// Grid option — defined once, never stored in DB
const dataTypeDefinitions = {
  money: {
    baseDataType: 'number',      // required — one of the built-in base types
    extendsDataType: 'number',   // inherit defaults from this type
    valueFormatter: (p) => p.value == null ? '' : `£${p.value.toFixed(2)}`,
    valueParser: (p) => Number(p.newValue),
    dataTypeMatcher: (v) => typeof v === 'number',  // for auto-inference
    columnTypes: 'moneyColumn',  // can link to a columnType!
  }
}

// ColDef — stored in DB
{ field: 'price', cellDataType: 'money' }
```

---

### 3. `components` — Register Components by String Name

Lets you reference React components by a plain string in colDef (safe to store in DB).

```ts
// Grid option
const components = {
  statusRenderer: StatusCellRenderer,
  statusEditor: StatusCellEditor,
  ratingRenderer: RatingCellRenderer,
}

// ColDef — stored in DB
{ field: 'status', cellRenderer: 'statusRenderer', cellEditor: 'statusEditor' }
```

---

## Full Value Lifecycle Map

```
DB row data
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ READING/DISPLAYING                                          │
│                                                             │
│  field / valueGetter  ──► raw value                        │
│       │                        │                           │
│       │              valueFormatter ──► display string     │
│       │                        │        (shown in cell     │
│       │                        │         when not editing) │
│       │                        │                           │
│       │              cellRenderer ──► custom JSX           │
│       │                             (replaces text,        │
│       │                              use for rich UI)      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ EDITING                                                     │
│                                                             │
│  editable: true  ──► enables editing                       │
│                                                             │
│  cellEditor ──► which editor component to use              │
│    • 'agTextCellEditor'         (default)                  │
│    • 'agNumberCellEditor'                                   │
│    • 'agSelectCellEditor'                                   │
│    • 'agRichSelectCellEditor'   (Enterprise)               │
│    • 'agCheckboxCellEditor'                                 │
│    • 'agDateCellEditor'                                     │
│    • 'myCustomEditor'           (your registered component)│
│                                                             │
│  cellEditorParams ──► extra params passed to editor        │
│  cellEditorPopup ──► show editor in popup overlay          │
│  cellEditorPopupPosition ──► 'over' | 'under'              │
│                                                             │
│  Editor receives: value (current), onValueChange(newVal)   │
│  Editor calls onValueChange() as user types                │
└─────────────────────────────────────────────────────────────┘
    │
    ▼ (on edit stop)
┌─────────────────────────────────────────────────────────────┐
│ SAVING BACK TO DATA                                         │
│                                                             │
│  valueParser ──► converts editor string → typed value      │
│    (e.g. "42" → 42, "2024-01-01" → Date object)           │
│    params: { newValue: string, oldValue, data, ... }       │
│    returns: the parsed value to store in row data          │
│                                                             │
│  valueSetter ──► writes parsed value into row data         │
│    (use when field path is complex / computed)             │
│    params: { newValue, oldValue, data, ... }               │
│    returns: true if data changed (triggers cell refresh)   │
│                                                             │
│  If no valueSetter: grid writes to data[field] directly    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ EVENTS                                                      │
│  onCellValueChanged ──► fires after value saved to data    │
│    params.newValue, params.oldValue, params.data           │
│    ► This is where you persist to DB                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Property Responsibility Table

| Concern | Property | Where defined | Stored in DB? |
|---|---|---|---|
| Which field in row object | `field` | colDef | ✅ yes |
| Custom value read logic | `valueGetter` | colDef or columnType | ❌ no (function) |
| Display formatting | `valueFormatter` | colDef or columnType or dataTypeDef | ❌ no (function) |
| Rich cell display | `cellRenderer` | colDef (as string name) | ✅ yes (string) |
| Renderer params | `cellRendererParams` | colDef | ✅ yes (plain obj) |
| Which editor | `cellEditor` | colDef (as string name) | ✅ yes (string) |
| Editor params | `cellEditorParams` | colDef | ✅ yes (plain obj) |
| Editor as popup | `cellEditorPopup` | colDef | ✅ yes (bool) |
| String → typed value | `valueParser` | colDef or columnType or dataTypeDef | ❌ no (function) |
| Write value back | `valueSetter` | colDef or columnType | ❌ no (function) |
| Enable editing | `editable` | colDef or columnType | ✅ yes (bool) |
| Type bundle name | `type` | colDef | ✅ yes (string) |
| Data type name | `cellDataType` | colDef | ✅ yes (string) |
| Filter type | `filter` | colDef or columnType | ✅ yes (string) |

---

## Recommended Pattern for DB-Stored Grids

**What to store in DB (colDef):**
```json
{
  "field": "price",
  "headerName": "Price",
  "type": "currency",
  "cellDataType": "money",
  "editable": true,
  "cellEditorParams": { "min": 0 }
}
```

**What to define in app code (never in DB):**
```ts
// columnTypes — bundles of colDef properties
const columnTypes = {
  currency: {
    cellRenderer: 'currencyRenderer',
    cellEditor: 'agNumberCellEditor',
    filter: 'agNumberColumnFilter',
    valueFormatter: (p) => formatCurrency(p.value),
    valueParser: (p) => parseFloat(p.newValue),
  }
}

// dataTypeDefinitions — value lifecycle per type
const dataTypeDefinitions = {
  money: {
    baseDataType: 'number',
    extendsDataType: 'number',
    valueParser: (p) => parseFloat(p.newValue),
    valueFormatter: (p) => formatMoney(p.value),
  }
}

// components — string → component mapping
const components = {
  currencyRenderer: CurrencyRenderer,
  statusEditor: StatusEditor,
}
```

**Grid setup:**
```tsx
<AgGridReact
  columnDefs={colDefsFromDB}       // loaded from DB, plain JSON
  columnTypes={columnTypes}         // defined in code
  dataTypeDefinitions={dataTypeDefinitions}
  components={components}
  onCellValueChanged={persistToDb}  // ← save edits back to DB
/>
```

---

## Key Distinctions: `columnTypes` vs `dataTypeDefinitions`

| | `columnTypes` | `dataTypeDefinitions` |
|---|---|---|
| Referenced by | `colDef.type` | `colDef.cellDataType` |
| What it is | Bundle of colDef props | Value lifecycle + type semantics |
| Can contain | Any colDef prop (except `type`/`cellDataType`) | `valueFormatter`, `valueParser`, `dateParser`, `dateFormatter`, `dataTypeMatcher`, `columnTypes` |
| Auto-configures filter? | No | Yes (inherits from baseDataType) |
| Auto-configures editor? | Only if you add `cellEditor` | Yes (inherits from baseDataType) |
| Inheritance | No | Yes via `extendsDataType` |
| Multiple per column? | Yes (`type: ['a','b']`) | No (one per column) |

---

## `dataTypeDefinitions` — What Each Property Does

```ts
{
  baseDataType: 'number',     // REQUIRED. Underlying JS type for filtering/sorting
  extendsDataType: 'number',  // Inherit defaults from this type (usually same as base)

  valueFormatter: (p) => string,    // raw value → display string
  valueParser: (p) => typedValue,   // editor string → value to store

  // Only for dateString / dateTimeString base types:
  dateParser: (str) => Date,        // string stored in data → Date (for filter/sort)
  dateFormatter: (date) => string,  // Date → string to store back

  dataTypeMatcher: (v) => boolean,  // return true if this value belongs to this type
                                    // enables auto-inference (cellDataType: true)

  columnTypes: 'myColumnType',      // link to a columnType for additional colDef props
  appendColumnTypes: false,         // true = append to parent's types instead of replace
  suppressDefaultProperties: false, // true = don't inherit parent's colDef properties
}
```

---

## Built-in Editor String Names

| String | Description |
|---|---|
| `'agTextCellEditor'` | Plain text input (default) |
| `'agNumberCellEditor'` | Number input |
| `'agCheckboxCellEditor'` | Checkbox |
| `'agDateCellEditor'` | Date picker (Date value) |
| `'agDateStringCellEditor'` | Date picker (string value) |
| `'agSelectCellEditor'` | HTML `<select>` |
| `'agLargeTextCellEditor'` | Textarea |
| `'agRichSelectCellEditor'` | Rich dropdown (Enterprise) |

---

## Built-in Renderer String Names

| String | Description |
|---|---|
| `'agGroupCellRenderer'` | Row group expand/collapse |
| `'agCheckboxCellRenderer'` | Boolean as checkbox |
| `'agAnimateShowChangeCellRenderer'` | Animates value changes |
| `'agAnimateSlideCellRenderer'` | Slides value changes |
| `'agSparklineCellRenderer'` | Sparkline chart (Enterprise) |

---

## Priority Order (highest wins)

```
colDef property
  > colDef.type[0] > colDef.type[1] > ... (first in array wins)
    > defaultColDef
      > dataTypeDefinitions (for value lifecycle props only)
```

---

## AG Grid Version

This cheat sheet applies to **AG Grid v35.1.0** (React, Enterprise).
