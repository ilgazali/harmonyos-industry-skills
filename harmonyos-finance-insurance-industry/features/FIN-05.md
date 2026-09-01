---
id: FIN-05
title: Stock-code keyboard - custom numeric, letter and system keyboard switching
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/05_stock_keyboard.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_keyboard-0000002296858156
sample: huawei_industry_tree/07_finance_insurance/downloads/StockKeyboard.zip
kits: ["@kit.ArkUI"]
apis: [TextInput.customKeyboard, TextInputController, Grid, GridItem, ForEach, onChange, onPaste, caretPosition, Tabs]
permissions: []
min_api: 20
modules: [entry, keyboard, common]
findings: [HW-07-0019, HW-07-0020, HW-07-0021, HW-07-0022, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card when an input needs a **purpose-built keypad that the user can
still swap for the system keyboard** - stock codes, account numbers, licence
plates, product codes. The distinguishing feature over `FIN-01`'s secure
keyboard is the **tab bar**: numeric, letters, and a hand-back to the system IME,
switchable while the field stays focused.

## Feature checklist

- Tapping the search field raises a custom keyboard instead of the system one.
- A tab bar switches between numeric, English letters, and the system keyboard.
- The numeric pad carries stock-code shortcuts (600, 601, …) alongside digits.
- Input is capped at a configurable length.
- Caret position, selection replacement and paste all behave correctly.

## Architecture

Three modules - and packaging the keyboard as its own HAR is the point.

```
keyboard/                              reusable HAR
├── src/main/ets/common/KeyboardConstants.ets
├── src/main/ets/components/StockKeyboardComponent.ets       the panel
├── src/main/ets/components/StockKeyboardTabviewComponent.ets the tab bar
├── src/main/ets/components/StockNumKeyboardComponent.ets     numeric layout
├── src/main/ets/components/StockABCKeyboardComponent.ets     letter layout
├── src/main/ets/utils/StockKeyboardController.ets            text + caret state
└── src/main/ets/viewmodel/StockKeyboardData.ets              key definitions
entry/                                 the demo host
common/                                shared constants
```

`StockKeyboardController` is the piece that makes this work: it owns the text,
the caret bounds (`leftCaretPos`, `rightCaretPos`, `targetCaretPos`), the
keyboard type and the length cap. The layout components are stateless renderers
that call into it.

**Switching to the system keyboard** is a two-line move: set the controller's
type, and detach the custom keyboard.

```typescript
this.keyboardController.keyboardType = this.type;
this.isCustomKeyboardAttach = this.type !== KeyboardType.SYSTEM_KEYBOARD;
```

`customKeyboard` is only bound while `isCustomKeyboardAttach` is true, so
clearing it hands the field back to the IME without losing focus or text.

## Implementation steps

1. **Package the keyboard as a HAR** so it can be reused across apps.
2. **Put the text and caret state in a controller**, not in the layout
   components.
3. **Track `leftCaretPos` / `rightCaretPos`** so an insertion replaces a
   selection correctly.
4. **Compute remaining room from the text that survives the edit**, not from the
   current length (`HW-07-0019`).
5. **Restore the caret after every change** with
   `textInputController.caretPosition(targetCaretPos)` - but only when the
   custom keyboard is attached.
6. **Key each `ForEach` on the character it renders** (`HW-07-0021`).

## Verified snippets

From `StockKeyboard.zip`. Corrected forms are marked.

**Keyboard type switching — `keyboard/.../StockKeyboardTabviewComponent.ets`** (as shipped)

```typescript
TabButton()
  .onClick(() => {
    this.keyboardController.keyboardType = this.type;
    // detaching the custom keyboard hands the field back to the system IME
    this.isCustomKeyboardAttach = this.type !== KeyboardType.SYSTEM_KEYBOARD;
  });
```

**Insertion with a length cap — `keyboard/.../StockKeyboardController.ets`** (corrected, see `HW-07-0019`, `HW-07-0020`)

```typescript
maxLength: number = Infinity;

// FIX: shipped code opens with a three-clause guard that resets maxLength to
// Infinity whenever it is falsy - which silently turns a cap of 0 into no cap.
private insert(value: string): string {
  // FIX: shipped code computes room as maxLength - this.text.length, ignoring
  // the selected range that the insertion replaces. Typing over a full
  // selection at the cap then discards the character entirely.
  const kept = this.text.length - (this.rightCaretPos - this.leftCaretPos);
  const room = this.maxLength - kept;
  const accepted = value.length > room ? value.substring(0, Math.max(0, room)) : value;

  this.text = this.text.substring(0, this.leftCaretPos)
    + accepted
    + this.text.substring(this.rightCaretPos);
  this.targetCaretPos = this.leftCaretPos + accepted.length;
  return this.text;
}

onInput(value: string | Resource): string {
  if (typeof value === 'string') {
    return this.insert(value);
  }
  switch (value.id) {
    case $r('app.media.keyboard_del_icon').id:
      if (this.rightCaretPos === this.leftCaretPos && this.leftCaretPos > 0) {
        this.leftCaretPos--;                    // no selection: eat one char back
      }
      this.text = this.text.substring(0, this.leftCaretPos) + this.text.substring(this.rightCaretPos);
      this.targetCaretPos = this.leftCaretPos;
      break;
    // ... clear, case switch, etc.
  }
  return this.text;
}

onPaste(value: string) {
  this.insert(value);      // FIX: shipped code duplicates the truncation logic here
}
```

**The caret model is the reusable idea.** Holding `leftCaretPos` and
`rightCaretPos` rather than a single index means one code path covers "insert at
the caret" and "replace the selection": with no selection the two are equal and
the middle substring is empty. The backspace branch shows the same trick - it
only decrements `leftCaretPos` when there is no selection, because with one there
is already something to delete.

**Restoring the caret — same file** (as shipped)

```typescript
onChange(value: string) {
  this.text = value;
  if (this.keyboardType !== KeyboardType.SYSTEM_KEYBOARD) {
    this.textInputController?.caretPosition(this.targetCaretPos);
  }
}
```

The type check matters: when the system IME is driving, it owns the caret, and
forcing a position would fight it.

**The key grids** (corrected, see `HW-07-0021`)

```typescript
// numeric: a Grid, key heights carried in the data, widths shared evenly
Grid() {
  ForEach(numKeyboardData, (item: KeyboardMenu) => {
    GridItem() {
      Button(item.textOrImage, { type: ButtonType.Normal });
    };
  }, (item: KeyboardMenu) => item.textOrImage as string);   // FIX: shipped key is JSON.stringify(item)
};

// letters: Column of Rows, one Row per keyboard row
Column() {
  Row({ space: KeyboardConstants.KEYBOARD_MENU_SPACE }) {
    ForEach(abcKeyboardData[0], (item: KeyboardMenu, index: number) => {
      ABCButton({ item: item, rowIndex: 0, columnIndex: index });
    }, (item: KeyboardMenu) => item.textOrImage as string);
  };
  // ... three more rows
};
```

Two layouts for two shapes: a `Grid` where keys are uniform and positions come
from the data, `Column` + `Row` where each row has its own key count and the
wide keys need per-row handling.

## Permissions & config

**None.**

Resource directories: `entry` and `common` carry `base` and `dark`; the
`keyboard` module carries only `base` (`HW-07-0022`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The demo caps input at 8 characters, set from `TextInputComponent`.
- `customKeyboard` replaces the IME for that field entirely - anything the IME
  would have given you (prediction, voice, the user's own layout) is gone.
- The controller is per-field: two fields sharing one controller would share
  text and caret.

## Pitfalls

- **`HW-07-0019` — the length cap ignores the text being replaced.** Type eight
  digits with a cap of eight, select all, type a nine: the field is left empty
  and the keypress is lost. The same six lines are duplicated in `onPaste`, so
  pasting over a selection loses the paste too.
- **`HW-07-0020` — a three-clause guard testing one condition,** which also
  rewrites a cap of `0` to `Infinity` - the opposite of what a caller asking for
  zero-length input wants. The field is typed `number` with a default, so the
  `undefined` cases it checks for cannot occur.
- **`HW-07-0021` — five `ForEach` sites key on `JSON.stringify(item)`,** so a
  full alphabetic keyboard serialises about thirty objects on every render pass,
  on a component that re-renders per keystroke.
- **`HW-07-0022` — the keyboard HAR ships no dark resources** while the two
  modules around it do, so the panel does not follow the system theme.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `customKeyboard`, `TextInputController`, `caretPosition`, `onPaste`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `Grid` / `GridItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` keys
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `FIN-01` - the fixed-layout secure keyboard; compare the two before choosing
- `FIN-09` - the randomised-order keyboard, for PIN entry
