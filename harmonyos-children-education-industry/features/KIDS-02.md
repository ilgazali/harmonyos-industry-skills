---
id: KIDS-02
title: Parent gate - an arithmetic challenge dialog in front of the settings page
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/02_children_demo_caculater.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_caculater-0000002235299782
sample: huawei_industry_tree/08_children_education/downloads/demo_Caculater.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["cryptoFramework.createRandom", "Random.generateRandomSync", "@CustomDialog", CustomDialogController, Tabs, TabContent, Navigation, NavPathStack, pushPathByName, NavDestination, "@StorageLink", "@Watch", "@Provide", "@Consume", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "UIContext.px2vp", Grid, Swiper]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0001, HW-08-0002, HW-08-0003, HW-08-0004, HW-08-0005, HW-08-0006, HW-08-0007, HW-08-0008, HW-08-0009, HW-08-0010, HW-08-0011, HW-08-0012, HW-08-0013, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card when a screen must be **reachable by an adult but not by the
child holding the device** - settings, an in-app purchase, disabling the
screen-time limit, a parent mode. The mechanism is a challenge the child
cannot answer: a randomly generated arithmetic expression whose answer must be
typed on a digit pad before the route is pushed.

Two ideas are worth taking:

- **The gate is a `@CustomDialog` that never returns a verdict.** It writes
  the typed digits into `AppStorage`, and the page watches that key with
  `@Watch`. When the accumulated string equals the answer, the page closes the
  dialog and navigates. No callback, no result parameter.
- **The expression is generated, not stored.** A small evaluator builds
  `n op n op n =`, reduces it by precedence, and returns the expression string
  and the answer string together, so there is nothing on disk for a curious
  child to find.

**Read the Pitfalls before copying anything.** Thirteen findings on a
seven-file sample: the generator can hang the app, the tab highlight never
moves, and the navigation parameter is read under the wrong route name. This
is the least carefully written sample in the industry.

## Feature checklist

- A profile tab with a settings link in its header.
- Tapping it generates a fresh expression and opens the challenge dialog.
- The dialog shows the expression, a read-only answer field and a ten-key pad.
- Each digit appends to the answer; a match opens the settings page and the
  dialog closes.
- A close button dismisses the dialog and clears the input.
- The whole app is laid out full-screen under the status and navigation bars.

## Architecture

One `entry` module, two pages, one dialog, three files of logic.

```
entry/src/main/ets
├── common
│   ├── Constants.ets          digit and symbol alphabets, sizes
│   ├── Objects.ets            Calculator, doRandBySync, Movie, Param
│   └── Utils.ets              CalculateUtil: generate, reduce, checkAbs
├── entryability/EntryAbility.ets   immersive layout, avoid-area heights
├── entrybackupability/             not in the documented tree (HW-08-0012)
├── pages
│   ├── Profile.ets            Tabs > TabContent > Navigation, the dialog owner
│   └── Setting.ets            the gated NavDestination
└── views/DialogConfirm.ets    the @CustomDialog: expression, field, keypad
```

**Tabs outside, Navigation inside.** The document states the nesting and the
sample repeats the reason in a comment: the `Navigation` goes inside a
`TabContent`, not the other way round, or the tab bar does not render. This
matters because both are full-screen containers competing for the same
viewport, and it is the one structural decision the sample gets right.

**The verification loop runs through `AppStorage`, not through a callback:**

```
Profile                                    DialogConfirm (@CustomDialog)
  AppStorage 'inputData' ──@StorageLink──> @Link inputData
        │                                        │
        │                              digit tapped: inputData += num
        │                                        │
   @Watch('watchInputData') <────────────────────┘
        │
   inputData === resultData ?
        │ yes
   dialogController.close()
   navPathStack.pushPathByName('Setting', ...)
   setTimeout(() => inputData = '', 1000)
```

The dialog is constructed with `$data`, `$resultData` and `$inputData`, so all
three are `@Link`-bound and the dialog is a pure view. The decision lives in
the page.

**The avoid-area heights travel through `AppStorage` too.** `EntryAbility`
reads them once with `getWindowAvoidArea` and then keeps them current with
`on('avoidAreaChange')`; both pages read them with `@StorageLink` and convert
with `px2vp`. That is the documented immersive-layout pattern and it is the
second thing this sample gets right.

## Implementation steps

1. **Go full screen in `onWindowStageCreate`**, read both avoid areas into
   `AppStorage`, and register `avoidAreaChange` - then release it in
   `onWindowStageDestroy` (`HW-08-0008`).
2. **Generate the expression** with a secure random source, drawing each value
   from its own bytes (`HW-08-0002`), and **bound the retry loop**
   (`HW-08-0001`).
3. **Regenerate on every open**, so the same expression is never shown twice.
4. **Bind the dialog's three strings with `@Link`** and let the page own the
   comparison.
5. **Watch the input and compare on every keystroke**, then close and push.
6. **Decorate any field bound with `$$`** (`HW-08-0013`).
7. **Read navigation parameters under the name the route was pushed with**
   (`HW-08-0004`).

## Verified snippets

All snippets are from `demo_Caculater.zip`. Corrected forms are marked.

**The gate itself — `entry/src/main/ets/pages/Profile.ets`** (as shipped)

```typescript
AppStorage.setOrCreate('inputData', '');

@Entry
@Component
struct Profile {
  @Provide navPathStack: NavPathStack = new NavPathStack();
  @State data: string = '';                    // the expression, e.g. "3  x  4  +  2  ="
  @State resultData: string = '';              // the answer, e.g. "14"
  @StorageLink('inputData') @Watch('watchInputData') inputData: string = '';

  dialogController: CustomDialogController = new CustomDialogController({
    builder: CustomDialogExample({ data: $data, resultData: $resultData, inputData: $inputData }),
    autoCancel: false,                         // tapping outside must not dismiss the gate
    customStyle: true,
    backgroundColor: Color.Transparent,
  });

  watchInputData(): void {
    if (this.inputData === this.resultData) {
      this.dialogController.close();
      this.navPathStack.pushPathByName('Setting', ...);
      setTimeout(() => { this.inputData = ''; }, 1000);   // clear after the dialog is gone
    }
  }
}
```

**`autoCancel: false` is the security-relevant line.** The default is `true`,
which dismisses a dialog on a tap outside it - here that would leave the child
back on the profile page having learned nothing, which is fine, but it also
means the gate could be dismissed accidentally mid-entry. Setting it false
forces the explicit close button.

**Comparing on `@Watch` rather than on a submit button** is what makes the pad
feel instant: there is no confirm key, the dialog closes the moment the last
digit lands. The cost is that there is no backspace either - the pad has ten
digits and a close button, so a mistyped digit can only be undone by closing
and reopening.

**The delayed clear is deliberate.** `inputData` is reset a second after the
dialog closes rather than immediately, so the correct answer is not wiped out
from under the dialog while it is still animating away.

**Generating the expression — `entry/src/main/ets/common/Utils.ets`** (corrected, see `HW-08-0001` and `HW-08-0006`)

```typescript
static checkAbs(): [data: string, result: string] {
  // FIX: the sample loops `while (true)` with no bound, and a failing
  // doRandBySync returns -1, which can never satisfy the exit test.
  const MAX_ATTEMPTS = 20;
  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
    const dataResult = CalculateUtil.getRandomListAndResult();
    if (parseInt(dataResult[1]) > 0) {
      return dataResult;
    }
  }
  return ['2  +  3  =', '5'];                  // deterministic fallback
}

static checkHaveMulAndDivAndExe(list: Calculator[]): [Calculator[], string] {
  let data: string = '';
  while (list.some((calculator: Calculator) => calculator.type === arrType[1])) {
    const step = CalculateUtil.mulAndDiv(list);   // FIX: the sample calls this twice
    list = step[0];
    data = step[1];
  }
  return [list, data];
}
```

**The expression is only ever accepted when positive** -
`parseInt(resultData) > 0` - because a negative answer would need a minus key
the pad does not have. That is the right constraint; it just needs a bound,
since it is enforced by rejection rather than by construction.

**The precedence pass is two sweeps**, multiplication before addition, each
reducing one operator at a time and rebuilding the list around it. With three
operands and two operators it converges in at most two steps.

**A secure digit — `entry/src/main/ets/common/Objects.ets`** (corrected, see `HW-08-0002`)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

// FIX: the sample requests 24 bytes and then does
//   parseInt(randData.data.toString()) % num
// data is a Uint8Array, so toString() is "203,17,88,..." and parseInt reads
// only the first byte - 23 bytes are discarded and % introduces modulo bias.
export function doRandBySync(num: number): number {
  const rand = cryptoFramework.createRandom();
  const limit = Math.floor(256 / num) * num;         // reject the biased tail
  while (true) {
    const randData = rand.generateRandomSync(1);
    if (randData.data.length === 0) {
      throw new Error('random generation failed');   // not a -1 the caller cannot detect
    }
    const byte = randData.data[0];
    if (byte < limit) {
      return byte % num;
    }
  }
}
```

**Using `cryptoFramework` rather than `Math.random()` is the right instinct**
for a gate, even a soft one: the expression must not be predictable from the
app's own state. The mistake is in how the bytes are consumed, not in the
choice of source.

**Immersive layout — `entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-08-0008`)

```typescript
onWindowStageCreate(windowStage: window.WindowStage): void {
  const windowClass: window.Window = windowStage.getMainWindowSync();
  windowClass.setWindowLayoutFullScreen(true)
    .then(() => { /* logged */ })
    .catch((err: BusinessError) => { /* logged */ });

  // 1. read both avoid areas once
  let avoidArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);
  avoidArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  // 2. keep them current
  try {
    windowClass.on('avoidAreaChange', this.avoidAreaCallback);   // FIX: sample has no try/catch
  } catch (exception) { /* 401 */ }

  windowStage.loadContent('pages/Profile', (err) => { /* ... */ });
}

onWindowStageDestroy(): void {
  // FIX: the sample only logs here
  this.windowClass?.off('avoidAreaChange', this.avoidAreaCallback);
}
```

**Reading once and then subscribing is both halves of the pattern.** The
one-shot read gives the first frame correct padding; the subscription covers
rotation, multi-window and a keyboard appearing. Doing only the read leaves
the layout stale, doing only the subscription leaves the first frame wrong.

**The heights are in px and the padding is in vp**, hence `px2vp` at both use
sites - `padding({ top: this.uiContext!.px2vp(this.topRectHeight) })`. Mixing
those two up is the usual reason immersive padding looks right on one device
and wrong on the next.

**The tab bar — `entry/src/main/ets/pages/Profile.ets`** (corrected, see `HW-08-0013`)

```typescript
@State tabindex: number = 1;      // FIX: the sample declares `tabindex: number = 1;`

build() {
  Tabs({ barPosition: BarPosition.End, index: $$this.tabindex, controller: this.tabsController }) {
    TabContent() { /* ... */ }.tabBar(this.tabbarBuilder(0, 'Home'));

    TabContent() {
      // the sample's comment: Tabs must be set before Navigation or the tab bar does not show
      Navigation(this.navPathStack) { /* ... */ }
    }.tabBar(this.tabbarBuilder(1, 'Mine'));
  }
}
```

`$$` on `Tabs.index` is supported, but the guide is explicit that the UI only
updates when the bound variable carries a state decorator - and the tab bar
builder reads `this.tabindex` to colour the active label.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The gate is entirely
local: no network, no storage, no account.

Two profiles matter:

- `main_pages.json` lists `pages/Profile` only - the settings page is not a
  page but a `NavDestination`.
- `router_map.json` registers the gated route:

  ```json
  {
    "routerMap": [
      {
        "name": "Setting",
        "pageSourceFile": "src/main/ets/pages/Setting.ets",
        "buildFunction": "SettingBuilder",
        "data": { "description": "Setting" }
      }
    ]
  }
  ```

  The name here is what `pushPathByName` and `getParamByName` must both use,
  which is the mismatch behind `HW-08-0004`.

`EntryAbility.onCreate` pins the app to light mode with
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, which is why
the hardcoded white and grey backgrounds throughout the pages do not break -
and also why the `resources/dark` colours in the project are never used.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The gate is a speed bump, not a security boundary.** The expression and
  its answer are both in component state, the comparison is a string equality
  in the page, and nothing is verified against a credential. It stops a child
  who cannot do arithmetic; it does not stop one who can, and an older child
  is exactly the one who will try.
- **The difficulty is fixed in code.** `numberOfNum = 3` in
  `CalculateUtil.getList` gives three operands and two operators, and the
  operator pool is one multiplication sign plus plus and minus. There is no
  division and no setting to raise or lower the level for the child's age.
- **The keypad has no backspace.** Ten digits and a close button; a mistyped
  digit means closing the dialog and starting again with a new expression.
- The first tab is an empty blue `Column` - only the profile tab is built out.
- The app is pinned to light mode, so the `dark` colour resources are dead.

## Pitfalls

- **`HW-08-0001` — `checkAbs` is an unbounded `while (true)` on the UI
  thread,** and `doRandBySync` returns `-1` on failure. With `-1` every
  operator slot becomes a number, no reduction happens, the answer is `-1`,
  and the loop restarts forever. The entry page never finishes
  `aboutToAppear`, so the app freezes with nothing drawn.
- **`HW-08-0013` — `tabindex` is bound with `$$` but carries no decorator,**
  so switching tabs updates the field and re-renders nothing. The highlight
  stays on the tab the app started on for the whole session.
- **`HW-08-0002` — the random helper asks for 24 bytes and uses one.**
  `Uint8Array.toString()` is comma-separated, so `parseInt` stops at the first
  comma; the remaining 23 bytes are discarded and `% 10` biases digits 0-5.
- **`HW-08-0004` — `Setting` calls `getParamByName('Index')` for a route
  pushed as `'Setting'`,** and the `Param` it would receive is built from two
  fields that are never assigned. Silent because the field is never read.
- **`HW-08-0005` — each carousel slide is wrapped in a `ListItem`,** whose
  reference states its parent can only be `List` or `ListItemGroup`.
- **`HW-08-0006` — each reduction step is evaluated twice,** and the string
  returned beside the list describes a list one step further on. It survives
  only because the caller overwrites the value immediately afterwards.
- **`HW-08-0007` — `@Provide uiContext: UIContext | undefined` is consumed as
  `@Consume uiContext: UIContext`.** The guide requires identical types.
  Neither side is needed: `this.getUIContext()` works in any component.
- **`HW-08-0008` — the `avoidAreaChange` listener is never released,** and its
  registration is the one window call in the file without a `try`/`catch`.
- **`HW-08-0003` — `Rect` is imported from
  `@ohos.application.AccessibilityExtensionAbility`** rather than
  `@kit.AccessibilityKit`, for an accessibility-framework type used by two
  fields that never hold a value.
- **`HW-08-0009` — the keypad iterates a `string[]` with the item typed
  `number`.** `inputData += num` concatenates, which is what the gate needs,
  so the code works only because the annotation is wrong.
- **`HW-08-0011` — the divide-by-zero guard tests for `'/'`,** which the
  generator never emits; the division branch it protects is equally
  unreachable, and enabling division the obvious way would leave the guard
  still not firing.
- **`HW-08-0010` — eight user-visible labels are Chinese literals** next to
  `$r` images in the same builder calls, while `string.json` holds the rest.
- **`HW-08-0012` — the documented project tree omits `entrybackupability`,**
  which the sample ships and `module.json5` registers.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-common-components-custom-dialog.md` - `@CustomDialog` and `CustomDialogController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-components-custom-dialog
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `autoCancel`, `customStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - the `$$` rules and the components that accept it
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - the same-type requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageLink` and `@Watch`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md`, `ts-basic-components-navigation.md` - the two containers and their nesting
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` - the parent constraint behind `HW-08-0005`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - `generateRandomSync` and the guard the guide uses
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `DataBlob.data` is a `Uint8Array`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/03_application-framework/arkts-develop-apply-immersive-effects.md` - the full-screen and avoid-area pattern
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-develop-apply-immersive-effects
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-04` - the industry's other parental-control sample, which locks the app on a time budget
