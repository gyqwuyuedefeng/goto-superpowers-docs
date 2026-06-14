# LineParam Compact Toolbar Design

## Context

Backend class `goto/common/src/main/java/com/freedom/model/form/LineParam.java` defines line parameters for drawing forms. The active frontend component is `goto-web/src/views/goto/components/form/draw/lineParam.vue`.

The current frontend layout renders line settings as a boxed vertical list with visible text labels for each field. The target style should follow the compact, icon-first pattern used by `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`.

## Scope

Only these parameters are in scope:

- `color`
- `lineType`
- `lineWidth`
- `length`

`lineEnd` remains in the backend model and should not be deleted, but it should not be added to this UI pass.

## Goal

`lineParam.vue` should show the main `formLabel` on its own line, then render the four active line controls in a compact row. The row should prefer one-line display, but controls may wrap when the container is too narrow.

Each compact control should preserve a usable minimum width so the UI wraps by control block instead of compressing inputs until they become unreadable.

## Recommended Approach

Use a single compact flex row under the main label:

- Keep `formLabel` as the section title on a separate line.
- Keep using existing data bindings and child controls where practical: `SettingForm`, `ColorPicker`, `GSelectDictImage`, and `GInput`.
- Wrap each field in an `el-tooltip` with the title text:
  - `颜色`
  - `线型`
  - `宽度`
  - `长度`
- Use icon or preview-first UI:
  - Color: color picker entry or swatch-like compact wrapper.
  - Line type: current SVG line preview from `GSelectDictImage`.
  - Width: compact numeric input with a width icon.
  - Length: compact numeric input with a length icon.

The component should use scoped CSS to avoid changing shared field components globally.

## Layout Rules

The controls should use `display: flex`, `align-items: center`, and `flex-wrap: wrap`.

Recommended minimum widths:

- Color: about `36px`.
- Line type: about `56px`.
- Width: about `86px` to `96px`.
- Length: about `86px` to `96px`.

The exact values may be tuned during implementation based on the actual DOM of `ColorPicker`, `GSelectDictImage`, and `GInput`, but the intent is fixed: maintain readable controls and wrap cleanly when needed.

## Data Flow

No backend or API changes are required.

The component should continue to emit `modify` through the existing `handleModify(fieldName, newValue, oldValue, parent)` path, preserving `modifyParams`.

The four fields continue to bind to:

- `setting.color.value`
- `setting.lineType.value.value`
- `setting.lineWidth.value`
- `setting.length.value`

The min/max helper fields remain used for numeric controls:

- `setting.lineWidthMin.value`
- `setting.lineWidthMax.value`
- `setting.lengthMin.value`
- `setting.lengthMax.value`

## Error Handling

No new async behavior or recoverable errors are introduced.

The UI should tolerate field visibility through the existing `SettingForm :container` behavior. If a field is hidden by `container.visible`, its compact block should not occupy layout space.

## Testing

Manual verification is sufficient for this focused UI change:

1. Open a draw form that renders `GLineParam`.
2. Confirm `formLabel` appears on its own line.
3. Confirm `颜色`, `线型`, `宽度`, and `长度` render as compact controls in one row when space allows.
4. Confirm each control shows the correct `el-tooltip` title on hover.
5. Narrow the side panel or viewport and confirm controls wrap cleanly without overlapping or unreadable input text.
6. Change each of the four values and confirm the existing modify behavior still updates downstream state.
7. Confirm `lineEnd` is not rendered.
