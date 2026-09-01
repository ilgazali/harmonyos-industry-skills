---
id: LIFE-03
title: Chinese licence-plate keyboard - an invisible TextInput driving a three-layout custom Grid keypad
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
sample: huawei_industry_tree/02_convenient_life/downloads/VehicleKeyboard.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [TextInput, TextInputController, customKeyboard, "controller.stopEditing", "focusControl.requestFocus", Grid, GridItem, rowsTemplate, columnsTemplate, rowStart, rowEnd, columnStart, columnEnd, "@Provide", "@Consume", "@Link", "@Prop", "@State", "$$", CustomDialogController, "util.format", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "AppStorage.setOrCreate", "ConfigurationConstant.ColorMode"]
permissions: []
min_api: 20
modules: [entry, vehicleKeyboard]
findings: [HW-02-0011, HW-02-0012, HW-02-0013, HW-02-0014, HW-02-0015, HW-02-0016, HW-02-0017, HW-02-0018, HW-02-0019]
status: verified-with-fixes
---

## When to use

Load this card when you need a **field whose alphabet changes per character
position** - a Chinese licence plate is the canonical case: position 0 is a
province character (京, 沪, 粤...), position 1 is a letter, positions 2-6 are
letters or digits.

The system keyboard cannot express that. The technique here is to keep a real
`TextInput` in the tree for its focus and IME machinery, **shrink it to nothing
and push it off-screen**, draw the visible plate as seven `Stack` cells, and
attach a `Grid`-based custom keypad through `customKeyboard`. Tapping a cell
selects an index; the index picks which of the three key sets the `Grid`
renders.

The same shape fits any fixed-length code with position-dependent alphabets:
VINs, bank sort codes, national ID formats, seat labels.

Use `LIFE-01` for the app shell this page would sit in.

## Feature checklist

- Seven character cells plus a dashed "new energy" eighth slot, with a `·`
  separator after the second cell.
- The selected cell is highlighted (blue background, white text) and shows a
  caret image below the character.
- Tapping any cell opens the keypad appropriate to that position: province
  characters at index 0, upper-case letters at index 1, letters and digits from
  index 2 on.
- The plate string is always seven characters wide - it starts as seven spaces,
  and editing overwrites in place rather than appending.
- After the seventh character, the keyboard closes itself.
- Delete blanks the current cell and steps back; at the end of the string it
  truncates.
- The confirm button refuses a plate that still contains a space, then reveals a
  payment panel showing the plate with the dot inserted.

## Architecture

Two modules: an `entry` HAP and a `vehicleKeyboard` HAR that owns the whole
control.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       colour mode, full screen, avoid areas -> AppStorage
├── components
│   ├── LicensePlateComponent.ets       thin wrapper: card background around VehicleInput
│   └── PaymentInfoComponent.ets        the payment panel revealed after confirm
├── constants/StyleConstants.ets
└── pages/MainPage.ets                  @Entry - owns @Provide('licensePlate'), the dialog, the footer

features/vehicleKeyboard
├── index.ets                           export { VehicleInput }
└── src/main/ets
    ├── components/VehicleInput.ets     THE CARD: cells + hidden TextInput + key handler
    ├── components/Keyboard.ets         the Grid keypad
    └── constants/KeyboardConstant.ets  three key-data arrays, built at module load
```

The document's tree calls the entry page `Index.ets`; the file is `MainPage.ets`
(`HW-02-0011`). The struct inside it is still named `Index`, which is where the
error came from.

**The state has exactly one source of truth: a seven-character string.**
`MainPage` declares `@Provide('licensePlate') licensePlate: string = '       '`
- seven spaces, not the empty string. That is the load-bearing decision:

- Every cell can index into it (`this.inputValue[index]`) without a bounds check.
- "Not yet filled in" is representable as a space, so validation is a single
  `/\s/.test(...)`.
- Editing is always an overwrite (`substring(0, i) + v + substring(i + 1)`),
  never an insert, so the length never drifts.

`VehicleInput` takes it as `@Consume`, `Keyboard` takes it as `@Link` from
`VehicleInput`. Both are legal initialisers: "Since API version 9, the syntax
for initializing the child component @Link from the parent component @State is
`Comp({ aLink: this.aState })`", and `@Consume` is on the list of allowed
sources.

Flow of one keystroke:

```
GridItem.onClick  ->  Keyboard.onKeyboardEvent?.(item)      // receiver is Keyboard (HW-02-0017)
                  ->  mutates @Link inputValue / inputIndex
                  ->  propagates up to VehicleInput's @Consume / @State
                  ->  TextInput text is $$-bound, so .onChange fires
                  ->  onInputIndexChange(inputIndex) swaps the key set
                  ->  focusControl.requestFocus('textInput') keeps the keypad open
```

The `TextInput.onChange` handler is what re-selects the key set after each
keystroke; the keypad stays up because focus is requested again every time.

## Implementation steps

1. **Model the plate as a fixed-width string of spaces.** `@Provide` it from the
   page so both the input component and the payment panel can reach it without
   prop drilling.
2. **Build the key data at module load, not per render.** For each alphabet,
   loop over the characters and compute `position: [x, x, y, y]` from
   `i / columns` and `i % columns`; export the array and its `GridConfig`
   alongside.
3. **Give the delete key ordinary grid coordinates.** The sample pins it with
   `.position({ left: 153, top: 141 })`, which breaks on any other width
   (`HW-02-0016`). Use `position: [3, 3, 8, 9]` on a 10-column keyboard and let
   the fractional columns place it.
4. **Draw the plate** as a `Row` of `Stack` cells over `inputIndexArray`,
   inserting the `·` after index 1 and the dashed new-energy slot at the end.
   Each cell's `onClick` calls `onInputIndexChange(index)`.
5. **Add the hidden `TextInput`** with `.position({ left: -999, top: -999 })`,
   `.width(0)`, `.height(0)`, `.opacity(0)`, `.maxLength(7)`, an `.id()` so
   focus can be requested, and `text: $$this.inputValue` for the two-way bind.
   The document's snippet omits all of this (`HW-02-0012`).
6. **Attach the keypad** with `.customKeyboard(this.keyboardBuilder())`. The
   builder passes the current `items`/`rowCount`/`columnCount` plus the two
   `@Link` sources.
7. **Bind the key handler properly.** Pass
   `(item: IKeyAttribute) => this.onKeyboardEvent(item)`, not the bare method
   reference - the sample's version runs with the child as `this`
   (`HW-02-0017`).
8. **In `onInputIndexChange`, swap the key set and request focus again.** The
   `focusControl.requestFocus(id)` call at the end is what keeps the custom
   keyboard on screen across the index change.
9. **In `TextInput.onChange`, close the keyboard at the end of the plate**:
   when `inputIndex >= 7`, call `controller.stopEditing()`.
10. **Validate on confirm** with `/\s/.test(plate)` and format the display
    string with `util.format('%s·%s', plate.slice(0, 2), plate.slice(2))`.
11. **For the immersive padding, read two avoid areas, not one** - `TYPE_SYSTEM`
    for the top and `TYPE_NAVIGATION_INDICATOR` for the bottom
    (`HW-02-0013`) - inside the `setWindowLayoutFullScreen` promise
    (`HW-02-0014`), and subscribe to `avoidAreaChange` (`HW-02-0015`).

## Verified snippets

All snippets are from `VehicleKeyboard.zip`. Corrected forms are marked.

**Key data built once at module load - `VehicleKeyboard.zip#features/vehicleKeyboard/src/main/ets/constants/KeyboardConstant.ets`** (as shipped)

```typescript
export interface GridConfig { row: number; column: number; }

export enum EKeyType { INPUT, DELETE }

export interface IKeyAttribute {
  label: string | Resource;
  value?: string;
  type?: EKeyType;
  fontSize?: number;
  fontColor?: string | Color;
  backgroundColor?: string | Color;
  position?: [number, number, number, number];   // rowStart, rowEnd, columnStart, columnEnd
  width?: number;
  offset?: Position | Edges;
}

let province = ['京', '豫', '川', '赣', '津', '沪', '粤', '渝', '港', '甘', /* ... 34 total ... */ '蒙'];
const provinceGridConfig: GridConfig = { row: 4, column: 10 };
const provinceKeyData: IKeyAttribute[] = [];
for (let i = 0; i < province.length; i++) {
  let x = parseInt(String(i / provinceGridConfig.column), 10);
  let y = i % provinceGridConfig.column;
  provinceKeyData.push({
    label: province[i], value: province[i], type: EKeyType.INPUT,
    fontSize: 16, fontColor: Color.Black, backgroundColor: Color.White,
    position: [x, x, y, y]
  });
}
```

**`position: [x, x, y, y]` is a one-cell span.** `rowStart === rowEnd` and
`columnStart === columnEnd` place the item in exactly that cell, which is what
lets a flat array drive a two-dimensional grid without any layout maths in the
component.

The upper-case alphabet deliberately **omits I and O** - they are not used in
Chinese plates because they collide with 1 and 0. The numeric keyboard is
`['0'..'9', ...upperCase]`, so 34 keys over 10 columns.

**The delete key - same file** (corrected, see `HW-02-0016`)

```typescript
// As shipped - the offset pins the key at a fixed vp coordinate:
//   position: [3, 3, 4, 4],
//   offset: { left: 153, top: 141 }      // 164 for the 7-column upper-case board
// and Keyboard.ets applies it with .position(item.offset), which takes the item
// out of grid flow and makes rowStart/columnStart dead.

provinceKeyData.push({                      // FIX: place it with grid coordinates only
  label: $r('app.media.delete'),
  type: EKeyType.DELETE,
  fontSize: 16,
  fontColor: Color.Black,
  backgroundColor: '#D4D3D3',
  position: [3, 3, 8, 9]                    // last row, spanning the final two columns
});
```

Two different magic numbers were needed for two boards (153 for 10 columns, 164
for 7) precisely because the offsets do not scale - and the HAR advertises
`tablet` and `2in1`, where neither number is right.

**The keypad - `VehicleKeyboard.zip#features/vehicleKeyboard/src/main/ets/components/Keyboard.ets`** (as shipped)

```typescript
@Component
export struct Keyboard {
  @Link inputValue: string;
  @Link inputIndex: number | null;
  @Prop items: IKeyAttribute[];
  @Prop rowCount: number;
  @Prop columnCount: number;
  public onKeyboardEvent: Function | null = null;
  private rowSpace: number = 5;
  private columnSpace: number = 5;
  private itemHeight: number = 42;

  build() {
    Column() {
      Grid() {
        ForEach(this.items, (item: IKeyAttribute) => {
          GridItem() {
            this.keyboardGridItem(item);
          }
          .width(item.width ? item.width : '100%')
          .height(this.itemHeight)
          .backgroundColor(item.backgroundColor)
          .borderRadius(KeyboardConstants.BORDER_RADIUS_FOUR)
          .rowStart(item?.position?.[0])
          .rowEnd(item?.position?.[1])
          .columnStart(item?.position?.[2])
          .columnEnd(item?.position?.[3])
          .onClick(() => {
            this.onKeyboardEvent?.(item);
          });
        }, (item: IKeyAttribute, index: number) => JSON.stringify(item) + index);
      }
      .rowsTemplate(new Array(this.rowCount).fill('1fr').join(' '))
      .columnsTemplate(new Array(this.columnCount).fill('1fr').join(' '))
      .rowsGap(this.rowSpace)
      .columnsGap(this.columnSpace)
      .width('100%')
      .height(this.itemHeight * this.rowCount + this.rowSpace * (this.rowCount - 1));
    }
    .width('100%')
    .height(KeyboardConstants.TEXT_INPUT_HEIGHT)   // 210
    .backgroundColor($r('app.color.text_input_color'));
  }
}
```

**`new Array(n).fill('1fr').join(' ')` is the idiom for a runtime-sized grid.**
`rowsTemplate` and `columnsTemplate` take a space-separated string, so the row
and column counts can change with the key set - 4x10 for province and numeric,
4x7 for upper case - without any conditional layout.

The explicit `.height(itemHeight * rowCount + rowSpace * (rowCount - 1))` is
needed because a `Grid` with `rowsTemplate` fills its own height; without the
computed value the 42 vp key height would not be honoured.

The key generator `JSON.stringify(item) + index` is safe here - the items carry
distinct labels, and the index disambiguates the rest.

**The hidden TextInput - `VehicleKeyboard.zip#features/vehicleKeyboard/src/main/ets/components/VehicleInput.ets:115`** (as shipped; the document omits five of these lines, `HW-02-0012`)

```typescript
// 隐藏，用于弹出自定义键盘  ("hidden - used to raise the custom keyboard")
TextInput({ controller: this.controller, text: $$this.inputValue })
  .id('textInput')
  .position({ left: -999, top: -999 })
  .width(0)
  .height(0)
  .opacity(0)
  .maxLength(7)
  .customKeyboard(this.keyboardBuilder())
  .onChange((value) => {
    this.codeTxt = value;
    if (this.inputIndex !== null) {
      if (this.inputIndex >= 7) {
        this.controller.stopEditing();   // 停止编辑，收起键盘
      } else {
        this.onInputIndexChange(this.inputIndex);
      }
    }
  });
```

**Four independent ways of hiding it, and all four are needed.** `width(0)` and
`height(0)` collapse the box, `opacity(0)` stops any residual caret or selection
handle painting, and `position({ left: -999, top: -999 })` moves it out of the
parent's bounds so it cannot intercept a tap. Remove any one and the effect is
visible on some device.

The component is still fully live: it owns focus, and its `customKeyboard` is
what the system raises. `$$this.inputValue` gives the two-way bind that makes
`onChange` fire when the key handler mutates the plate string.

`inputIndex >= 7` is reachable: the handler increments the index past the last
cell (6) when it overwrites in place, and `onChange` then closes the keyboard.

**The key handler - same file, lines 141-169** (as shipped)

```typescript
onKeyboardEvent(item: IKeyAttribute) {
  switch (item.type) {
    case EKeyType.INPUT:
      if (this.inputIndex === null || (this.inputIndex === this.inputValue.length)) {
        this.inputValue += item.value;                       // append at the end
      } else {
        this.inputValue =                                    // overwrite in place
          this.inputValue.substring(0, this.inputIndex) + item.value +
          this.inputValue.substring(this.inputIndex + 1);
        this.inputIndex = this.inputIndex + 1;
      }
      break;
    case EKeyType.DELETE:
      if (this.inputIndex === null || (this.inputIndex === this.inputValue.length)) {
        this.inputValue = this.inputValue.slice(0, -1);
      } else {
        this.inputValue =
          this.inputValue.substring(0, this.inputIndex) + ' ' +
          this.inputValue.substring(this.inputIndex + 1);
        this.inputIndex = Math.max(this.inputIndex - 1, 0);
      }
      break;
    default:
      hilog.error(0x0000, 'VehicleInput', 'Sorry, No found input type.');
  }
}
```

**Delete writes a space, it does not shorten the string.** That is what keeps
the seven-space invariant, and it is why the confirm check can be a single
whitespace regex. The `slice(0, -1)` branch only runs when the caret is past the
end, which the seven-space seed makes unreachable in normal use.

**Wiring the handler - same file, line 45** (corrected, see `HW-02-0017`)

```typescript
@Builder
keyboardBuilder() {
  Keyboard({
    items: this.items,
    rowCount: this.rowCount,
    columnCount: this.columnCount,
    onKeyboardEvent: (item: IKeyAttribute) => this.onKeyboardEvent(item),  // FIX: bind the owner
    inputValue: this.inputValue,     // @Link from @Consume - legal since API 9
    inputIndex: this.inputIndex      // @Link from @State
  });
}
```

The shipped code passes the bare `this.onKeyboardEvent`. `Keyboard` then calls
it as `this.onKeyboardEvent?.(item)`, so `this` inside the handler is the
`Keyboard` instance - it only works because `Keyboard` declares `@Link
inputValue` and `@Link inputIndex` under exactly those names.

**Index change and focus - same file, lines 171-187** (as shipped)

```typescript
onInputIndexChange(index: number) {
  this.inputIndex = index;
  if (index === 0) {
    this.items = provinceKeyData;
    this.rowCount = provinceGridConfig.row;
    this.columnCount = provinceGridConfig.column;
  } else if (index === 1) {
    this.items = upperCaseKeyData;
    this.rowCount = upperCaseGridConfig.row;
    this.columnCount = upperCaseGridConfig.column;
  } else {
    this.items = numericKeyData;
    this.rowCount = numericGridConfig.row;
    this.columnCount = numericGridConfig.column;
  }
  focusControl.requestFocus('textInput');
}
```

**`focusControl.requestFocus` is not optional.** It is what raises the custom
keyboard on the first cell tap and what keeps it up when the key set is swapped
mid-entry. Without it the keypad closes as soon as the layout changes.

**Page-level validation - `VehicleKeyboard.zip#entry/src/main/ets/pages/MainPage.ets:161`** (as shipped)

```typescript
.onClick(() => {
  if (/\s/.test(this.licensePlate)) {
    this.getUIContext().getPromptAction().showToast({ message: '请输入完整车牌号' });
  } else {
    this.isShow = true;
    this.addedDotString = util.format('%s·%s', this.licensePlate.slice(0, 2), this.licensePlate.slice(2));
    focusControl.requestFocus('button');
  }
});
```

**One regex is the whole validation,** because unfilled cells are spaces rather
than a short string. `focusControl.requestFocus('button')` pulls focus off the
hidden `TextInput`, which is how the keypad is dismissed on success.

**Immersive insets - `VehicleKeyboard.zip#entry/src/main/ets/entryability/EntryAbility.ets:41`** (corrected, see `HW-02-0013`, `HW-02-0014`, `HW-02-0015`)

```typescript
const windowClass: window.Window = windowStage.getMainWindowSync();
windowClass.setWindowLayoutFullScreen(true).then(() => {          // FIX: the sample drops this promise
  AppStorage.setOrCreate('topAvoid',
    windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height);
  AppStorage.setOrCreate('bottomAvoid',                           // FIX: NAVIGATION_INDICATOR, not SYSTEM
    windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height);
  windowClass.on('avoidAreaChange', this.avoidAreaCallback);      // FIX: the sample never subscribes
}).catch((err: BusinessError) => {
  hilog.error(0x0000, 'EntryAbility', 'fullscreen failed: %{public}s', JSON.stringify(err));
});
```

The shipped code takes `bottomRect.height` off the `TYPE_SYSTEM` area, which the
reference describes as the status bar - so the bottom padding is zero and the
`.position({ bottom: ... })` footer slides under the navigation bar.

**Light mode is pinned - same file, line 24** (as shipped)

```typescript
this.context.getApplicationContext()
  .setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
```

Worth knowing: the plate colours are hard-coded light values, so the sample
forces light mode rather than providing a dark palette.

## Permissions & config

None. No `requestPermissions` block appears in either module.

`VehicleKeyboard.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }]
  }
}
```

`main_pages.json` is `{ "src": ["pages/MainPage"] }`.

`VehicleKeyboard.zip#features/vehicleKeyboard/src/main/module.json5` (corrected, see `HW-02-0018`)

```json5
{
  "module": {
    "name": "vehicleKeyboard",
    "type": "har",
    "deviceTypes": ["phone"]        // FIX: shipped as ["default", "tablet", "2in1"]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)` for both
`compatibleSdkVersion` and `targetSdkVersion`.

Note the HAR's package name and the entry's dependency key differ in spelling -
`"name": "@ohos/vehiclekeyboard"` in the HAR's `oh-package.json5` against
`"@ohos/vehicle-keyboard": "file:../features/vehicleKeyboard"` in the entry's.
The import in `LicensePlateComponent.ets` uses the hyphenated dependency key.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 77-79, matching
  `compatibleSdkVersion: "6.0.0(20)"`).
- Phone only. The entry HAP declares `deviceTypes: ["phone"]`, and the hard-coded
  keypad offsets (`HW-02-0016`) are tuned for a phone width.
- The plate length is fixed at 7 in four independent places:
  `inputIndexArray = [0..6]`, `.maxLength(7)`, the `inputIndex >= 7` close
  condition, and the seven-space seed in `MainPage`. The dashed eighth
  "new energy" slot is **decoration only** - an 8-character new-energy plate
  cannot be entered.
- The three key sets are fixed at module load; the `EKeyType` enum has no
  confirm or shift key.
- Light mode is forced application-wide.
- `Keyboard` renders with `rowCount = 0` before the first cell tap, which makes
  its computed height `-5`. It is never shown in that state because
  `customKeyboard` builds lazily, but do not reuse `Keyboard` outside that flow
  without seeding the counts.

## Pitfalls

- **`HW-02-0011` - the document's tree says `pages/Index.ets`; the file is
  `pages/MainPage.ets`.** `main_pages.json` and `loadContent` both name
  `pages/MainPage`. Follow the tree and you get a blank window.
- **`HW-02-0012` - the document's step-1 snippet omits the hiding.** It shows
  only `.id()`, `.customKeyboard()` and `.onChange()`; the shipped code adds
  `.position({ left: -999, top: -999 })`, `.width(0)`, `.height(0)`,
  `.opacity(0)` and `.maxLength(7)`. Those five lines are the technique.
- **`HW-02-0013` - the bottom inset is read from the wrong avoid area.**
  `TYPE_SYSTEM` is the status bar; the bottom navigation bar is
  `TYPE_NAVIGATION_INDICATOR`. The footer ends up under the system bar.
- **`HW-02-0014` - `setWindowLayoutFullScreen` returns a promise that is
  dropped,** and the avoid area is queried on the next line, racing the layout
  change. Chain the reads inside `.then()` and add a `.catch()`.
- **`HW-02-0015` - the insets are never refreshed.** There is no
  `on('avoidAreaChange')` anywhere in the zip, so the padding is stale after a
  rotation.
- **`HW-02-0016` - the delete key is pinned with `.position({ left: 153, top:
  141 })` inside a `1fr` grid.** That override kills the item's grid
  coordinates and needs a different magic number per board. Use grid coordinates.
- **`HW-02-0017` - `onKeyboardEvent: this.onKeyboardEvent` is unbound.** It runs
  with `Keyboard` as `this` and works only because `Keyboard` declares
  identically named `@Link`s. Pass an arrow function instead.
- **`HW-02-0018` - the HAR's `deviceTypes` omits `phone`** and uses `default`,
  which the reference calls build-only ("Change the device type to phone").
- **`HW-02-0019` - `PaymentInfo` declares `@Consume('licensePlate')` and never
  reads it,** so it re-renders on every keystroke for nothing. It draws
  `addedDotString` instead.
- **Do not seed the plate with `''`.** Every cell indexes into the string
  directly and delete writes a space at the caret; an empty seed breaks both the
  bounds-free indexing and the single-regex validation.
- **Do not drop `focusControl.requestFocus('textInput')` from
  `onInputIndexChange`.** Swapping `items`/`rowCount` rebuilds the keypad, and
  without the focus request it closes instead of switching alphabets.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInput`, `customKeyboard`, `TextInputController.stopEditing`, `maxLength`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `Grid`, `rowsTemplate`/`columnsTemplate`, `GridItem` `rowStart`/`rowEnd`/`columnStart`/`columnEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-guides/03_application-framework/arkts-link.md` - `@Link` initialisation from `@State`/`@Consume` (`Comp({ aLink: this.aState })`, since API 9)
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - the `@Provide`/`@Consume` alias pairing used for `licensePlate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` - `AvoidAreaType`: `TYPE_SYSTEM` is the status bar, `TYPE_NAVIGATION_INDICATOR` is the bottom bar
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowLayoutFullScreen` returns `Promise<void>`; `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` - `deviceTypes`, including the "default ... cannot be released" note
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `LIFE-01` - the industry shell this page would sit in
