# LineParam Compact Toolbar Fix Design

## Context

The completed polish implementation for `goto-web/src/views/goto/components/form/draw/lineParam.vue` improved the structure, but the real UI still differs from the approved HTML preview:

- The color control renders as a small dot instead of a square swatch.
- In the default panel width, `length` wraps to the next line for controls where all four fields are visible.
- The actual DOM contains global `.container` and Element UI input width rules that make controls wider than the local design intended.

The previous spec and plan are considered completed. This is a follow-up fix, not a continuation of the completed plan.

## Scope

Only fix the current `lineParam.vue` compact toolbar implementation and its regression test:

- Preserve the current outer `SettingForm` wrapper pattern so hidden fields remove the whole compact control.
- Remove the redundant inner `SettingForm` around `lineWidth` and `length` inputs.
- Add local CSS overrides so global `.group-attribute-line .container .el-input { width: 200px; }` does not expand inputs inside this toolbar.
- Make the color picker reference fill the full square control, including the `ColorPicker` component root span.
- Keep the approved line type preview shape.
- Make the four controls fit in one row at the current default panel width when all four are visible.

`lineEnd` remains out of scope and should not be rendered.

## Recommended Approach

Use the existing completed structure, but tighten it locally:

1. Keep each field wrapped by outer `SettingForm`:

```vue
<SettingForm class="line-param-setting line-param-setting--length" :container="setting.length">
  <el-tooltip content="长度" placement="top">
    <div class="line-param-field line-param-field--number line-param-field--length">
      ...
    </div>
  </el-tooltip>
</SettingForm>
```

2. Remove nested `SettingForm` inside width and length. `GInput` should be a direct child after the `W` or `L` badge.

3. Add local deep selectors under `lineParam.vue` only:

- `.line-param-field--color > span`
- `.line-param-field--color .el-popover__reference-wrapper`
- `.line-param-field--color .color-reference`
- `.line-param-field--number .el-input`
- `.line-param-field--number .el-input-number`
- `.line-param-field--number .el-input__inner`

These selectors should force the compact controls to respect the toolbar dimensions without changing shared components globally.

## Visual Rules

The approved visual target remains:

- Color is a square swatch-style control.
- Line type is about `70px` wide, with `5px` horizontal padding and a `20px` high inner preview.
- Width and length use `W` and `L` badges.
- All visible controls share one visual height.

To fit all four controls in one default-width row:

- Reduce toolbar gap from `8px` to about `6px`.
- Reduce numeric controls from `104px` to about `88px` to `92px`.
- Keep the value text prominent, but avoid the wide `200px` inherited input width.

The toolbar may still wrap on genuinely narrow containers, but it should not wrap at the current default panel width shown in the screenshot.

## Data Flow

No backend or API changes are required.

The existing `handleModify(fieldName, newValue, oldValue, parent)` emit path remains unchanged.

The same fields remain bound:

- `setting.color.value`
- `setting.lineType.value.value`
- `setting.lineWidth.value`
- `setting.length.value`

## Testing

Update the existing static regression test:

1. Assert width and length do not contain nested `SettingForm`.
2. Assert numeric controls have a compact width near `88px` to `92px`.
3. Assert toolbar gap is `6px`.
4. Assert local CSS overrides `.line-param-field--number .el-input` and `.line-param-field--number .el-input-number` with `width: 100% !important`.
5. Assert color root span and color reference wrapper are forced to full width and height.
6. Keep assertions for outer `SettingForm`, `W`/`L`, line type preview sizing, and `lineEnd` exclusion.

Manual verification:

1. Open the draw UI at the same default width as the screenshot.
2. Confirm `刻度线` shows color, line type, width, and length in one row.
3. Confirm color renders as a square swatch, not a dot.
4. Confirm hidden `length` still removes the whole control for `轴线`.
5. Confirm width and length still update values normally.
