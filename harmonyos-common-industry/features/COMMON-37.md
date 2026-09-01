---
id: COMMON-37
title: Tap-the-characters-in-order human verification - a reusable custom dialog wrapper plus a TextTimer countdown
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/37_text_order_verification.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_order_verification-0000002347027492
sample: huawei_industry_tree/19_common_technical_solutions/downloads/TextOrderVerification.zip
kits: ["@kit.ArkUI", "@kit.BasicServicesKit"]
apis: ["PromptAction.openCustomDialog", "PromptAction.closeCustomDialog", ComponentContent, "ComponentContent.update", "ComponentContent.dispose", wrapBuilder, WrappedBuilder, "promptAction.BaseDialogOptions", DismissDialogAction, DismissReason, DialogAlignment, TextTimer, TextTimerController, TextTimerConfiguration, "TextTimer.contentModifier", "TextTimer.onTimer", ContentModifier, "PromptAction.showToast", "AppStorage.setOrCreate", "AppStorage.get"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0111, HW-19-0112, HW-19-0113, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a registration or login screen needs a **lightweight
tap-in-order challenge** before it will send an SMS code, plus the resend
countdown that follows it.

Two reusable pieces come out of it, and they are worth more than the challenge
itself:

- a **dialog utility** that opens, updates and closes a `@Builder`-defined dialog
  through `ComponentContent` - so the dialog content can be re-rendered while it
  is open;
- a **custom `TextTimer` content area** through `contentModifier`, which is how
  you get a countdown that renders as your own components rather than the
  built-in time string.

**Read HW-19-0111 and HW-19-0113 first.** The shipped verification logic accepts
a wrong order under a common condition, and the document does not say that a
client-side verdict is not a security control.

## Feature checklist

The application must:

- Present four characters in scrambled positions with a prompt image showing the
  required order.
- Record the tap order and show the sequence number on each tapped character.
- Fail the challenge if **any** tap is out of order (HW-19-0111).
- Refresh the challenge and let the user retry on failure.
- On success, close the dialog, mark the state verified and enable the send-code
  button.
- Run a resend countdown with a custom-rendered `TextTimer`.
- Dispose the `ComponentContent` after the dialog closes.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes the `UIContext` into `AppStorage` under `uiContext` |
| `common/Constants.ets` | `COUNT = 60000`, `ELAPSED_TIME = 6000`, `DURATION_300 = 300`, `CODE_STATE = [0, 1, 2]`, and the layout numbers |
| `model/TextModel.ets` | one character: its glyph, its required `clickIndex`, whether it `isClicked`, and its shown `curClickIndex` |
| `components/CustomDialogBuilder.ets` | the dialog content as a global `@Builder`, plus `clickText` and `refreshText` |
| `components/CustomTextTimer.ets` | `MyTextTimerModifier` and the `customTextTimer` builder |
| `utils/DialogUtil.ets` | `DialogUtils` - open / update / close a `ComponentContent` dialog by type |
| `utils/Logger.ets` | `hilog` wrapper |
| `pages/Login.ets` | the login form, the send-code button and the `TextTimer` |

**The dialog wrapper is the reusable part.** `DialogUtils.showDialog` builds a
`ComponentContent` from a `WrappedBuilder` plus its argument object, opens it with
`promptAction.openCustomDialog`, and records it in a list keyed by
`DialogShowType`. That record is what makes the other two operations possible:

- `updateDialog` calls `info.com.update(args)` - re-rendering the open dialog with
  new data, which is how the tapped sequence numbers appear;
- `closeDialog` calls `prompt.closeCustomDialog(info.com)` and then
  `info.com.dispose()` in the resolution handler.

`onWillDismiss` is set so only a close-button dismissal is honoured:

```ts
onWillDismiss: (dismissDialogAction: DismissDialogAction) => {
  if (dismissDialogAction.reason === DismissReason.CLOSE_BUTTON) {
    dismissDialogAction.dismiss();
  }
}
```

**The custom countdown.** `TextTimer` normally renders a formatted time string;
`contentModifier` replaces its content area with a builder that receives a
`TextTimerConfiguration`. The remaining seconds are computed from two of its
fields:

```ts
Text(Math.max(config.count / 1000 - config.elapsedTime / 100, 0).toFixed(0))
```

`count` is documented as "Timer duration, in milliseconds", so `/1000` gives
seconds. `elapsedTime` is "Elapsed time of the timer, in the minimum unit of the
format", and with the default format (`HH:mm:ss.SS`) that minimum unit is
1/100 s - so `/100` also gives seconds. The two different divisors are correct
**because the two fields are in different units**; that is worth knowing before
anyone "fixes" it.

**Verification state lives in module-level variables** mirrored into
`AppStorage`: `curClickOrder`, `orderIsCorrected`, `isValidated`,
`isObtainedCode`. They are read once at module load and maintained in place.

## Implementation steps

1. **Define the dialog content as a global `@Builder`** taking the data it
   renders, so it can be wrapped with `wrapBuilder` and re-rendered by `update`.
2. **Write the dialog utility** around `ComponentContent`: keep the instance in a
   registry so `update` and `close` can find it, and `dispose()` it after
   `closeCustomDialog` resolves.
3. **Open with `openCustomDialog(componentContent, options)`**, setting
   `onWillDismiss` to control which dismissal reasons are honoured.
4. **On each tap**: mark the character clicked, increment the order counter,
   store the displayed sequence number, compare it with the required index, and
   **only ever clear** the correctness flag (HW-19-0111).
5. **Re-render** with `DialogUtils.upDateCustomDialog(textList)` so the sequence
   numbers appear.
6. **After the last tap**, decide, toast the result, refresh the challenge, and on
   success close the dialog and set the verified flag. Use `setTimeout` for the
   short delays and cancel them if the dialog goes away (HW-19-0112).
7. **Render the countdown** with `TextTimer({ isCountDown: true, count,
   controller })` plus a `contentModifier`, starting it in `onAppear` and
   resetting it in `onTimer` when `elapsedTime` reaches the threshold.
8. **Say what the check is worth.** Document that the verdict is client-side
   (HW-19-0113).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The dialog utility

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/ets/utils/DialogUtil.ets`

```ts
export enum DialogShowType {
  OPEN
}

interface DialogModel {
  com: ComponentContent<object>;
  popType: DialogShowType;
}

export class DialogUtils {
  private static popShare: DialogUtils;
  private infoList: DialogModel[] = [];

  static shareInstance(): DialogUtils {
    if (!DialogUtils.popShare) {
      DialogUtils.popShare = new DialogUtils();
    }
    return DialogUtils.popShare;
  }

  static showDialog<T extends object>(type: DialogShowType, contentView: WrappedBuilder<[T]>, args: T,
    options?: promptAction.BaseDialogOptions): void {
    let uiContext = AppStorage.get<UIContext>('uiContext');
    if (uiContext) {
      let prompt = uiContext.getPromptAction();
      let componentContent: ComponentContent<T> = new ComponentContent(uiContext, contentView, args);
      AppStorage.setOrCreate('componentContent', componentContent);
      let customOptions: promptAction.BaseDialogOptions = {
        alignment: options?.alignment || DialogAlignment.Center,
        onWillDismiss: (dismissDialogAction: DismissDialogAction) => {
          if (dismissDialogAction.reason === DismissReason.CLOSE_BUTTON) {
            dismissDialogAction.dismiss();
          }
        }
      };
      // Open pop-ups using openCustomDialog
      prompt.openCustomDialog(componentContent, customOptions,);
      let infoList = DialogUtils.shareInstance().infoList;
      infoList[0] = { com: componentContent, popType: type };
      DialogUtils.shareInstance().infoList = infoList;
    }
  }
}
```

### Update and close

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/ets/utils/DialogUtil.ets`

```ts
static updateDialog<T extends object>(type: DialogShowType, args: T, options?: promptAction.BaseDialogOptions): void {
  let uiContext = AppStorage.get<UIContext>('uiContext');
  if (uiContext) {
    let sameTypeList = DialogUtils.shareInstance().infoList.filter((model) => model.popType === type);
    if (!sameTypeList || sameTypeList.length < 1) {
      return;
    }
    let info = sameTypeList[sameTypeList.length - 1];
    if (info && info.com) {
      // Update pop-ups using update
      info.com.update(args);
    }
  }
}

static closeDialog(type: DialogShowType): void {
  // ... find info as above ...
  prompt.closeCustomDialog(info.com)
    .then(() => {
      LOGGER.info('customDialog closed.');
      if (info.com !== null) {
        info.com.dispose();
        AppStorage.setOrCreate('isDisposed', true);
      }
    }).catch((error: BusinessError) => {
      LOGGER.error(`closeCustomDialog args error code is ${error.code}, message is ${error.message}`);
    });
}
```

### The tap handler (as shipped - see HW-19-0111 and HW-19-0112)

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/ets/components/CustomDialogBuilder.ets`

```ts
function clickText(textList: TextModel[], textItem: TextModel) {
  if (!textItem.isClicked) {
    textItem.isClicked = true;

    if (curClickOrder !== undefined) {
      curClickOrder++;
      AppStorage.setOrCreate('curClickOrder', curClickOrder);
      textItem.curClickIndex = curClickOrder;
    }

    if (curClickOrder !== textItem.clickIndex) {
      if (orderIsCorrected) {
        orderIsCorrected = false;
      }
    } else {
      orderIsCorrected = true;        // FIX (HW-19-0111): never set back to true
    }

    DialogUtils.upDateCustomDialog(textList);

    // After 300 meters, make a judgment to ensure the full sequence number is displayed by the fourth click
    let intervalID1 = setInterval(() => {          // FIX (HW-19-0112): setTimeout
      if (curClickOrder === 4) {
        if (!orderIsCorrected) {
          AppStorage.get<UIContext>('uiContext')?.getPromptAction().showToast({
            message: $r('app.string.verify_failed'),
          });
          refreshText(textList);
          DialogUtils.upDateCustomDialog(textList);
        } else {
          AppStorage.get<UIContext>('uiContext')?.getPromptAction().showToast({
            message: $r('app.string.verify_succeeded'),
          });
          refreshText(textList);
          // Close the pop-up after 300m to fully display the serial number
          let intervalID2 = setInterval(() => {
            DialogUtils.closeCustomDialog();
            clearInterval(intervalID2);
          }, Constants.DURATION_300);

          isValidated = true;
          AppStorage.setOrCreate('isValidated', isValidated);
          isObtainedCode = Constants.CODE_STATE[1];
          AppStorage.setOrCreate<number>('isObtainedCode', isObtainedCode);
        }
      }
      clearInterval(intervalID1);
    }, Constants.DURATION_300);
  }
}
```

Corrected correctness flag:

```ts
// in refreshText, alongside curClickOrder = 0
orderIsCorrected = true;

// in clickText
if (curClickOrder !== textItem.clickIndex) {
  orderIsCorrected = false;   // sticky: never set back to true here
}
```

### The custom countdown

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/ets/components/CustomTextTimer.ets`

```ts
export class MyTextTimerModifier implements ContentModifier<TextTimerConfiguration> {
  constructor() {
  }

  applyContent(): WrappedBuilder<[TextTimerConfiguration]> {
    return wrapBuilder(customTextTimer);
  }
}

@Builder
function customTextTimer(config: TextTimerConfiguration) {
  Row() {
    // 剩余时间：用初始时间减去计时器经过的时间
    Text(Math.max(config.count / 1000 - config.elapsedTime / 100, 0).toFixed(0))
      .fontSize($r(`app.float.font_size_14`))
      .lineHeight($r(`app.float.line_height_19`))
      .fontWeight(FontWeight.Normal)
      .fontColor($r('app.color.font_color_black_ninety'));
    Text($r('app.string.unit_second'))
      .fontSize($r(`app.float.font_size_14`))
      .lineHeight($r(`app.float.line_height_19`))
      .fontWeight(FontWeight.Normal)
      .fontColor($r('app.color.font_color_black_ninety'));
  };
}
```

### Wiring the timer into the page

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/ets/pages/Login.ets`

```ts
@State myTimerModifier: MyTextTimerModifier = new MyTextTimerModifier();
textTimerController: TextTimerController = new TextTimerController();

if (this.isObtainedCode === Constants.CODE_STATE[1] && this.isValidated) {
  Row() {
    TextTimer({ isCountDown: true, count: Constants.COUNT, controller: this.textTimerController })
      .contentModifier(this.myTimerModifier)
      .onTimer((utc: number, elapsedTime: number) => {
        LOGGER.info('textTimer notCountDown utc is：' + utc + ', elapsedTime: ' + elapsedTime);
        if (elapsedTime >= Constants.ELAPSED_TIME) {
          this.isObtainedCode = Constants.CODE_STATE[2];
          this.textTimerController.reset();
          this.countDownFished = true;
        }
      })
      .onAppear(() => {
        this.textTimerController.start();
        this.countDownFished = false;
      });
  }
}
```

`Constants.COUNT = 60000` ms and `Constants.ELAPSED_TIME = 6000` - 6000 units of
1/100 s, that is the same 60 seconds expressed in `elapsedTime`'s unit.

## Permissions & config

**No permissions are required** and none are declared - the whole feature is UI,
with no network call anywhere in the project (which is also the substance of
HW-19-0113).

`TextOrderVerification.zip#TextOrderVerification/entry/src/main/module.json5`
declares the usual `EntryAbility` with the home skill and an
`EntryBackupAbility`. The ability publishes the `UIContext` into `AppStorage`
under `uiContext`, which is what lets `DialogUtils` reach a `PromptAction` from
outside a component.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `openCustomDialog` /
  `closeCustomDialog` on `PromptAction` and `ComponentContent.update` are API 12
  interfaces.
- **`ComponentContent` must be disposed.** `closeCustomDialog` closes it;
  `dispose()` releases it, and the sample does that in the resolution handler.
- **`update()` re-renders the open dialog with a new argument object** - it is
  what makes an interactive dialog possible without reopening it.
- **`onWillDismiss` decides which dismissals are honoured.** With only
  `DismissReason.CLOSE_BUTTON` handled, a back press or an outside tap does not
  close this dialog.
- **`TextTimerConfiguration.count` is in milliseconds** with a documented maximum
  of 86400000 (24 hours) and a default of 60000.
- **`TextTimerConfiguration.elapsedTime` is "in the minimum unit of the
  format"** - it changes meaning if `format` is set. `Login.ets` declares
  `@State format: string = 'ss'` but never applies it, so the default format is in
  force and `elapsedTime` counts hundredths of a second. Applying `.format('ss')`
  would silently break both the `/100` divisor and the `ELAPSED_TIME` threshold.
- **`onTimer` does not fire when the screen is locked or the app is in the
  background**, so the countdown cannot be relied on as a timer of record.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`orderIsCorrected` is set back to `true` on every matching tap, which is
  incorrect.** Only the last comparison survives, so tapping 2, 1, 3, 4 passes.
  Make the flag sticky - clear it on a mismatch and never set it in `clickText`.
  (HW-19-0111)
- **Both delays use `setInterval` with a self-clear, which is incorrect.**
  `setTimeout` expresses them, and neither handle is reachable from a teardown,
  so a dialog dismissed within 300 ms still gets a `closeCustomDialog` call and a
  verdict computed against refreshed state. (HW-19-0112)
- **The document never says the verdict is client-side only.** Challenge, answer
  and result all live on the device, and the flag it sets unlocks the SMS-code
  request issued by that same device. (HW-19-0113)
- **`@State format: string = 'ss'` in `Login.ets` is dead.** It is never passed to
  `.format()`; if it were, the countdown arithmetic would break silently - see the
  `elapsedTime` note above.
- **The verification state is module-level `let`s seeded from `AppStorage` at
  import time.** `orderIsCorrected` is never written back, so `AppStorage` and the
  module disagree about it - only the module copy is ever read.
- **`infoList[0] = info` overwrites slot 0** rather than pushing, so the registry
  holds at most one dialog even though `closeDialog` and `updateDialog` are
  written to search a list by type.
- **`curClickOrder === 4` hardcodes the challenge length.** The character count is
  fixed in `TextModel` data; a five-character challenge would never complete.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` -
  `openCustomDialog`, `closeCustomDialog`, `BaseDialogOptions`, `onWillDismiss`,
  `DismissDialogAction` and `DismissReason`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` -
  `ComponentContent`, `update` and `dispose`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` -
  `TextTimer`, `count` ("Timer duration, in milliseconds"), `isCountDown`,
  `format` and its effect on the update unit, `elapsedTime` ("in the minimum unit
  of the format"), `onTimer` and its background/lock-screen caveat,
  `contentModifier` and `TextTimerConfiguration`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-content-modifier.md` -
  the `ContentModifier<T>` interface.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-content-modifier
- https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-dialog-encapsulation -
  the dialog-encapsulation practice this sample's `DialogUtils` follows.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_order_verification-0000002347027492
