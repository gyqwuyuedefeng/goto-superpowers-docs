# Legend Position None Hide Legend Box Just Design

## Background

The runtime drawing parameter panel has a shared "图例设置" group. Its data includes a `legendSettings` parameter with `legendPosition` and `legendBoxJust`.

The current global runtime association flow is:

1. `goto-web/src/constant/associationParamModify.js` calls `PublicFunctions.generalAssociationParamModify(commonParams)`.
2. `goto-web/src/constant/modify/publicFunction.js` detects `commonParams.setting.paramName === 'legendSettings'`.
3. `goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js` handles `legendPosition` changes.

That handler already hides most legend controls when `legendPosition.value.value === 'none'`, but it does not hide `legendBoxJust`, which renders as "对齐方式" in the runtime drawing panel.

## Scope

This change only affects the runtime drawing parameter panel.

In scope:

- Hide "对齐方式" when "图例位置" is set to "不显示" (`legendPosition.value.value === 'none'`).
- Restore "对齐方式" when "图例位置" is changed back to a visible position.
- Keep the existing automatic `legendDirection` behavior for `top`, `bottom`, `left`, and `right`.

Out of scope:

- Do not change the plot setting/configuration editor under `views/goto/components/form/plotSetting`.
- Do not change dictionary data, default values, backend contracts, or generated plot code.
- Do not change the legend count rule that only shows direction and alignment when more than one legend is enabled.

## Design

Update `goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js`.

When `optionName === 'legendPosition'`:

- If `legendPosition.value.value === 'none'`, set `commonParams.setting.legendBoxJust.visible = false` along with the existing hidden legend controls.
- Otherwise, set `commonParams.setting.legendBoxJust.visible = true` along with the existing restored legend controls.

The runtime component `views/goto/components/form/draw/legendSettingsParam.vue` already renders the alignment control through:

```vue
<SettingForm v-if="legendCount > 1" :container="setting.legendBoxJust">
```

Because `SettingForm` uses the setting container visibility, updating `legendBoxJust.visible` in the shared association handler is enough for the runtime UI. No runtime template condition needs to be added.

## Data Flow

1. User changes "图例位置" in the runtime "图例设置" panel.
2. The draw form emits modify parameters with `{ optionName: 'legendPosition' }`.
3. `associationParamModify` runs the global association handler.
4. `legendSettingsParamModify` updates nested field visibility on the current `legendSettings` object.
5. Vue re-renders the runtime panel and hides or restores "对齐方式".

## Error Handling

The implementation should follow the current file style and keep the change minimal. The existing handler assumes all legend fields exist on `commonParams.setting`. This design keeps that assumption to avoid broad defensive refactoring outside the request.

## Testing

Manual verification:

- Open a plot with the runtime "图例设置" panel.
- Ensure more than one legend is enabled so "对齐方式" can normally appear.
- Change "图例位置" to "不显示"; "对齐方式" should hide.
- Change "图例位置" back to "左侧", "右侧", "顶部", "底部", or "内部"; "对齐方式" should show again when the multi-legend condition is met.
- Confirm the plot setting/configuration editor remains unchanged.

Code verification:

- Run the relevant frontend lint or test command available in `goto-web` if the project has one configured.
