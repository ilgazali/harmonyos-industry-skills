---
id: FIN-09
title: Randomised-order custom keyboard for password entry
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/09_out_of_order_custom_keyboard.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/out_of_order_custom_keyboard-0000002425425873
sample: huawei_industry_tree/07_finance_insurance/downloads/OutOrderKeyboard.zip
kits: ["@kit.ArkUI", "@kit.CryptoArchitectureKit", "@kit.BasicServicesKit", "@kit.AbilityKit"]
apis: [TextInput.customKeyboard, TextInputController, caretPosition, Grid, GridLayoutOptions, emitter.on, emitter.off, cryptoFramework.createRandom, generateRandomSync, window.setWindowPrivacyMode]
permissions: [ohos.permission.PRIVACY_WINDOW]
min_api: 20
modules: [entry]
findings: [HW-07-0034, HW-07-0035, HW-07-0043, HW-07-0044, HW-07-0045, HW-07-0046, HW-07-0047, HW-07-0048, HW-07-0053, HW-07-0054]
status: verified-with-fixes
---

## When to use

Load this card when key positions must be **unpredictable between sessions** so
that touch positions do not leak the secret - PINs, payment passwords, app-lock
codes. A shoulder surfer, a smudge pattern on the glass, or a recording of the
screen bezel all reduce to nothing if the layout moves every time.

`FIN-01` is the fixed-layout secure keyboard and `FIN-05` the stock-code
keyboard; this one is the same controller with a shuffle on top. **The shuffle
as shipped is not uniform and drops three letters** - see the pitfalls. Take the
structure from here and the shuffle from the corrected snippet below.

## Feature checklist

- Tapping the password field raises a custom keypad, never the system IME.
- Letters, digits and symbols, switchable from keys on the panel.
- The letter and digit layouts are reshuffled each time they are shown.
- The window enters privacy mode while the keypad is up.
- Randomness comes from the crypto framework, not `Math.random`.

## Architecture

One module. The keyboard is a folder, not a HAR - contrast `FIN-05`, which
packages the same idea as a reusable library.

```
entry/src/main/ets/
├── outorderkeyboard/
│   ├── KeyboardConstants.ets       sizes, indices, KEYBOARD_TYPE_EVENT_ID = 20250807
│   ├── KeyboardController.ets      text, caret, maxLength, createSecurityRandom
│   ├── KeyboardData.ets            KeyboardType enum, key definitions
│   ├── OutOrderKeyboard.ets        the panel: three layouts + privacy mode
│   ├── ABCKeyboardComponent.ets    letters, 3 rows
│   ├── NumKeyboardComponent.ets    digits, a Grid
│   └── SymbolKeyboadComponent.ets  symbols  (sic - the filename is misspelled)
├── pages/LoginPage.ets             @Provide keyboardController, the password field
└── utils/DisplayUtil.ets
```

**Two channels connect the parts.** `@Provide` / `@Consume` carries the
controller and the input text down to every layout; an `emitter` event carries
the "keyboard type changed" signal *sideways*, from the controller to whichever
layout is becoming visible, so it can reshuffle itself.

```typescript
// KeyboardController - broadcast
changeKeyboardType(value: KeyboardType) {
  emitter.emit({ eventId: KeyboardConstant.KEYBOARD_TYPE_EVENT_ID }, { data: { value: value } });
}

// each layout - listen for its own type
emitter.on(event, (eventData: emitter.EventData) => {
  if (eventData.data?.value === KeyboardType.NUM_KEYBOARD) {
    this.resortNumKeyboardData();
  }
});
```

The three layouts are all mounted and switched with `.visibility()`, not `if`,
so none of them is destroyed while the keyboard is up.

## Implementation steps

1. **Provide one `KeyboardController`** from the page hosting the field.
2. **Bind it with `customKeyboard`** so the system IME never appears.
3. **Shuffle on `aboutToAppear` and on the type-change event.**
4. **Use a Fisher-Yates shuffle**, not `Array.sort` with a random comparator
   (`HW-07-0034`).
5. **Draw the random bytes once from one reused generator** (`HW-07-0035`).
6. **Slice the shuffled alphabet across the rows without dropping any**
   (`HW-07-0043`).
7. **Turn on privacy mode in the panel's `onAppear`** and off in `onDisAppear`.

## Verified snippets

From `OutOrderKeyboard.zip`. Corrected forms are marked.

**Privacy mode tied to the panel's lifetime — `outorderkeyboard/OutOrderKeyboard.ets`** (as shipped)

```typescript
Flex({ direction: FlexDirection.Column }) {
  ABCKeyboardComponent()
    .visibility(this.keyboardController.keyboardType === KeyboardType.ABC_KEYBOARD ?
      Visibility.Visible : Visibility.None);
  NumKeyboardComponent()
    .visibility(this.keyboardController.keyboardType === KeyboardType.NUM_KEYBOARD ?
      Visibility.Visible : Visibility.None);
  SymbolKeyboardComponent()
    .visibility(this.keyboardController.keyboardType === KeyboardType.SYMBOL_KEYBOARD ?
      Visibility.Visible : Visibility.None);
}
.onAppear(() => {
  WindowUtils.setWindowPrivacyModeInPage(this.getUIContext().getHostContext() as common.UIAbilityContext, true);
})
.onDisAppear(() => {
  WindowUtils.setWindowPrivacyModeInPage(this.getUIContext().getHostContext() as common.UIAbilityContext, false);
});
```

**Scoping privacy mode to the panel is the right instinct** and worth copying:
protection begins and ends with the thing that needs protecting, with no flag to
forget. Compare `FIN-08`'s `HW-07-0031`, where the same call is made by hand at
the wrong moment. This sample's own header comment says as much - *若窗口是否为隐私
模式不希望通过键盘拉起状态控制，请将此处删除*.

**The shuffle** (corrected, see `HW-07-0034`, `HW-07-0035`, `HW-07-0046`)

```typescript
// FIX: shipped code is
//   this.abcMenuData.sort(() => this.keyboardController.createSecurityRandom() - 0.5);
// A random comparator is not a shuffle. Measured over 200,000 runs on ten
// elements: digit 0 lands first 19.4% of the time instead of 10%, and the
// largest deviation from uniform across the position matrix is 93.7%. A
// cryptographic random source does not fix it - the bias is in the sorting.

// KeyboardController - one generator, one batched draw
private random = cryptoFramework.createRandom();

randomBytes(n: number): Uint8Array {
  return this.random.generateRandomSync(n).data;   // FIX: shipped code constructs a new
                                                   // generator per comparison, ~100 per shuffle
}

shuffle<T>(arr: T[]): void {
  const bytes = this.randomBytes(arr.length);
  for (let i = arr.length - 1; i > 0; i--) {
    // FIX: shipped createSecurityRandom divides the byte by 255, so it can return
    // exactly 1.0 and Math.floor(r * n) can yield n. Divide by 256.
    const j = Math.floor((bytes[i] / 256) * (i + 1));
    const tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
  }
}
```

Fisher-Yates needs `n - 1` draws and is uniform by construction. For the
26-letter keyboard that is 25 draws in one call instead of roughly a hundred
generator constructions and a hundred blocking single-byte reads, all on the UI
thread at the moment the user is waiting for the keypad.

**Slicing the shuffled alphabet across rows** (corrected, see `HW-07-0043`)

```typescript
abcMenuData: string[] =
  ['q','w','e','r','t','y','u','i','o','p','a','s','d','f','g','h','j','k','l','z','x','c','v','b','n','m'];  // 26

sliceAbcKeyboardData() {
  // FIX: shipped bounds are slice(0, 9), slice(10, 18), slice(19, 25) - 9 + 8 + 6 = 23 keys.
  // slice excludes its end index, so indices 9, 18 and 25 are dropped on every shuffle and
  // three randomly chosen letters cannot be typed at all.
  this.abcKeyboardData0 = this.abcMenuData.slice(0, 10);   // 10
  this.abcKeyboardData1 = this.abcMenuData.slice(10, 19);  //  9
  this.abcKeyboardData2 = this.abcMenuData.slice(19, 26);  //  7
}
```

**The one-line check that catches this class of bug:** the rendered key count
must equal `abcMenuData.length`. The shipped code passes no such check and the
loss is invisible in review because the shuffled rows look plausible either way.

**Insertion with a length cap — `outorderkeyboard/KeyboardController.ets`** (corrected, see `HW-07-0044`, `HW-07-0045`)

```typescript
maxLength: number = Infinity;   // LoginPage sets it to 16

// FIX: shipped onInput opens with
//   if (!this.maxLength || typeof this.maxLength === 'undefined' || this.maxLength === undefined)
//     this.maxLength = Infinity;
// - one condition written three ways, which also rewrites a cap of 0 into no cap.

private insert(value: string): string {
  // FIX: shipped code computes room as maxLength - this.text.length, ignoring the
  // selection the insertion replaces. At the cap with everything selected, room is 0,
  // the character is truncated away and the field is left empty.
  const kept = this.text.length - (this.rightCaretPos - this.leftCaretPos);
  const room = this.maxLength - kept;
  const accepted = value.length > room ? value.substring(0, Math.max(0, room)) : value;

  this.text = this.text.substring(0, this.leftCaretPos) + accepted + this.text.substring(this.rightCaretPos);
  this.targetCaretPos = this.leftCaretPos + accepted.length;
  return this.text;
}

onPaste(value: string) {
  this.insert(value);      // FIX: shipped code duplicates the truncation logic here
}
```

Identical to `FIN-05`'s `HW-07-0019` / `HW-07-0020` - the same controller ships
in both samples with the same two defects.

**Event cleanup** (corrected, see `HW-07-0047`)

```typescript
private onTypeChange = (eventData: emitter.EventData) => {
  if (eventData.data?.value === KeyboardType.ABC_KEYBOARD) {
    this.resortABCKeyboardData();
  }
};

aboutToAppear(): void {
  this.resortABCKeyboardData();
  emitter.on({ eventId: KeyboardConstant.KEYBOARD_TYPE_EVENT_ID }, this.onTypeChange);
}

aboutToDisappear(): void {
  // FIX: shipped code calls emitter.off(KEYBOARD_TYPE_EVENT_ID), which the reference
  // defines as "Unsubscribes from all events with the specified event ID" - so one
  // component's teardown cancels its siblings' subscriptions too.
  emitter.off(KeyboardConstant.KEYBOARD_TYPE_EVENT_ID, this.onTypeChange);
}
```

## Permissions & config

`entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.PRIVACY_WINDOW" }
]
```

Normal-level, granted at install. The document lists it under 权限说明.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `customKeyboard` replaces the IME for that field entirely - no prediction, no
  voice input, no user layout, and no `type(InputType.Password)` dot rendering
  from the IME.
- The cap is set once on the controller (`MAX_LENGTH_16` in `LoginPage`), so one
  controller means one cap for every field it drives.
- `generateRandomSync` blocks; keep the draw count proportional to `n`, not to
  the comparison count.
- `KEYBOARD_TYPE_EVENT_ID` is a bare number in app scope - collisions with other
  emitter users are the caller's problem.

## Pitfalls

- **`HW-07-0043` — three letters vanish from the keyboard on every shuffle**
  (blocker). `slice(0, 9) / slice(10, 18) / slice(19, 25)` renders 23 of 26 keys;
  which three are missing changes with each shuffle, so a password containing one
  of them cannot be typed.
- **`HW-07-0034` — `Array.sort` with a random comparator is not a shuffle.**
  Digit 0 lands first 19.4% of the time rather than 10%. For a keypad whose only
  purpose is unpredictability, the bias is the whole feature failing quietly.
- **`HW-07-0035` — a new crypto generator per comparison,** ~100 constructions
  and 100 blocking draws per alphabetic shuffle, on the UI thread.
- **`HW-07-0044` — the length cap ignores the replaced selection,** so typing
  over a full selection at 16 characters empties the field.
- **`HW-07-0045` — a three-clause guard testing one condition,** which turns a
  cap of `0` into `Infinity`.
- **`HW-07-0046` — `createSecurityRandom` divides by 255,** so it can return
  exactly `1.0` and any `Math.floor(r * n)` index is one past the end once every
  256 draws.
- **`HW-07-0047` — `emitter.off(eventId)` removes every listener on the id,** not
  the caller's own.
- **`HW-07-0048` — a key colour chosen by `&&`-ing a resource with a boolean,**
  which works only because a `Resource` is truthy.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `customKeyboard`, `TextInputController`, `caretPosition`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `Grid`, `GridLayoutOptions`, irregular items
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `on`, `off(eventId)` vs `off(eventId, callback)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createRandom`, `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `setWindowPrivacyMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `FIN-01` - the fixed-layout secure keyboard
- `FIN-05` - the stock-code keyboard, which ships the same controller as a HAR
- `FIN-08` - the PIN dialog this keypad is meant to sit under
