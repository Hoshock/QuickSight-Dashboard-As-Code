# Sheets and Layouts

A sheet is one "page" in the analysis. It holds visuals, filter/parameter controls, text boxes, images — positioned by exactly one layout.

## SheetDefinition

```json
{
  "SheetId":               "string",
  "Name":                  "string",
  "Title":                 "string",
  "Description":           "string",
  "ContentType":           "INTERACTIVE" | "PAGINATED",
  "FilterControls":        [ ... ],
  "Layouts":               [ { "Configuration": { ... } } ],
  "ParameterControls":     [ ... ],
  "SheetControlLayouts":   [ ... ],
  "TextBoxes":             [ ... ],
  "Visuals":               [ ... ],
  "CustomActionDefaults":  { ... },
  "Images":                [ ... ]
}
```

| Field                 | Type                   | Required | Limit                                                                    |
| --------------------- | ---------------------- | -------- | ------------------------------------------------------------------------ |
| `SheetId`             | String                 | ✅       | 1–512 chars, pattern `[\w\-]+`                                           |
| `Name`                | String                 | ❌       | 1–2048 chars                                                             |
| `Title`               | String                 | ❌       | 1–1024 chars                                                             |
| `Description`         | String                 | ❌       | 1–1024 chars                                                             |
| `ContentType`         | String                 | ❌       | `INTERACTIVE` (dashboard) or `PAGINATED` (report). Default `INTERACTIVE` |
| `FilterControls`      | `FilterControl[]`      | ❌       | 0–200 items                                                              |
| `Layouts`             | `Layout[]`             | ❌       | **Exactly 1** item when present                                          |
| `ParameterControls`   | `ParameterControl[]`   | ❌       | 0–200 items                                                              |
| `SheetControlLayouts` | `SheetControlLayout[]` | ❌       | 0–1 item                                                                 |
| `TextBoxes`           | `SheetTextBox[]`       | ❌       | 0–100 items                                                              |
| `Visuals`             | `Visual[]`             | ❌       | 0–75 items. See [definition-visuals.md](./definition-visuals.md)         |
| `Images`              | `SheetImage[]`         | ❌       | 0–10 items                                                               |

`SheetId` is the only required field on paper, but a sheet with no layout + visuals renders as blank. Practical sheets have at least `Layouts` + `Visuals`.

### SheetControlLayouts

Positions filter/parameter controls in a dedicated control bar at the top of the sheet (separate from the main visual layout). Structure:

```json
{
  "Configuration": {
    "GridLayout": {
      "Elements": [
        {
          "ElementId": "fc-region",
          "ElementType": "FILTER_CONTROL",
          "ColumnIndex": 0,
          "ColumnSpan": 6,
          "RowIndex": 0,
          "RowSpan": 1
        }
      ]
    }
  }
}
```

Controls can be placed in EITHER `Layouts[0]` (inline with visuals) OR `SheetControlLayouts[0]` (dedicated bar), but not both for the same control.

> **⚠️ SheetControlLayouts uses a 12-column grid** (not 36 like the main visual `Layouts`). `ColumnIndex` range is 0–11, `ColumnSpan` max is 6. Exceeding these causes synchronous `InvalidParameterValueException`. Place controls in rows of 2 (span=6 each) or 3 (span=4 each).

### ContentType

- `INTERACTIVE` — standard dashboard sheet, with `GridLayout` or `FreeFormLayout`. End users click/hover/filter.
- `PAGINATED` — report sheet for paginated reports. Uses `SectionBasedLayout`. Designed for scheduled PDF/print export.

`ContentType` must match the `Layouts[0].Configuration` variant:

| ContentType   | Valid layout                     |
| ------------- | -------------------------------- |
| `INTERACTIVE` | `GridLayout` or `FreeFormLayout` |
| `PAGINATED`   | `SectionBasedLayout`             |

## Layout

Wraps a `LayoutConfiguration` (one-of three layout types).

```json
{
  "Configuration": {
    "GridLayout":        { ... },
    "FreeFormLayout":    { ... },
    "SectionBasedLayout": { ... }
  }
}
```

Only one of the three keys may be present. The corresponding visuals/elements must be referenced by ID within that layout's `Elements` (grid) / `Elements` (free-form) / `BodySections` + `HeaderSections` + `FooterSections` (section-based).

### GridLayout

Responsive tiled layout. Visuals snap to a 36-column grid (the effective column count is derived from the element's `ColumnIndex` + `ColumnSpan`; elements span integer columns and rows).

```json
{
  "GridLayout": {
    "Elements": [
      {
        "ElementId":   "visual-1",
        "ElementType": "VISUAL" | "FILTER_CONTROL" | "PARAMETER_CONTROL" | "TEXT_BOX" | "IMAGE",
        "ColumnIndex": 0,
        "ColumnSpan":  12,
        "RowIndex":    0,
        "RowSpan":     6
      }
    ],
    "CanvasSizeOptions": {
      "ScreenCanvasSizeOptions": {
        "ResizeOption":          "FIXED" | "RESPONSIVE",
        "OptimizedViewPortWidth": "1600px"
      }
    }
  }
}
```

- `Elements` — 0–430 items. Every visual / control / text box on the sheet must have exactly one entry here, keyed by `ElementId`.
- `ElementType` — determines which object on the sheet the `ElementId` refers to.
- `ColumnIndex` / `ColumnSpan` / `RowIndex` / `RowSpan` — integer grid positions. Elements can overlap (the last wins in render order).
- `ResizeOption` — `FIXED` keeps pixel dimensions; `RESPONSIVE` scales with the viewport.

### FreeFormLayout

Pixel-precise absolute positioning. Use when Grid is too rigid (overlapping visuals, non-snapping positions, artistic dashboards).

```json
{
  "FreeFormLayout": {
    "Elements": [
      {
        "ElementId":   "visual-1",
        "ElementType": "VISUAL",
        "XAxisLocation": "0px",
        "YAxisLocation": "0px",
        "Width":          "640px",
        "Height":         "400px",
        "Visibility":     "VISIBLE" | "HIDDEN",
        "RenderingRules": [ ... ]
      }
    ],
    "CanvasSizeOptions": {
      "ScreenCanvasSizeOptions": { "OptimizedViewPortWidth": "1600px" }
    }
  }
}
```

Positions and sizes are CSS length strings (`"640px"`). `RenderingRules` lets an element show/hide conditionally based on parameter values.

> **⚠️ `CanvasSizeOptions` is required.** Omitting it causes `InvalidParameterValueException: Provided layout doesn't match default layout configuration`.

### SectionBasedLayout

Paginated-report layout. Sections stack vertically and paginate based on paper size. **Only valid when `ContentType = PAGINATED`.**

```json
{
  "SectionBasedLayout": {
    "HeaderSections": [ { "SectionId": "hdr", "Layout": { "FreeFormLayout": { ... } }, "Style": { ... } } ],
    "BodySections":   [ { "SectionId": "body-1", "Content": { "Layout": { "FreeFormLayout": { ... } } }, "Style": { ... }, "PageBreakConfiguration": { ... } } ],
    "FooterSections": [ { "SectionId": "ftr", "Layout": { "FreeFormLayout": { ... } }, "Style": { ... } } ],
    "CanvasSizeOptions": {
      "PaperCanvasSizeOptions": {
        "PaperSize":        "US_LETTER" | "A0" | "A1" | "A2" | "A3" | "A4" | "A5" | "JIS_B4" | "JIS_B5" | "US_LEGAL" | "US_TABLOID_LEDGER" | "US_EXECUTIVE",
        "PaperOrientation": "PORTRAIT" | "LANDSCAPE",
        "PaperMargin":      { "Top": "0.5in", "Bottom": "0.5in", "Left": "0.5in", "Right": "0.5in" }
      }
    }
  }
}
```

Each section holds a `FreeFormLayout` internally. `BodySections` supports `PageBreakConfiguration` for forcing breaks before/after a section.

## FilterControl

A UI widget for a filter, rendered in the sheet (not to be confused with the filter itself, which lives in `AnalysisDefinition.FilterGroups`).

```json
{
  "DateTimePicker":     { "FilterControlId": "...", "Title": "...", "SourceFilterId": "...", "DisplayOptions": { ... } },
  "List":               { "FilterControlId": "...", "Title": "...", "SourceFilterId": "...", "Type": "MULTI_SELECT" | "SINGLE_SELECT", "DisplayOptions": { ... }, "SelectableValues": { ... }, "CascadingControlConfiguration": { ... } },
  "Dropdown":           { ... },
  "TextField":          { ... },
  "TextArea":           { ... },
  "Slider":             { ... },
  "RelativeDateTime":   { ... },
  "CrossSheet":         { "FilterControlId": "...", "SourceFilterId": "...", "CascadingControlConfiguration": { ... } }
}
```

Union type — exactly one variant per control. `SourceFilterId` points at a `FilterId` defined in `FilterGroups[].Filters[]`.

> **⚠️ CrossSheet is REQUIRED** when the filter's `FilterGroup.ScopeConfiguration` is `AllSheets` or spans multiple sheets. Using a sheet-local control type (e.g., `Dropdown`, `List`) with a multi-sheet filter causes `InvalidParameterValueException`. Use `{ "CrossSheet": { "FilterControlId": "...", "SourceFilterId": "..." } }` and define the control's appearance via `DefaultFilterControlConfiguration` on the filter itself.

## ParameterControl

A UI widget bound to a parameter (from `ParameterDeclarations`).

```json
{
  "DateTimePicker": { "ParameterControlId": "...", "Title": "...", "SourceParameterName": "...", "DisplayOptions": { ... } },
  "List":           { "ParameterControlId": "...", "Title": "...", "SourceParameterName": "...", "Type": "MULTI_SELECT" | "SINGLE_SELECT", "SelectableValues": { ... }, ... },
  "Dropdown":       { ... },
  "TextField":      { ... },
  "TextArea":       { ... },
  "Slider":         { ... }
}
```

`SourceParameterName` points at `ParameterDeclarations[].Name`.

## SheetTextBox

```json
{
  "SheetTextBoxId": "intro-text",
  "Content": "<xml-rich-text-content>"
}
```

`Content` requires QuickSight's internal XML rich-text format. The safest approach is to create a TextBox via the QuickSight console, then export via `DescribeAnalysisDefinition` to obtain the correct XML. Plain text and Markdown are not accepted.

Positioned via the sheet's `Layouts[0]` — `ElementId` in the layout must match `SheetTextBoxId`.

## SheetImage

Inline static image asset on a sheet.

```json
{
  "SheetImageId":       "header-logo",
  "ImageContentAltText": "Company logo",
  "Source": {
    "SheetImageStaticFileSource": { "StaticFileId": "my-logo" }
  },
  "Scaling":      { "ScalingType": "SCALE_TO_HEIGHT" | "SCALE_TO_WIDTH" | "SCALE_TO_CONTAINER" | "SCALE_NONE" },
  "Interactions": { "ImageMenuOption": { "AvailabilityStatus": "ENABLED" | "DISABLED" } },
  "Tooltip":      { "TooltipText": { "PlainText": "..." }, "Visibility": "VISIBLE" | "HIDDEN" },
  "Actions":      [ /* VisualCustomAction-shaped */ ]
}
```

`StaticFileId` refers to `AnalysisDefinition.StaticFiles[].ImageStaticFile.StaticFileId`.
