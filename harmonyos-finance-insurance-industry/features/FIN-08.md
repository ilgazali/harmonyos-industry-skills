---
id: FIN-08
title: Bank card number - reveal behind a PIN, copy to the clipboard
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/08_bank_card_number_copy.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/bank_card_number_copy-0000002369744416
sample: huawei_industry_tree/07_finance_insurance/downloads/CardNumberCopy.zip
kits: ["@kit.BasicServicesKit", "@kit.ArkUI", "@kit.IMEKit", "@kit.AbilityKit"]
apis: [pasteboard.createData, pasteboard.getSystemPasteboard, setData, window.getLastWindow, setWindowPrivacyMode, inputMethod.getController, attach, on insertText, on deleteLeft, detach, PromptAction.openCustomDialog]
permissions: [ohos.permission.PRIVACY_WINDOW]
min_api: 20
modules: [entry]
findings: [HW-07-0030, HW-07-0031, HW-07-0032, HW-07-0033, HW-07-0049, HW-07-0053, HW-07-0054]
status: verified-with-fixes
---

## When to use

Load this card when a screen must **hold a sensitive string masked, reveal it
after a check, and let the user copy it** - card numbers, account numbers,
recovery codes, API keys. It combines three pieces that are usually written
separately: a PIN entry driven straight from the input method, an anti-screenshot
window mode, and a clipboard write.

Read the pitfalls before adopting: as shipped this sample verifies nothing and
protects the wrong thing. The mechanics are worth copying; the control flow
around them is not.

## Feature checklist

- The card number renders masked until the user passes a check.
- Tapping 查看卡号 opens a six-digit PIN dialog.
- While the dialog is up, screenshots and screen recording are blocked.
- After the check the label becomes 复制卡号 and a tap copies the number.
- A toast confirms the copy.

## Architecture

One page, one static helper, no state container.

```
entry/src/main/ets/
├── common/Constants.ets        VERIFY_CODE_LENGTH = 6, PASSWORD_ANONYMOUS_SYMBOL = '*'
├── model/TransactionItem.ets   mock transaction rows
├── pages/Home.ets              card list
└── pages/CardDetail.ets        @Entry - masking, PIN dialog, privacy mode, copy
```

`CardDetail` holds everything: `account` (the displayed string, masked or not),
`isValidatePass`, `codeText` (the digits typed so far), and the
`inputMethod.InputMethodController` the dialog listens through.

**The dialog has no TextInput.** The six boxes are `Text` components, and the
digits arrive from the input method controller directly - this is the piece
worth extracting.

## Implementation steps

1. **Declare `ohos.permission.PRIVACY_WINDOW`** in `module.json5`.
2. **Enable privacy mode before opening the dialog**, and keep it on for as long
   as the unmasked value is on screen (`HW-07-0031`).
3. **Attach the input method controller** when the dialog opens, and subscribe
   to `insertText` and `deleteLeft`.
4. **Accumulate digits into `codeText`**, filtering non-numerics.
5. **Verify the entered code against something** before revealing (`HW-07-0030`).
6. **Detach the controller** and unsubscribe both events when done.
7. **Set a `shareOption` on the paste data** and clear the clipboard afterwards
   (`HW-07-0032`).

## Verified snippets

From `CardNumberCopy.zip`. Corrected forms are marked.

**Privacy-mode helper — `pages/CardDetail.ets`** (as shipped)

```typescript
class WindowUtils {
  static setWindowPrivacyModeInPage(context: common.UIAbilityContext, isFlag: boolean) {
    window.getLastWindow(context).then((lastWindow) => {
      lastWindow.setWindowPrivacyMode(isFlag, (err: BusinessError) => {
        const errCode: number = err.code;
        if (errCode) {
          return;                    // note: silent - see FIN-01's HW-07-0006
        }
      });
    });
  }
}
```

**Driving a PIN box from the input method — same file** (as shipped, and the
reason to read this sample)

```typescript
private inputController: inputMethod.InputMethodController = inputMethod.getController();
private textConfig: inputMethod.TextConfig = {
  inputAttribute: {
    textInputType: inputMethod.TextInputType.NUMBER,
    enterKeyType: inputMethod.EnterKeyType.GO
  }
};

async attach() {
  await this.inputController.attach(true, this.textConfig);   // true = show the keyboard now
  this.listen();
}

listen() {
  this.inputController.on('insertText', (text: string) => {
    if (this.codeText.length >= Constants.VERIFY_CODE_LENGTH || isNaN(Number(text)) || text === ' ') {
      return;
    }
    this.codeText += text;
    // ... completion handling
  });
  this.inputController.on('deleteLeft', () => {
    this.codeText = this.codeText.substring(0, this.codeText.length - 1);
  });
}

detach() {
  this.inputController.off('insertText');
  this.inputController.off('deleteLeft');
  this.inputController.detach(() => { });
}
```

`attach` binds the app to the IME without any editable component existing, so
the digits can be captured into plain state and rendered as dots. That is the
whole trick: **no `TextInput` means no text buffer for anything else to read,
and full control over what each box displays.** `off` here takes the event name,
so unsubscribing is precise - unlike `emitter.off`, which is why `FIN-09` has
`HW-07-0047`.

**Completion handling** (corrected, see `HW-07-0030`, `HW-07-0031`)

```typescript
if (this.codeText.length === Constants.VERIFY_CODE_LENGTH) {
  // FIX: shipped code never compares codeText with anything - it only checks the
  // length, so any six digits pass. Verify before revealing.
  const ok = await this.verifyPin(this.codeText);
  this.codeText = '';
  if (!ok) {
    this.showPinError();
    return;
  }
  this.promptAction.closeCustomDialog(this.customDialogComponentId);
  this.detach();
  this.isValidatePass = true;
  this.account = await this.fetchFullAccount();
  // FIX: shipped code calls setWindowPrivacyModeInPage(..., false) right here,
  // four lines before the real number is assigned - protecting the six digits the
  // user typed and not the sixteen they unlock. Keep privacy mode ON while the
  // number is visible and release it in aboutToDisappear.
}
```

**The masked dots** (corrected, see `HW-07-0049`)

```typescript
ForEach(this.codeIndexArray, (item: number) => {
  ListItem() {
    // FIX: shipped false branch is `this.codeText[item]`, which is undefined -
    // the expression it just tested for falsiness.
    Text(this.codeText[item] ? Constants.PASSWORD_ANONYMOUS_SYMBOL : '')
      .border({ width: 0.2, style: { bottom: BorderStyle.Solid }, color: '#0A59F7' })
      .width('16.5%').height(40).textAlign(TextAlign.Center);
  };
}, (item: number) => item.toString());
```

**The clipboard write** (corrected, see `HW-07-0032`)

```typescript
let pasteData = pasteboard.createData(pasteboard.MIMETYPE_TEXT_PLAIN, this.account);
// FIX: shipped code writes it straight out with no shareOption and never clears it.
pasteData.setProperty({ ...pasteData.getProperty(), shareOption: pasteboard.ShareOption.InApp });
let systemPasteboard = pasteboard.getSystemPasteboard();
await systemPasteboard.setData(pasteData);
this.promptAction.showToast({ message: 'success', duration: 2000, bottom: 400 });
// and clear it once it is no longer needed
```

`shareOption` is the property that decides how far a clipboard item may travel -
in-app, on-device, or across devices. For a card number it is the difference
between a copy and a leak.

## Permissions & config

`entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.PRIVACY_WINDOW" }
]
```

`PRIVACY_WINDOW` is a normal-level permission granted at install time - no
runtime request, no user prompt. Which is also why a failure to apply it is
invisible unless you check the callback.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `setWindowPrivacyMode` applies to the **window**, not a component - everything
  on screen is covered or nothing is.
- `inputMethod.getController()` returns a process-wide singleton; two screens
  attaching at once will fight over it.
- `attach` must be awaited before subscribing, and `detach` must run on every
  exit path including dismissal, or the app stays bound to the IME.
- The system clipboard is shared with every app on the device and, if the user
  has enabled it, with their other devices.

## Pitfalls

- **`HW-07-0030` — the dialog verifies nothing.** It tests only that six digits
  were typed; `codeText` is cleared without being compared. Any six digits
  reveal the card number.
- **`HW-07-0031` — privacy mode is switched off four lines before the number is
  revealed,** so the PIN is protected from screenshots and the card number is not.
- **`HW-07-0032` — no `shareOption`, and the clipboard is never cleared,** so the
  number stays readable by any app until something overwrites it.
- **`HW-07-0033` — the mask is 17 characters for a 16-digit number,** so the field
  reflows on reveal and misstates the length.
- **`HW-07-0049` — the dot ternary returns `undefined` for empty slots.**

## References

- `documentation/harmonyos-guides/04_system/use-pasteboard-to-copy-and-paste.md` - `PasteData`, `shareOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/use-pasteboard-to-copy-and-paste
- `documentation/harmonyos-references/03_system/js-apis-pasteboard.md` - `createData`, `setData`, `ShareOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pasteboard
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `setWindowPrivacyMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod.md` - `getController`, `attach`, `on('insertText')`, `detach`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod
- `FIN-06` - the app lock, for how the credential this dialog should check ought to be stored
- `FIN-01` - the secure keyboard, the other approach to PIN entry in this industry
- `FIN-09` - the randomised keypad, if the PIN positions must also be unpredictable
