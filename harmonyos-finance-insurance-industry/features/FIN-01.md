---
id: FIN-01
title: Insurance app - layered architecture with a secure keyboard and card recognition
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/01_practice-insurance-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-insurance-app-architecture-v1-0000001938013084
sample: huawei_industry_tree/07_finance_insurance/downloads/InsuranceDemo.zip
kits: ["@kit.ArkUI", "@kit.VisionKit", "@kit.AbilityKit", "@kit.LocationKit", "@kit.PerformanceAnalysisKit"]
apis: [TextInput.customKeyboard, TextInputController.stopEditing, CardRecognition, CardType, CardSide, CardRecognitionResult, Window.setWindowPrivacyMode, window.getLastWindow, NavPathStack.pushPathByName, NavPathStack.popToName, geoLocationManager.getCurrentLocation, geoLocationManager.getAddressesFromLocation, "@Provide", "@Consume", Grid, GridItem]
permissions: [ohos.permission.PRIVACY_WINDOW, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [products/phone, commons/basic, commons/dfx, commons/jsbridge, commons/router, features/account, features/insurance, features/mine, features/service, features/tools]
findings: [HW-07-0001, HW-07-0002, HW-07-0003, HW-07-0004, HW-07-0005, HW-07-0006, HW-07-0007, HW-07-0008, HW-07-0009, HW-07-0010]
status: verified-with-fixes
---

## When to use

Load this card when building an **insurance, banking or any app that collects
identity documents and credentials**: policy management, claims, product
catalogue, KYC onboarding. It defines the module split and the two capabilities
that make this industry distinct - a **custom secure keyboard** and **ID/bank
card recognition**.

More than any other card in this corpus, this one is about **what not to do with
the data once you have it**. Six of its ten findings are security or privacy
defects, and they are the kind that pass code review and fail a privacy audit.

## Feature checklist

- Four bottom tabs: home, insurance, service, mine.
- Insurance: product catalogue with tabbed categories and a promotions carousel.
- Service: a hub of H5 pages reached through a JS bridge.
- Mine: orders, policies, claims, assets, wallet, settings.
- **Login through a custom keyboard**, never the system IME.
- **ID card scanning** that fills in the number and name automatically.
- Screenshot protection on screens showing credentials.

## Architecture

Three layers, ten modules, one IDE project. The product layer is its own
directory tree (`products/phone`), matching the layered-modularisation practice.

| Layer | Modules | Purpose |
|---|---|---|
| Product customisation | `products/phone` | entry HAP: splash, login, tab host, home |
| Basic feature | `features/{account,insurance,mine,service,tools}` | HAR per business area |
| Common capability | `commons/{basic,dfx,jsbridge,router}` | tools, telemetry, H5 bridge, routing |

Splitting the common layer into **four separate HARs** rather than one is the
notable choice here: routing, DFX and the JS bridge each get their own module,
so a feature that needs routing does not pull in the web bridge.

**Permissions are declared only in `products/phone`** - the nine other manifests
carry no `requestPermissions` block at all. That is the correct arrangement, and
worth contrasting with the transit practice where all nine modules redeclare
them.

`@Provide('topRectHeight')` in `MainPage` publishes the status-bar inset to
every page. Note the type disagreement across consumers (`HW-07-0004`).

## Implementation steps

1. **Declare only the permissions you request** (`HW-07-0003`).
2. **Bind a custom keyboard to sensitive inputs** with
   `TextInput().customKeyboard(...)`.
3. **Keep the key data outside the component** as a constants array, and give
   `ForEach` a stable key (`HW-07-0007`).
4. **Enable privacy mode on every screen that shows credentials**, not just
   login (`HW-07-0002`), and check that it succeeded (`HW-07-0006`).
5. **Use `CardRecognition` from Vision Kit** for ID and bank cards - it needs no
   camera permission of your own.
6. **Handle every result code**, not only success (`HW-07-0005`).
7. **Never log the recognised values** (`HW-07-0001`).

## Verified snippets

All snippets are from `InsuranceDemo.zip`. Corrected forms are marked.

**Binding a secure keyboard — `features/account/.../LoginComponent.ets`** (as shipped)

```typescript
TextInput({ text: this.inputValue, controller: this.controller })
  .customKeyboard(this.SecureKeyBoardWindow());
```

`customKeyboard` replaces the system IME entirely for that input, which is the
whole point: a third-party keyboard cannot observe what is typed. Close it from
inside the keyboard with `this.controller.stopEditing()`.

**The keyboard component — `features/account/.../CustomKeyboard.ets`** (corrected, see `HW-07-0007`)

```typescript
@Component
export struct CustomKeyboard {
  @Prop items: IKeyAttribute[];
  @Prop curKeyboardType: EKeyboardType;
  @Link inputValue: string;
  public controller: TextInputController = new TextInputController();
  public onKeyboardEvent: Function | null = null;

  build() {
    Column() {
      this.titleBar();
      Grid() {
        ForEach(this.items, (item: IKeyAttribute) => {
          GridItem() {
            this.myGridItem(item);
          }
          .rowStart(item?.position?.[0])      // grid position from the data
          .rowEnd(item?.position?.[1])
          .columnStart(item?.position?.[2])
          .columnEnd(item?.position?.[3])
          .onClick(() => {
            this.onKeyboardEvent?.(item);
          });
        }, (item: IKeyAttribute) => item.value ?? item.type);   // FIX: shipped key is
                                                                // JSON.stringify(item) + index
      }
      .columnsTemplate(this.curKeyboardType === EKeyboardType.NUMERIC
        ? '1fr 1fr 1fr'
        : '1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr 1fr')
      .rowsTemplate('1fr 1fr 1fr 1fr')
      .rowsGap(this.rowSpace)
      .columnsGap(this.columnSpace);
    }
    .backgroundColor(Color.Black);
  }
}
```

**The layout trick worth taking**: a **20-column grid** for the alphabetic and
symbol keyboards, and 3 columns for the numeric one. Twenty divides evenly for a
ten-key row (each key spans 2) *and* leaves room for wide keys like shift and
backspace to span 3, so every key lands on an integer cell boundary. Each key
carries its own `[rowStart, rowEnd, columnStart, columnEnd]` in the data, so the
layout is data-driven rather than hand-positioned.

The event flow is worth copying too: the parent owns `onKeyboardEvent` and the
child calls back into it; `inputValue` is `@Link` so both ends see the same
string.

```typescript
onKeyboardEvent(item: IKeyAttribute) {
  switch (item.type) {
    case EKeyType.INPUT:    this.inputValue += item.value; break;
    case EKeyType.DELETE:   this.inputValue = this.inputValue.slice(0, -1); break;
    case EKeyType.NUMERIC:  /* swap this.items to numericKeyData */ break;
    case EKeyType.CAPSLOCK: /* swap to upper/lowerCaseKeyData */ break;
    case EKeyType.SPECIAL:  /* swap to specialKeyData */ break;
  }
}
```

Swapping the whole `items` array to switch keyboards - rather than mutating keys
in place - is the right mechanism. The shipped code does both, and the in-place
mutation is what forces the `JSON.stringify` key.

**ID card recognition — `features/mine/.../CredentialsPage.ets`** (corrected, see `HW-07-0005`, `HW-07-0009`)

```typescript
import { CardRecognition, CardType, CardSide, CardRecognitionResult } from '@kit.VisionKit';

CardRecognition({
  supportType: CardType.CARD_ID,      // CARD_BANK for bank cards, AUTO to detect
  cardSide: CardSide.FRONT,
  callback: ((params: CardRecognitionResult) => {
    if (params.code !== 200) {
      // FIX: shipped code ignores every non-200 code, stranding the user on the scanner
      this.pathStack.pop();
      return;
    }
    const front = params.cardInfo?.front;
    if (!front?.idNumber || !front?.name) {   // FIX: shipped code passes undefined onward
      this.pathStack.pop();
      return;
    }
    this.pathStack.popToName('AuthenticationPage', new Info(front.idNumber, front.name));
  })
})
```

`CardRecognition` is a **full-screen component**, not a function call: you place
it in the view tree and it takes over with its own camera UI. That is why the
sample declares **no camera permission** - Vision Kit runs the capture in its
own secure context and hands back only the parsed fields.

Returning the result by `popToName(...)` with a payload, picked up by the
caller's `pushPathByName` callback, is the clean way to get data back from a
scanner page.

**Screenshot protection — `commons/basic/.../WindowUtils.ets`** (corrected, see `HW-07-0006`)

```typescript
// FIX: the shipped version returns void and handles neither promise
static async setWindowPrivacyModeInPage(context: common.UIAbilityContext,
  isFlag: boolean): Promise<boolean> {
  try {
    const lastWindow = await window.getLastWindow(context);
    await lastWindow.setWindowPrivacyMode(isFlag);
    return true;
  } catch (err) {
    Logger.error(`setWindowPrivacyMode failed: ${JSON.stringify(err)}`);
    return false;
  }
}
```

Applied per page (corrected, see `HW-07-0002`):

```typescript
aboutToAppear() {
  WindowUtils.setWindowPrivacyModeInPage(ctx, true);
}
aboutToDisappear() {
  WindowUtils.setWindowPrivacyModeInPage(ctx, false);   // release it on the way out
}
```

**Apply this to every screen showing credentials**, not only login. The shipped
sample protects the login form and leaves the ID-number screen and the live
card-scanning camera unprotected. Privacy mode blocks screenshots, screen
recording **and** the recent-tasks thumbnail - which is the exposure people
forget.

Releasing it in `aboutToDisappear` matters: the flag is on the window, not the
page, so leaving it set would silently disable screenshots app-wide.

## Permissions & config

`products/phone/src/main/module.json5` - and nowhere else:

```json5
"requestPermissions": [
  { "name": "ohos.permission.APPROXIMATELY_LOCATION", ... },
  { "name": "ohos.permission.PRIVACY_WINDOW", ... }
]
```

The shipped file also declares `ohos.permission.LOCATION`, which nothing ever
requests (`HW-07-0003`).

**No camera permission is needed** for `CardRecognition` - Vision Kit owns the
capture.

`compatibleSdkVersion: "6.0.0(20)"`.

## Constraints

- DevEco Studio 6.0.0 Release or later; HarmonyOS 6.0.0 Release SDK or later.
- **The login performs no authentication.** The document states it: eleven
  digits and any password gets you in. Replace it before shipping anything.
- `CardRecognition` supports ID cards and bank cards; the sample wires only the
  front of an ID card.
- The service tab is largely H5 behind a JS bridge, so much of the product
  surface is not ArkTS at all.
- The document gives three different device scopes and the manifest a fourth
  (`HW-07-0008`).

## Pitfalls

- **`HW-07-0001` — the ID number and real name are written to the log.** With
  `'test ABC'` prefixes, an empty tag, and concatenated into the format string so
  they carry no `%{private}s` marking and are never redacted. The user's location
  and civic address are logged in full elsewhere.
- **`HW-07-0002` — privacy mode protects the login form, not the ID screens.**
  The screen rendering the ID number and the live card-scanning camera are both
  screenshottable and both appear in the task thumbnail.
- **`HW-07-0003` — `ohos.permission.LOCATION` is declared and never requested,**
  so the app advertises precise location it does not use.
- **`HW-07-0004` — `@Provide` publishes a number, six `@Consume`s declare a
  string.** One consumer gets it right, which shows the type was known.
- **`HW-07-0005` — only result code 200 is handled.** Cancel, failure and
  permission problems all leave the user on the scanner with no message.
- **`HW-07-0006` — a failure to enable screenshot protection is invisible.**
  Neither promise in `WindowUtils` is handled, so the control fails open and
  silent.
- **`HW-07-0007` — `JSON.stringify(item) + index` keys the keyboard grid,** and
  the click handler mutates `item.label` in place, so the caps-lock redraw works
  only as a side effect of the bad key.
- **`HW-07-0008` — four different answers to which devices are supported.**
- **`HW-07-0009` — `string | undefined` card fields are passed into `string`
  parameters,** so a partial recognition renders the literal `undefined`.
- **`HW-07-0010` — the privacy notice reads 同事 where it means 同时,** in the
  sentence promising the user their data will be protected.

## References

- `documentation/harmonyos-guides/08_ai/vision-cardrecognition.md` - `CardRecognition`, card types, result codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/vision-cardrecognition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `customKeyboard`, `TextInputController.stopEditing`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowPrivacyMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` keys and why not to serialise the item
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - declaring only what is requested
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `FIN-09` - the randomised-order keyboard; this one has a fixed layout
- `FIN-06` - app lock, the other security practice in this industry
