# LineParam Compact Toolbar Polish Design

## Context

The first compact toolbar implementation for `goto-web/src/views/goto/components/form/draw/lineParam.vue` is complete, but the rendered result has two issues:

- When an individual parameter such as `length` is hidden, the child input disappears but the outer compact field box remains visible.
- The width and length indicators use the same arrow-like SVG, making the controls visually unclear.

The current component wraps `SettingForm` inside `el-tooltip` and `.line-param-field`. Because `SettingForm` owns the `container.visible` check, hidden fields leave the outer wrapper behind.

## Scope

Only polish the existing compact toolbar behavior and appearance:

- Fix hidden parameter rendering for `color`, `lineType`, `lineWidth`, and `length`.
- Replace the width and length SVG icons with text badges: `W` for width and `L` for length.
- Make the color control render as a square swatch-style control.
- Make color, line type, width, and length controls share the same visual height.
- Reduce visual heaviness of compact field wrappers while keeping the one-row-first, wrap-when-needed layout.

`lineEnd` remains out of scope and should not be rendered.

## Goal

The line parameter toolbar should look cleaner and should not show empty shells for hidden parameters.

For example, if `setting.length.visible` is false, the full length control should disappear: no tooltip, no icon/badge, no border box, and no leftover spacing except normal flex gap between remaining controls.

## Recommended Approach

Move `SettingForm` to wrap the entire compact field:

```vue
<SettingForm :container="setting.length">
  <el-tooltip content="长度" placement="top">
    <div class="line-param-field line-param-field--number line-param-field--length">
      ...
    </div>
  </el-tooltip>
</SettingForm>
```

This keeps visibility logic in the existing shared component and avoids duplicating `v-if="setting.xxx.visible"` in `lineParam.vue`.

For numeric controls:

- Replace SVG icons with compact text badges.
- Use `W` for `lineWidth`.
- Use `L` for `length`.
- Keep the full Chinese titles in `el-tooltip`.

## Visual Rules

The toolbar should keep the existing structure:

- `formLabel` on its own line.
- Compact controls in a horizontal flex row.
- `flex-wrap: wrap` remains enabled.
- Minimum widths remain stable so inputs do not collapse.

The field chrome should be lighter than the current screenshot:

- Keep a subtle border to show clickable/editable bounds.
- Avoid overly wide empty-looking boxes.
- Use one shared control height for all four fields, such as `36px`.
- Color should be a true square control: width and height should match.
- Line type should use the same height as color, width, and length, with the line preview centered.
- Numeric value should be visually stronger than the `W` or `L` badge.
- `W` and `L` should use a small, fixed-width badge area with secondary text color.

Approved preview sizing intent:

- Color: square, about `36px` by `36px`.
- Line type outer control: about `70px` wide by the shared control height.
- Line type inner preview: about `20px` high, `4px` border radius, and close to the outer shell with about `5px` horizontal padding.
- Width and length: about `96px` to `104px` wide by the shared control height.

The implementation may tune exact widths based on the child component DOM, but equal height is a fixed requirement.

## Data Flow

No backend or API changes are required.

The existing `handleModify(fieldName, newValue, oldValue, parent)` emit path remains unchanged.

The same fields remain bound:

- `setting.color.value`
- `setting.lineType.value.value`
- `setting.lineWidth.value`
- `setting.length.value`

## Error Handling

No new async behavior is introduced.

If a field container is missing or has `visible: false`, existing `SettingForm` behavior decides whether that whole compact block is rendered.

## Testing

Update the existing static regression test for `lineParam.vue`:

1. Assert `SettingForm` wraps each tooltip-backed compact field, so hidden fields remove the whole field wrapper.
2. Assert the old structure where `.line-param-field` wraps `SettingForm` is not used.
3. Assert width uses text badge `W`.
4. Assert length uses text badge `L`.
5. Assert the shared arrow SVG is no longer used for both width and length.
6. Assert a shared control height variable or rule is used by all compact fields.
7. Assert the color field is square.
8. Assert the line type preview uses the approved tighter inner spacing and smaller preview radius.
9. Keep existing assertions for tooltip titles, compact classes, wrapping, and `lineEnd` exclusion.

Manual verification:

1. Open a draw form that renders `GLineParam`.
2. Confirm width shows `W` and length shows `L`.
3. Confirm tooltip still shows `宽度` and `长度`.
4. Confirm hiding `length` removes the whole length field box.
5. Confirm color appears as a square swatch-style control.
6. Confirm color, line type, width, and length have the same visual height.
7. Confirm the line type preview has little empty space between the white preview and the outer shell, and the white preview is not overly pill-shaped.
8. Confirm the toolbar still wraps cleanly when narrow.
