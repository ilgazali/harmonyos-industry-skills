---
id: UTIL-45
title: Keyboard candidate strip - two ways to reserve a suggestion row above a custom IME panel
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/45_kikainput.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
sample: huawei_industry_tree/15_utilities/downloads/SFI20260129174558242978.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.IMEKit"]
apis: [AbilityConstant, AbilityStage, UIAbility, Want, base, deviceInfo, display, hilog, inputMethod, inputMethodEngine, keyCode, window]
min_api: 20
modules: [entry (scene1), entry (scene2)]
findings: [HW-15-0092, HW-15-0093, HW-15-0094, HW-15-0095, HW-15-0096, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **third-party input method** and need a
row of candidate/association words above the keys. That row is the part of an
IME that everyone underestimates: a soft-keyboard panel is a window the system
sizes and positions for you, and anything you want to draw has to fit inside
it - or the panel has to be redefined.

The sample ships the two answers as two near-identical projects. **Scene 1**
keeps the panel at the normal keyboard height and gives up part of it: the
candidate strip replaces the keyboard's own toolbar row. **Scene 2** makes the
panel full screen with `adjustPanelRect({ fullScreenMode: true })` and then
declares a smaller *input region* (hot zone), so touches outside the keys pass
through to the app underneath while the panel is still free to paint the strip
anywhere on the screen.

The choice generalises beyond candidates: any IME overlay that must sit outside
the key area - a clipboard bar, an emoji drawer preview, a translation ribbon -
is the same decision. Scene 1 is cheap and safe; scene 2 is the only way to
draw above the panel, and it costs you a hand-maintained hot zone that the
sample does not compute correctly (`HW-15-0094`).

## Feature checklist

- Installing the app registers an input method with two subtypes (`en-US` and
  `zh-CN`), selectable from the system keyboard switcher.
- The keyboard panel is created at a device-appropriate height and moved to the
  bottom of the screen.
- The keyboard renders letters, numbers and symbols; letter keys commit text
  into the editing client, and backspace deletes from it.
- Typing letters accumulates a candidate string; when it is non-empty the
  toolbar row (`TopMenu`) is replaced by the candidate row (`SmallMenu`), and
  backspace trims it again.
- Typing `hel` replaces the typed prefix with a `hello world` preview
  (underlined pre-commit text) via `setPreviewText`.
- The keyboard button in the toolbar opens a submenu listing every enabled
  input method and the current one's subtypes; tapping one switches to it.
- Rotating the device re-measures the display and resizes/moves the panel.
- An external hardware keyboard is handled: key-down/key-up are intercepted,
  shift state tracked, arrows and delete mapped to cursor/delete calls.
- Scene 2 only: the panel covers the screen, the candidate row is drawn above
  the keys, and the app below still receives touches outside the hot zone.

## Architecture

Two full projects in one zip, `SFI20260129174558242978-scene1` and
`-scene2`, each a single `entry` module.

```
entry/src/main/ets
├── Application/AbilityStage.ets            boilerplate stage
├── InputMethodExtensionAbility
│   ├── InputMethodService.ets              the extension: 3 lines of real work
│   ├── model/KeyboardController.ets        panel lifecycle + InputHandler (713/740 lines)
│   └── pages/Index.ets                     @Entry of the PANEL, not of the app
├── common/StyleConfiguration.ets           per-device key metrics as $r() Resources
├── components/                             TopMenu, SmallMenu, KeyMenu, NumberMenu,
│                                           SymbolMenu, Submenu, EditView, KeyItem,
│                                           KeyItemNumber, KeyItemGray, DeleteItem,
│                                           SpaceItem, ReturnItem, CustomInput
├── entryability/EntryAbility.ets           the installer app; loads PrivatePreview
├── model/HardKeyUtils.ets                  physical keyCode -> character table
├── model/KeyboardKeyData.ets               the key rows, MenuType/SubMenuType enums
├── model/Log.ets                           hilog wrapper
└── pages/PrivatePreview.ets                one TextInput to type into for testing
```

**The documented tree does not match the zip.** Doc 45 shows `pages/Index.ets`,
`model/KeyboardController.ets` and `ServiceExtAbility/ServiceExtAbility.ets`
under a project called `KikaInput`; the zip puts `Index.ets`,
`KeyboardController.ets` and `InputMethodService.ets` under
`InputMethodExtensionAbility/`, and ships no `ServiceExtAbility` directory (the
class inside `InputMethodService.ets` is still named `ServiceExtAbility`). The
doc also never mentions that the download contains two projects.

**What actually differs between the two scenes** is three regions of two files
- everything else is byte-identical:

| | scene1 | scene2 |
|---|---|---|
| `KeyboardController.initWindow` | `panel.resize(dWidth, keyHeight)` | `adjustPanelRect(FLG_FIXED, {fullScreenMode:true, ...})` then `panel.resize(100, 100)` |
| `Index.build` top row | `TopMenu()` / `SmallMenu()` swap | extra full-width `Row` with the candidate `Text` above a `Column().height('40%')` |
| `Index.build` alignment | keys centred | `justifyContent(FlexAlign.End)` - keys pinned to the bottom of a full-screen panel |

**The design decision worth copying** is the split between
`InputMethodService`, `KeyboardController` and `InputHandler`. The extension
ability does nothing but forward `onCreate`/`onDestroy` to a module-level
`keyboardController` singleton; `KeyboardController` owns the panel and every
`inputMethodAbility.on(...)` subscription; `InputHandler` owns the *client*
(the `InputClient` handed to you when an editor attaches) and is published into
`AppStorage` under `'inputHandler'`, so any key component deep in the tree can
call `InputHandler.getInstance().insertText(...)` without the page threading a
callback down five levels. Panel lifecycle and text lifecycle are different
lifetimes - the panel outlives an individual editing session - and keeping them
in separate objects is why the key components stay this small.

The candidate string is the counter-example and is worth avoiding as shipped:
`Index` declares `@Provide lianxiang: string`, `KeyItemNumber` and
`DeleteItem` `@Consume` and mutate it, and `SmallMenu` renders it three times
side by side to fake three candidates. There is no candidate engine, no
dictionary, and `KeyItem` (the caseless keys) forgets to append at all - so
symbols type into the client but never into the strip. Treat `lianxiang` as the
wiring diagram for where a real engine's output would go, not as an
implementation.

## Implementation steps

1. **Declare the extension** in `module.json5` with `"type": "inputMethod"`,
   `exported: true`, and a `ohos.extension.input_method` metadata entry
   pointing at `$profile:input_method_config` - the profile that lists the
   subtypes. No permission is needed; being an IME is granted by the user
   enabling it in Settings.
2. **Create the panel in `onCreate`**, not lazily: `createPanel` →
   `resize` → `moveTo` → `setUiContent`, chained, because each call is only
   valid after the previous one resolved.
3. **Derive the height from the display**, `dHeight * keyHeightRate`, and
   publish the resulting metrics into `AppStorage` so the ArkUI side can style
   keys without re-measuring.
4. **Re-run the same measurement from `display.on('change')`** so a rotation
   resizes and repositions the panel.
5. **For an in-panel strip (scene 1)**, just render it as the first child of
   the panel's root `Column` and give the key area `layoutWeight(1)`.
6. **For an above-panel strip (scene 2)**, call `adjustPanelRect` with
   `fullScreenMode: true` *before* `resize`, and give it `portraitInputRegion`
   / `landscapeInputRegion` rectangles covering only the keys. Compute those
   from the display metrics - do not hardcode them, and do not shrink the panel
   to `100x100` afterwards (`HW-15-0094`).
7. **Register every listener in one place and remove every one of them** in the
   symmetric teardown, keeping the callback references (`HW-15-0095`).
8. **Read the surrounding text before inserting**, or accept that
   `getForwardSync` after `insertTextSync` already contains the new character
   (`HW-15-0096`).
9. **Match on the item you were given** when switching input methods, and stop
   at the first match (`HW-15-0092`).
10. **Route `setImmersiveMode` through the panel that actually exists** -
    the one held by `KeyboardController`, not the page's never-assigned field
    (`HW-15-0093`). For the application side of the same feature see `UTIL-13`.

## Verified snippets

All snippets are from `SFI20260129174558242978.zip`. The scene is named on each
one. Corrected forms are marked.

**Scene 1 panel creation — `entry/src/main/ets/InputMethodExtensionAbility/model/KeyboardController.ets`** (as shipped)

```typescript
const KEYBOARD_HEIGHT_RATE_DEFAULT: number = 0.35;
const KEYBOARD_HEIGHT_RATE_PHONE: number = 0.38;
const NAVIGATIONBAR_HEIGHT_DEFAULT: number = 72;

private initWindow(): void {
  if (this.mContext === undefined) {
    return;
  }
  let dis = display.getDefaultDisplaySync();
  let dWidth = dis.width;
  let dHeight = dis.height;
  let navigationBar_height = NAVIGATIONBAR_HEIGHT_DEFAULT;
  let keyHeightRate = KEYBOARD_HEIGHT_RATE_DEFAULT;
  AppStorage.setOrCreate('windowWidth', dis.width);
  AppStorage.setOrCreate('windowHeight', dis.height);
  let isLandscape = dis.width > dis.height;
  let isRkDevice = false;
  AppStorage.setOrCreate('isLandscape', isLandscape);
  if (dWidth === DEVICE_PHONE.width && dHeight === DEVICE_PHONE.height) {
    navigationBar_height = 0;
    keyHeightRate = KEYBOARD_HEIGHT_RATE_PHONE;
  }             // ... four more exact width/height branches: phone landscape, RK, tablet
  let keyHeight = dHeight * keyHeightRate;
  this.barPosition = dHeight - keyHeight - navigationBar_height;
  let panelInfo: inputMethodEngine.PanelInfo = {
    type: inputMethodEngine.PanelType.SOFT_KEYBOARD,
    flag: inputMethodEngine.PanelFlag.FLG_FIXED
  };
  AppStorage.setOrCreate('inputStyle',
    StyleConfiguration.getInputStyle(isLandscape, isRkDevice, deviceInfo.deviceType));

  inputMethodAbility.createPanel(this.mContext, panelInfo).then((panel: inputMethodEngine.Panel) => {
    this.panel = panel;
    panel.resize(dWidth, keyHeight).then(() => {
      panel.moveTo(0, this.barPosition).then(() => {
        panel.setUiContent('InputMethodExtensionAbility/pages/Index').then(() => {
          this.inputHandle.addLog('loadContent finished');
        });
      });
    });
  });
}
```

**Three things carry the design.** `PanelType.SOFT_KEYBOARD` with
`PanelFlag.FLG_FIXED` is what makes the system treat this window as *the*
keyboard - it docks at the bottom and applications avoid it automatically.
The nesting of `resize` → `moveTo` → `setUiContent` is not stylistic: `moveTo`
positions a panel whose size must already be known, and loading UI into a panel
that has not been laid out gives you a frame at the wrong size. And the panel
height is a *ratio of the display*, published to `AppStorage` alongside a
`KeyStyle` record, so `StyleConfiguration` can hand every key component its
width and font size as `Resource` references.

The device matching is by exact width/height pairs (`1344x2772` phone,
`720x1280` RK board, `2560x1600` tablet), which is fine for a demo and wrong
for a product: any display not in the table falls back to the 0.35 ratio and a
72 px navigation bar. Note also the RK branch assigns
`KEYBOARD_HEIGHT_RATE_DEFAULT` (0.35) to `navigationBar_height`, a pixel value
- a 0.35 px navigation bar (`HW-15-0094`).

**Scene 2 full-screen panel with a hot zone — same file, `initWindow`** (corrected, see `HW-15-0094`)

```typescript
inputMethodAbility.createPanel(this.mContext, panelInfo).then((panel: inputMethodEngine.Panel) => {
  this.panel = panel;
  // FIX: shipped code hardcodes 300/650/2000x500 and 0/1613/1200x1000 for one display.
  // The hot zone is relative to the panel's top-left, so derive it from the metrics.
  let keyAreaTop = dHeight - keyHeight - navigationBar_height;
  let portraitInputRegion: Array<window.Rect> = [{
    left: 0,
    top: keyAreaTop,            // distance from the top of the (full-screen) panel
    width: dWidth,
    height: keyHeight + navigationBar_height
  }];
  let landscapeInputRegion: Array<window.Rect> = [{
    left: 0, top: keyAreaTop, width: dWidth, height: keyHeight
  }];
  let panelFlag: inputMethodEngine.PanelFlag = inputMethodEngine.PanelFlag.FLG_FIXED;

  let panelRect: inputMethodEngine.EnhancedPanelRect = {
    landscapeAvoidY: keyAreaTop,       // apps avoid only below this line
    landscapeInputRegion: landscapeInputRegion,
    portraitAvoidY: keyAreaTop,
    portraitInputRegion: portraitInputRegion,
    fullScreenMode: true
  };
  panel.adjustPanelRect(panelFlag, panelRect);
  panel.resize(dWidth, dHeight).then(() => {   // FIX: shipped code is resize(100, 100)
    panel.moveTo(0, 0).then(() => {
      panel.setUiContent('InputMethodExtensionAbility/pages/Index');
    });
  });
});
```

**`fullScreenMode` is what buys the space, `InputRegion` is what gives it
back.** With `fullScreenMode: true` the panel window covers the screen, so
`landscapeRect`/`portraitRect` become optional and you can draw a candidate
strip at any y. But a full-screen window would swallow every touch, so the
reference makes the touch-sensitive area a separate declaration: the
`InputRegion` arrays hold one to four rectangles, "relative to the left vertex
of the input method panel window", outside of which events go to the app below.
`portraitAvoidY` is the third piece - "distance between the avoid line and the
top of the panel, in px" - and it is what the *host application* avoids, so it
must sit at the top of the keys, not at the top of the window; anything above
it is drawn over the app rather than pushing it up.

The shipped code gets the mechanism right and the numbers wrong. Its rectangles
(`top: 1613`, `width: 1200`, and a landscape `300/650/2000x500`) are copied
from the reference example and only make sense on one display; worse,
`resize(100, 100)` immediately after `adjustPanelRect` reduces the panel to a
100x100 px square, at which point the hot-zone rectangles reach far outside the
window they are relative to. Doc 45 reproduces the same `resize(100, 100)`
verbatim, so a reader copying the documented snippet inherits the bug.

**The panel page — `entry/src/main/ets/InputMethodExtensionAbility/pages/Index.ets`** (scene 1, corrected, see `HW-15-0093`)

```typescript
@Entry
@Component
struct Index {
  @Provide menuType: number = MenuType.NORMAL;
  @StorageLink('inputPattern') @Watch('inputPatternChange') inputPattern: InputType = InputType.Normal;
  @StorageLink('submenuType') submenuType: number = SubMenuType.NORMAL;
  @StorageLink('isLandscape') @Watch('change') isLandscape: boolean = false;
  @StorageLink('isRkDevice') isRkDevice: boolean = true;
  @StorageLink('inputStyle') inputStyle: KeyStyle =
    StyleConfiguration.getInputStyle(this.isLandscape, this.isRkDevice, DEVICE_TYPE);
  @StorageLink('subtypeChange') subtypeChange: number = 0;
  @Provide lianxiang: string = '';

  aboutToAppear(): void {
    // FIX: `this.panel` is never assigned in this struct; the live panel is the
    // controller's. Expose it there - one accessor on KeyboardController:
    //   public setImmersiveMode(mode: inputMethodEngine.ImmersiveMode): void {
    //     this.panel?.setImmersiveMode(mode);
    //   }
    // and keep the callback in a field so it can be off'd on disappear.
    inputMethodEngine.getKeyboardDelegate().on('editorAttributeChanged',
      (attr: inputMethodEngine.EditorAttribute) => {
        if (attr.immersiveMode == 1) {
          keyboardController.setImmersiveMode(inputMethodEngine.ImmersiveMode.DARK_IMMERSIVE);
        }
      });
  }

  build() {
    Stack() {
      Column() {
        if (this.lianxiang.length === 0) {
          TopMenu()
        } else {
          SmallMenu()                     // 预留联想词区域 - the reserved candidate row
        }
        Column() {
          if (this.submenuType > SubMenuType.NORMAL) {
            if (this.submenuType === SubMenuType.MENU) {
              Submenu()                       // the IME/subtype switcher
            } else {
              EditView();                     // the cursor/selection pad
            }
          } else if (this.menuType === MenuType.NORMAL) {
            if (this.subtypeChange == 0) {
              KeyMenu()
            } else {
              NumberMenu()
            }
          } else if (this.menuType === MenuType.NUMBER) {
            NumberMenu()
          } else {
            SymbolMenu()
          }
        }
        .width('100%')
        .layoutWeight(1)
        .justifyContent(FlexAlign.Center)
      }
      .height('100%')
    }
    .height('100%')
  }
}
```

**This is scene 1's whole answer to the question:** the candidate row is not an
extra band, it is a *swap*. `TopMenu` (the toolbar with the switcher and the
hide-keyboard arrow) and `SmallMenu` (the candidate row) occupy the same slot
and have the same `downMenuHeight`, so the keys below never move. The key area
takes `layoutWeight(1)` of what is left, which is why the panel height computed
in `initWindow` is the only geometry input the page needs.

Scene 2 wraps this same body in an outer `Column().height('100%')
.justifyContent(FlexAlign.End)`, drops a `Row` holding the candidate `Text`
above it, and caps the keyboard block at `height('40%')`. That is the entire UI
cost of moving the strip outside the keyboard - the rest is the panel geometry.

`this.panel` in the shipped struct is declared and never assigned, so
`this.panel?.setImmersiveMode(...)` is optional-chained into a no-op and the
documented dark-immersive feature never runs; the reference is explicit that
`setImmersiveMode` throws `12800002` when no panel has been created. The
`editorAttributeChanged` subscription is also re-registered on every
`aboutToAppear` and never removed.

**Text insertion and the preview — `KeyboardController.ets`, `InputHandler.insertText`** (corrected, see `HW-15-0096`)

```typescript
public insertText(text: string): void {
  if (this.mTextInputClient === undefined) {
    this.addLog('insertText this.mTextInputClient is undefined');
    return;
  }
  this.mTextInputClient.insertTextSync(text);

  let indexCursor: number = this.mTextInputClient.getTextIndexAtCursorSync();
  if (indexCursor >= 0) {
    // FIX: getForwardSync runs AFTER insertTextSync, so it already contains `text`.
    // The shipped code appends it a second time (typing 'hel' yields 'hell').
    this.intputText = this.mTextInputClient.getForwardSync(indexCursor);
  }

  let length = 2;
  let textPre: string = this.mTextInputClient.getForwardSync(length + text.length);
  if (textPre === 'hel') {
    try {
      let range: inputMethodEngine.Range = { start: indexCursor - textPre.length, end: indexCursor };
      this.mTextInputClient.setPreviewText(previewContent, range).catch((err: BusinessError) => {
        this.addLog(`insertText preViewText Failed to setPreviewText: ${err.code} ${err.message}`);
      });
    } catch (err) {
      this.addLog(`insertText preViewText Failed to setPreviewText: ${err.code} ${err.message}`);
    }
  }
}
```

**`setPreviewText` is the pre-commit mechanism** - text shown in the editor,
styled by the `'previewTextStyle': 'underline'` private command sent in
`onInputStart`, that is not committed until `finishTextPreview` (called from
`sendKeyFunction` when the enter key is pressed). It takes a `Range` in
*character indices of the editor's text*, which is exactly where the shipped
code goes wrong: it double-counts the freshly typed character, and elsewhere in
the same class `selectToEnd`/`moveCursorToEnd` pass `end: 1000` as a stand-in
for "the end" and fall back to `this.cursorInfo.x` - a pixel coordinate from
`cursorContextChange` - as a character index. Get the indices from
`getTextIndexAtCursorSync` and nothing else.

**Switching input method — `entry/src/main/ets/components/Submenu.ets`** (corrected, see `HW-15-0092`)

```typescript
async switchInputMethod(item: string) {
  this.inputMethods = await inputMethod.getSetting().getInputMethods(true); // enabled IMEs
  let currentInputMethod = inputMethod.getCurrentInputMethod();
  for (let i = 0; i < this.inputMethods.length; i++) {
    // FIX: shipped condition is `item != currentInputMethod.name`, which does not
    // depend on i at all - so it switches to EVERY enabled IME in turn.
    if (this.inputMethods[i].name === item && item !== currentInputMethod.name) {
      await inputMethod.switchInputMethod(this.inputMethods[i]);
      break;
    }
  }
}
```

The list beside it is built from `getInputMethods(true)` (enabled IMEs only)
and, for the current one, `listCurrentInputMethodSubtype()` - the two calls a
switcher needs. The subtype branch is correct as shipped:
`switchCurrentInputMethodSubtype` is awaited inside a `try/catch` and re-reads
`getCurrentInputMethodSubtype().id` afterwards instead of assuming success.
The `switchInputMethod` loop is the one to rewrite - as shipped its guard is
loop-invariant, so tapping any IME other than the current one walks the whole
enabled list and leaves the user on whichever happens to be last.

## Permissions & config

**None.** Neither scene declares `requestPermissions`. Being an input method is
authorised by the user enabling it in Settings, declared through the extension
type:

```json5
"extensionAbilities": [
  {
    "srcEntry": "./ets/InputMethodExtensionAbility/InputMethodService.ets",
    "name": "InputMethodService",
    "type": "inputMethod",
    "exported": true,
    "metadata": [
      { "name": "ohos.extension.input_method", "resource": "$profile:input_method_config" }
    ]
  }
]
```

`input_method_config.json` declares the two subtypes (`InputMethodExtAbility`
= `en-US`, `InputMethodExtAbility1` = `zh-CN`); their ids are the strings the
`setSubtype` listener matches on to flip `subtypeChange` in `AppStorage`, which
is how the page swaps `KeyMenu` for `NumberMenu`. `deviceTypes` is
`["default", "tablet"]`, and `main_pages.json` lists the panel page
(`InputMethodExtensionAbility/pages/Index`) alongside the host app's
`pages/PrivatePreview`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `adjustPanelRect(flag, EnhancedPanelRect)`, `setImmersiveMode` and
  `getImmersiveMode` are API 15 and later; the enhanced overload applies only
  to `SOFT_KEYBOARD` panels in `FLG_FIXED` or `FLG_FLOATING` state.
- `InputRegion` arrays are limited to 1-4 rectangles and default to the panel
  size; a fixed panel's avoid line may not sit more than 70% of the screen
  height above the bottom.
- There is no candidate engine. `lianxiang` is the concatenation of the letter
  keys pressed since the last backspace, rendered three times; `KeyItem`
  (symbols and caseless keys) never appends to it.
- The preview feature triggers on the literal string `hel` and inserts the
  literal `hello world`.
- Device adaptation is by exact display width/height pairs; unlisted displays
  fall back to a 0.35 height ratio.
- The zip contains two complete projects and both declare
  `bundleName: com.samples1.kikainput` at version `1.1.10`, so installing one
  replaces the other - the scenes cannot be compared side by side on one device.

## Pitfalls

- **`HW-15-0092`** (B/medium, confirmed): `switchInputMethod` compares the
  clicked name to the *current* IME - a condition that does not vary with the
  loop index - so it awaits `switchInputMethod` for every enabled IME in turn
  and lands on the last one. Fix: match `this.inputMethods[i].name === item`
  and `break`.
- **`HW-15-0093`** (B/medium, confirmed): the immersive-mode feature is dead.
  `Index`'s `panel` field is never assigned, so
  `this.panel?.setImmersiveMode(...)` is a no-op; the real panel lives in
  `KeyboardController`. The `editorAttributeChanged` listener is also
  re-registered per `aboutToAppear` with no matching `off`. Fix: route through
  the controller's panel and unsubscribe.
- **`HW-15-0094`** (B/medium, probable): scene 2 calls `panel.resize(100, 100)`
  after `adjustPanelRect`, discarding the computed `dWidth`/`keyHeight`, and
  hardcodes hot-zone and avoid-line rectangles (`1613`, `1200`, `300/650/2000x500`)
  for a single display. Both scenes also assign the 0.35
  `KEYBOARD_HEIGHT_RATE_DEFAULT` ratio to the pixel variable
  `navigationBar_height` in the RK branch. Doc 45 repeats the
  `resize(100, 100)`. Fix: derive every rectangle from the display metrics.
- **`HW-15-0095`** (B/medium, confirmed): listener on/off pairing is broken.
  `off('inputStop')` is passed a fresh closure rather than the registered
  callback, `cursorContextChange` is only removed under `isDebug`, and
  `'inputStart'`, `'setSubtype'`, `'privateCommand'` and `display.on('change')`
  are never removed at all - so a destroy/recreate cycle stacks handlers and
  `resizePanel` runs once per past incarnation. Fix: store each callback and
  remove it symmetrically.
- **`HW-15-0096`** (B/medium, probable): `insertText` reads the surrounding
  text *after* `insertTextSync` and then appends the character again, so typing
  `hel` produces `hell` internally and the preview `Range` end exceeds the text
  length. `selectToEnd`/`moveCursorToEnd` compound it with a hardcoded index
  `1000` and a pixel `cursorInfo.x` used as a character index. Fix: read before
  inserting (or do not re-append) and use real indices.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethodengine.md` -
  `createPanel`, `Panel.resize`/`moveTo`/`setUiContent`, `adjustPanelRect`,
  `EnhancedPanelRect`, `setImmersiveMode`, `setPreviewText`, `finishTextPreview`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethodengine
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod.md` -
  `getSetting().getInputMethods`, `switchInputMethod`,
  `listCurrentInputMethodSubtype`, `switchCurrentInputMethodSubtype`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod-subtype.md` -
  `InputMethodSubtype` and the profile fields
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod-subtype
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod-extension-ability.md` -
  the extension ability lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod-extension-ability
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-i.md` - `window.Rect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-i
- `documentation/harmonyos-guides/03_application-framework/ime-kit-intro.md` - IME Kit overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ime-kit-intro
- `documentation/harmonyos-guides/03_application-framework/inputmethod-application-guide.md` -
  building an input method application end to end
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/inputmethod-application-guide
- `UTIL-13` - the other side of immersive keyboards: an *application* telling
  the input method to match its page with `keyboardAppearance`
- `UTIL-15` - focus handling in the editor the keyboard types into
