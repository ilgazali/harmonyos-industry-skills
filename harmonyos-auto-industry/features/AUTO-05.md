---
id: AUTO-05
title: One-tap dialling from a half-modal contact sheet
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/05_call_phone.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
sample: huawei_industry_tree/01_auto/downloads/CallPhone.zip
kits: ["@kit.TelephonyKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [call.makeCall, bindSheet, resourceManager.getStringSync, PromptAction.showToast, "@Provide", "@Consume", "@StorageLink"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-01-0031, HW-01-0032, HW-01-0033, HW-01-0034, HW-01-0035, HW-01-0050]
status: verified-with-fixes
---

## When to use

Load this card whenever the app has to **hand a phone number to the system
dialler** - dealership contact, roadside assistance, customer service, a store
listing. It also covers the surrounding pattern: a half-modal sheet listing
several numbers, each row tappable.

The important point is small and easy to get wrong in the other direction:
**`call.makeCall` needs no permission.** It opens the dialler with the number
pre-filled and lets the user press the call button. Do not request
`ohos.permission.PLACE_CALL` for this.

## Feature checklist

- A trigger (a contact button or a store row) that opens a half-modal sheet.
- The sheet lists two or more numbers with an icon and a label per row.
- Tapping a row opens the system dialler with that number shown.
- A close control on the sheet, plus dismissal by swipe.
- Numbers held as string resources, not hardcoded.

## Architecture

Single `entry` module, two components.

```
entry/src/main/ets
├── component/PhoneSheet.ets   the half-modal content (@Builder + @Component)
├── entryability/EntryAbility.ets
├── pages/CallPhone.ets        the page; owns the sheet's visibility flag
└── util/
    ├── Logger.ets
    └── ResourceToStr.ets      Resource -> string helper
```

Visibility is shared with `@Provide('isShowSheet')` on the page and
`@Consume('isShowSheet')` in the sheet component, so the sheet can close itself
without a callback round-trip. The sheet content is supplied to `bindSheet` as a
**global `@Builder` function** (`phoneSheet`) that instantiates the real
`@Component` (`PhoneSheetItem`) - that indirection is what lets a builder be
passed as a sheet body.

> The sample also threads a `Tmp { cancel, confirm }` object into the sheet.
> Nothing ever calls those closures - see `HW-01-0032`. The `@Provide` /
> `@Consume` flag is the mechanism that actually works.

## Implementation steps

1. **Declare the page with both decorators**: `@Entry` **and** `@Component`
   (`HW-01-0031`).
2. **Own the flag on the page** with `@Provide('isShowSheet')`.
3. **Write the sheet body as a global `@Builder` function** that instantiates an
   `@Component` struct, and have that struct take the flag with `@Consume`.
4. **Attach with `bindSheet(flag, builder, options)`**, and hook `onDisappear`
   so a swipe-dismiss resets the flag - without it the sheet cannot be reopened.
5. **Resolve the numbers once**, not inside `build()` (`HW-01-0033`).
6. **Call `call.makeCall(number, callback)` and inspect `err` in the callback** -
   it is the only channel the API reports failure on (`HW-01-0034`).

## Verified snippets

All snippets are from `CallPhone.zip`. Corrected forms are marked.

**Dialling — `component/PhoneSheet.ets`** (as shipped)

```typescript
import { call } from '@kit.TelephonyKit';
import { BusinessError } from '@kit.BasicServicesKit';
import Logger from '../util/Logger';

.onClick(() => {
  call.makeCall(number, (err: BusinessError) => {
    if (err) {
      Logger.error(`makeCall fail, err->${JSON.stringify(err)}`);
    } else {
      Logger.info(`makeCall success`);
    }
  });
});
```

This is the whole feature. `makeCall` launches the call screen showing the
number; it does **not** place the call, which is why no permission is involved.
The reference notes it **can only be called from a UIAbility** - it will not
work from a background service or an extension ability.

The document prints this same block with an empty callback body (`HW-01-0034`).
Keep the `err` check: the `AsyncCallback` is the only place a failure surfaces.

**Sheet body as a global builder — `component/PhoneSheet.ets`** (as shipped)

```typescript
interface Tmp {
  cancel: () => void;
  confirm: () => void;
}

@Builder
export function phoneSheet(tansParams: Tmp) {
  PhoneSheetItem({ params: tansParams });
}

@Component
export struct PhoneSheetItem {
  @Consume('isShowSheet') isShowSheet: boolean;
  params: Tmp = { cancel: () => {}, confirm: () => {} };

  build() {
    Column() {
      Row() {
        Text($r('app.string.phone')).fontSize(21).fontWeight(700);
        Image($r('app.media.ic_close'))
          .width(40)
          .onClick(() => {
            this.params.cancel();      // FIX: shipped code only sets the flag
            this.isShowSheet = false;
          });
      }
      .width('100%')
      .justifyContent(FlexAlign.SpaceBetween);
      // ... rows
    }
    .width('100%')
    .padding(16);
  }
}
```

The `@Builder` wrapper around a `@Component` is the reusable part: `bindSheet`
takes a builder, but you want a real component so it can hold `@Consume` state
and its own `@Builder` sub-parts.

**Attaching the sheet — `pages/CallPhone.ets`** (corrected, see `HW-01-0031` and `HW-01-0035`)

```typescript
@Entry
@Component                    // FIX: the shipped page omits @Component
struct CallPhone {
  @Provide('isShowSheet') isShowSheet: boolean = false;
  @StorageLink('topRectHeight') topRectHeight: number = 0;
  @StorageLink('bottomRectHeight') bottomRectHeight: number = 0;

  sheetCancel() { this.isShowSheet = false; }
  sheetConfirm() { this.isShowSheet = false; }

  // ... inside build():
  .bindSheet(this.isShowSheet, phoneSheet({
    cancel: () => {
      this.sheetCancel();
    },
    confirm: () => {          // FIX: shipped code marks this async for a sync body
      this.sheetConfirm();
    }
  }), {
    height: 240,
    showClose: false,
    backgroundColor: 'rgb(241, 243, 245)',
    onDisappear: () => {
      this.sheetCancel();     // essential: resets the flag after a swipe-dismiss
    }
  });
}
```

`onDisappear` is not optional here. `bindSheet`'s first argument is passed
one-way, so a swipe-down closes the sheet without writing `false` back to the
flag; the next tap on the trigger would set a value that is already `true` and
nothing would happen. Either hook `onDisappear` as above, or bind two-way with
`$$this.isShowSheet` as the petrol-station practice does.

**Resource to string — `util/ResourceToStr.ets`** (as shipped)

```typescript
export function getResourceString(resource: Resource, context: Context): string {
  let result: string = '';
  try {
    result = context.resourceManager.getStringSync(resource.id);
  } catch (error) {
    Logger.error(`[getResourceString]getStringSync failed, error:${JSON.stringify(error)}.`);
  }
  return result;
}
```

Useful when you need the **value** of a string resource rather than a `Resource`
to hand to a component - here, the number that goes into `makeCall`. Call it
from `aboutToAppear` into state, not from `build()` (`HW-01-0033`):

```typescript
@State storePhone: string = '';

aboutToAppear(): void {
  const ctx = this.getUIContext().getHostContext() as common.UIAbilityContext;
  this.storePhone = getResourceString($r('app.string.store_phone_number'), ctx);
}
```

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`, and it
should not: `makeCall` only launches the dialler UI.

`deviceTypes`: `["phone", "tablet", "2in1"]` - consistent with the industry
architecture sample. The reference lists `makeCall` as available on Phone,
PC/2in1, Tablet and Wearable, so the declaration is within range.

Resources ship `base`, `dark`, `en_US` and `zh_CN`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `call.makeCall` **can only be called from a UIAbility**.
- The device must have telephony capability; on a tablet or 2in1 without a SIM
  the dialler may not exist, and the callback's `err` is where that shows up.
- The sheet's `height: 240` is a fixed value - it does not adapt to the number
  of rows or to a larger font scale.

## Pitfalls

- **`HW-01-0031` — the page is `@Entry` without `@Component`.** The
  custom-component guide requires a component definition to start with
  `@Component struct`, and this struct uses `@Provide`, `@StorageLink` and
  `build()`. `PhoneSheet.ets` in the same project gets it right.
- **`HW-01-0032` — the `cancel` / `confirm` callbacks are never called.**
  `params` is assigned into the component and never read, so `sheetConfirm()` is
  dead code. The sheet closes only by mutating the `@Consume`d flag.
- **`HW-01-0033` — `getStringSync` runs inside `build()`,** twice per render,
  while the same builder passes other `Resource` objects declaratively.
- **`HW-01-0034` — the document empties the `makeCall` error callback.** That
  callback is the only place `makeCall` reports failure, and the step-2 snippet
  is nothing but that call.
- **`HW-01-0035` — `confirm: async () => {…}` is assigned to a `() => void`
  slot** for a synchronous body, discarding a promise; its sibling `cancel` is a
  plain arrow.

## References

- `documentation/harmonyos-references/03_system/js-apis-call.md` - `call.makeCall`; import is `{ call } from '@kit.TelephonyKit'`, no permission required, UIAbility-only
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-call
- `documentation/harmonyos-guides/03_application-framework/arkts-create-custom-components.md` - `@Entry` / `@Component` structure rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-create-custom-components
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet` options, `onDisappear`, two-way binding with `$$`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `AUTO-04` - the petrol-station practice; its `bindSheet` uses `$$` two-way binding instead of the `onDisappear` reset used here
- `AUTO-01` - the industry architecture this screen drops into
