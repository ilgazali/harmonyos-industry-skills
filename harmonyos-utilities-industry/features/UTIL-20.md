---
id: UTIL-20
title: Calculator - a data-driven keypad over a shunting-yard evaluator with decimal-safe arithmetic
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/20_calculator.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculator-0000002298744774
sample: huawei_industry_tree/15_utilities/downloads/Calculator.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [common, hilog, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0050, HW-15-0051, HW-15-0052, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you need **a keypad-driven expression editor with a live
result** - a calculator, but equally a unit converter, a tip splitter, a
quantity field in an order form, any input where the user builds a value from
buttons rather than a keyboard and expects to see the answer update as they
type.

Three separable pieces make it up, and each is reusable on its own. The
**keypad** is a `ForEach` over a view-model array, so key layout, glyphs and
sizes are data rather than markup. The **token list** is an
`Array<string>` where each element is either a whole number or a single
operator, which makes backspace, operator replacement and "0 can't be
followed by a digit" trivial rules over the last one or two elements. The
**evaluator** is a small shunting-yard: infix tokens to a postfix queue, then
a stack pass over the queue.

The piece most worth stealing is the arithmetic. JavaScript's `0.1 + 0.2` is
not `0.3`, and a calculator that shows that is broken. `CalculateUtil.add`
and `mulOrDiv` scale both operands to integers by the larger decimal length,
compute, then scale back and `toFixed` - a compact alternative to pulling in
a decimal library.

## Feature checklist

- A 4-column keypad: digits, `.`, `%`, `C`, backspace, `+`, `-`, `×`, `÷`,
  and a double-height `=`.
- The expression appears in a read-only `TextArea` at 48 vp; the running
  result sits below it at 26 vp in grey.
- Every digit press re-evaluates and updates the result line immediately; the
  decimal point alone does not.
- Pressing an operator after an operator replaces it, except that `-` after
  `+` or `×` is kept as a sign.
- Backspace removes one character, then one token, and clears both lines when
  the expression empties.
- `=` promotes the result to the expression line and seeds the next
  calculation with it.
- Division by zero shows the localized error string.
- Results longer than 16 digits switch to scientific notation.
- Numbers are meant to be shown with thousands separators (dead as shipped -
  `HW-15-0051`).

## Architecture

One `entry` module. A clean three-layer split: view model for the keys, util
for the maths, page for the interaction rules.

```
entry/src/main/ets
├── common/constants/CommonConstants.ets  operator glyphs, Symbol/Priority/SymbolicEnumeration enums,
│                                         NUM_MAX_LEN = 16, sizes and weights
├── common/util/CalculateUtil.ets         isSymbol, getPriority, parseExpression, dealQueue,
│                                         calResult, add, mulOrDiv, numberToScientificNotation
├── common/util/CheckEmptyUtil.ets        the null/empty guard every util method opens with
├── common/util/Logger.ets
├── entryability/EntryAbility.ets         full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/HomePage.ets                    @Entry: keypad, input rules, formatting
├── viewmodel/PressKeyItem.ets            flag, width, height, value, source?
└── viewmodel/PresskeysViewModel.ets      getPressKeys(): Array<Array<PressKeyItem>>
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that the operator glyphs are `×`
and `÷` (U+00D7, U+00F7) everywhere - in the enum, in the token list, in the
display - and never `*` or `/`. There is no translation layer between what
the user sees and what the evaluator switches on, so a whole class of
"display string vs internal string" bugs cannot occur. `CommonConstants.OPERATORS`
is the four-character string `'+-×÷'`, and `isSymbol` is one `indexOf` against
it.

The second is that `HomePage` never mutates `this.expressions` from inside the
evaluator: `getResult()` calls `CalculateUtil.parseExpression(this.deepCopy())`.
That matters because `parseExpression` **consumes** its argument - it
`shift()`s the array empty - so passing the live token list would erase the
user's expression as a side effect of showing them a result.

## Implementation steps

1. **Describe the keypad as data.** `getPressKeys()` returns four columns of
   `PressKeyItem`; `flag` selects the renderer (`0` image, `3` white-on-blue
   equals, `2` blue operator, else black digit) and the last item of the last
   column gets the double height, the 40 vp radius and the accent background.
2. **Suppress the system keyboard on the display field** with a zero-sized
   `@Builder` passed to `customKeyboard`, plus `selectionMenuHidden(true)`.
3. **Keep the expression as a token array,** one element per number or
   operator, and route every key through `inputNumber` or `inputSymbol`.
4. **Validate before appending.** `validateEnter` rejects a leading `%`, a
   second `.` in one number, a digit after a bare `0`, and a `%` after a `%`.
   Extend it to reject `%` after every operator, not just `-` (`HW-15-0052`).
5. **Replace, don't stack, operators**: `inputOperators` pops the previous
   operator before pushing the new one, keeping `-` when it follows `+` or a
   high-precedence operator so it reads as a sign.
6. **Evaluate with shunting-yard** over a copy: expand `%` into `n ÷ 100`
   first, drop a trailing operator, then convert to postfix using
   `comparePriority` and reduce the queue with a stack.
7. **Scale decimals to integers before arithmetic** rather than trusting
   binary floats, and re-scale with `toFixed(maxLen)`.
8. **Guard `NaN` alongside `±Infinity`** when converting the numeric result to
   its display string (`HW-15-0050`).
9. **Format thousands with a regex literal, at render time only** - not by
   reassigning the `@Watch`ed variable inside its own watcher
   (`HW-15-0051`).

## Verified snippets

All snippets are from `Calculator.zip`. Corrected forms are marked.

**The data-driven keypad - `entry/src/main/ets/pages/HomePage.ets`** (as shipped)

```typescript
ForEach(keysModel.getPressKeys(), (columnItem: Array<PressKeyItem>, columnItemIndex?: number) => {
  Column() {
    ForEach(columnItem, (keyItem: PressKeyItem, keyItemIndex?: number) => {
      Column() {
        if (keyItem.flag === 0) {
          Image(keyItem.source !== undefined ? keyItem.source : '')
            .width(keyItem.width)
            .height(keyItem.height);
        } else if (keyItem.flag === 3) {
          Text(keyItem.value)
            .fontSize(36)
            .fontColor(Color.White)
            .width(keyItem.width)
            .height(keyItem.height);
        } else {
          Text(keyItem.value)
            .fontColor(keyItem.flag === 2 ? 'rgba(10, 89, 247, 1)' : Color.Black)
            .fontSize(keyItem.width)
            .width('100%')
            .height(64);
        }
      }
      .width(64)
      .height(((columnItemIndex === (keysModel.getPressKeys().length - 1)) &&
        (keyItemIndex === (columnItem.length - 1))) ? 148 : 64)
      .onClick(() => {
        // 根据flag判断是数字还是符号
        if (keyItem.flag === 0 || keyItem.flag === 2 || keyItem.flag === 3) {
          this.inputSymbol(keyItem.value);
        } else {
          if (this.lastBtnIsEqu) {
            this.expressions = [];
          }
          this.lastBtnIsEqu = false;
          this.inputNumber(keyItem.value);
        }
      });
    }, (keyItem: PressKeyItem) => JSON.stringify(keyItem));
  }
}, (item: Array<Array<PressKeyItem>>) => JSON.stringify(item));
```

**`flag` does two jobs and that is the point.** It selects the renderer at
build time and the handler at click time, so adding a key means adding one
`PressKeyItem` to the view model - no new markup, no new branch. The keypad
is laid out as four `Column`s inside a `Row` with
`justifyContent(FlexAlign.SpaceBetween)`, which is why `=` can be 148 vp tall
and still line up: it simply occupies the space two ordinary keys would.

`lastBtnIsEqu` is the one piece of interaction state that is not in the token
list. After `=`, the result is pushed back as the sole token; pressing a
*digit* next should start a fresh calculation (so the array is cleared), while
pressing an *operator* should continue from the result (so it is not). That
asymmetry is the whole reason the flag exists.

Note the key generators: `JSON.stringify(keyItem)` is stable here only
because no two keys are identical. Prefer the index or the `value` for a
list whose items could repeat.

**The display field - same file** (as shipped)

```typescript
// 自定义键盘（宽高设为0）
@Builder
CustomKeyboardBuilder() {
  Column() {
    Grid() {
      // 键盘内容（实际不可见）
    }
    .height(0)
    .width(0);
  };
}

TextArea({ text: this.resultFormat(this.inputValue) })
  .selectionMenuHidden(true)
  .customKeyboard(this.CustomKeyboardBuilder())
  .fontSize(48)
  .maxLines(6)
  .textAlign(TextAlign.End)
  .align(Alignment.End)
```

**A zero-sized `customKeyboard` is the idiom for "focusable but no soft
keyboard".** The field must stay focusable so the caret and text selection
behave, but the app supplies its own keys, and any non-empty custom keyboard
would occupy the bottom of the screen. Setting both dimensions to `0`
satisfies `customKeyboard`'s contract while rendering nothing.
`selectionMenuHidden(true)` removes the cut/paste popup, which would
otherwise let the user paste arbitrary text into an expression the evaluator
assumes is well-formed.

**Decimal-safe arithmetic and the NaN guard - `entry/src/main/ets/common/util/CalculateUtil.ets`** (corrected, see `HW-15-0050`)

```typescript
add(arg1: string, arg2: string, symbol: string): number {
  let addFlag = (symbol === CommonConstants.ADD);
  if (this.containScientificNotation(arg1) || this.containScientificNotation(arg2)) {
    return addFlag ? Number(arg1) + Number(arg2) : Number(arg1) - Number(arg2);
  }
  let leftArr = arg1.split(CommonConstants.DOTS);
  let rightArr = arg2.split(CommonConstants.DOTS);
  let leftLen = leftArr.length > 1 ? leftArr[1] : '';
  let rightLen = rightArr.length > 1 ? rightArr[1] : '';
  let maxLen = Math.max(leftLen.length, rightLen.length);
  let multiples = Math.pow(CommonConstants.TEN, maxLen);
  if (addFlag) {
    return Number(((Number(arg1) * multiples + Number(arg2) * multiples) / multiples).toFixed(maxLen));
  }
  return Number(((Number(arg1) * multiples - Number(arg2) * multiples) / multiples).toFixed(maxLen));
}

numberToScientificNotation(result: number) {
  if (Number.isNaN(result) ||                       // FIX: only ±Infinity was guarded
    result === Number.NEGATIVE_INFINITY || result === Number.POSITIVE_INFINITY) {
    return 'NaN';
  }
  let resultStr = JSON.stringify(result);
  if (this.containScientificNotation(resultStr)) {
    return resultStr;
  }
  let prefixNumber = (resultStr.indexOf(CommonConstants.MIN) === -1) ? 1 : -1;
  result *= prefixNumber;
  if (resultStr.replace(CommonConstants.DOTS, '').replace(CommonConstants.MIN, '').length <
    CommonConstants.NUM_MAX_LEN) {
    return resultStr;
  }
  let suffix = (Math.floor(Math.log(result) / Math.LN10));
  let prefix = (result * Math.pow(CommonConstants.TEN, -suffix) * prefixNumber);
  return (prefix + CommonConstants.E + suffix);
}
```

**`'NaN'` is a sentinel string, not a number,** and that is what makes the
missing guard fatal. `getResult()` compares `calResult === 'NaN'` and shows
the localized error text; everything else is displayed verbatim. The
conversion from number to string is `JSON.stringify(result)` - and
`JSON.stringify(NaN)` is the string `'null'`, because JSON has no NaN. So
`0 ÷ 0` produces `NaN`, survives the `±Infinity` check, becomes `'null'`,
fails the `containScientificNotation` test, is shorter than 16 characters,
and is returned as the answer. The user sees the word `null`. `1 ÷ 0` is
handled correctly, which is exactly why the hole is easy to miss.

The scaling in `add` is the technique to carry away: take the longer of the
two fractional parts, multiply both operands by `10^maxLen` so they are
integers, do the arithmetic, divide back, and `toFixed(maxLen)` to erase the
residue the round trip leaves. `mulOrDiv` does the same with
`leftLen + rightLen` for products and `rightLen - leftLen` for quotients. The
guard clause above it matters too: once a value has reached scientific
notation the string tricks stop working, so those operands fall through to
plain float arithmetic.

**Thousands separators - `entry/src/main/ets/pages/HomePage.ets`** (corrected, see `HW-15-0051`)

```typescript
inputValueChange() {
  // FIX: shipped guard is `!== null || !== undefined` (always true) and it
  // reassigns the very variable it watches. Format at render, not in the watcher.
  Logger.info('[HomePage] inputValue: ' + this.inputValue);
}

resultFormat(value: string) {
  // FIX: shipped code is new RegExp('/(\d)(?=(\d{3})+\.)/g') - the pattern text
  // includes the '/' delimiters and '\d' collapses to 'd', so nothing ever matches.
  let reg = (value.indexOf('.') > -1) ? /(\d)(?=(\d{3})+\.)/g : /(\d)(?=(?:\d{3})+$)/g;
  return value.replace(reg, '$1,');
}
```

**`new RegExp` takes a pattern, not a literal.** Wrapping the source in
`/.../g` and passing it as a string produces a regex that matches a literal
forward slash followed by `(d)(?=...)`; worse, `'\d'` inside a normal string
literal is just `d`, because `\d` is not a recognised string escape. Both
errors point the same way: use a regex literal, or if the pattern must be
built dynamically, double every backslash and pass the flags as the second
argument. The result is that no number anywhere in this calculator has ever
been rendered as `1,234,567`.

The watcher is the reason fixing the regex alone is not enough.
`inputValueChange` is `@Watch`ed on `inputValue` and assigns to
`this.inputValue`; today that is harmless because `resultFormat` is a no-op,
but a working formatter would make the watcher fire on its own write. The
formatted string is also what would be re-read later, so commas would leak
into values the evaluator parses. Formatting belongs where the sample already
does it - `TextArea({ text: this.resultFormat(this.inputValue) })` and
`Text(this.resultFormat(this.calValue))` - and the token list must stay
comma-free.

**The `%` guard - same file** (corrected, see `HW-15-0052`)

```typescript
validateEnter(last: string, value: string) {
  if (!last && value === CommonConstants.PERCENT_SIGN) {
    return false;
  }
  // FIX: shipped code checks only `last === CommonConstants.MIN`
  if (CalculateUtil.isSymbol(last) && (value === CommonConstants.PERCENT_SIGN)) {
    return false;
  }
  if (last.endsWith(CommonConstants.PERCENT_SIGN)) {
    return false;
  }
  if ((last.indexOf(CommonConstants.DOTS) !== -1) && (value === CommonConstants.DOTS)) {
    return false;
  }
  if ((last === '0') && (value !== CommonConstants.DOTS) &&
    (value !== CommonConstants.PERCENT_SIGN)) {
    return false;
  }
  return true;
}
```

`%` is not an operator in this design - it is a suffix on a number, expanded
by `parseExpression` into a division by 100:

```typescript
if (item.indexOf(CommonConstants.PERCENT_SIGN) !== -1) {
  expressions[index] = (this.mulOrDiv(item.slice(0, item.length - 1),
    CommonConstants.ONE_HUNDRED, CommonConstants.DIV)).toString();
}
```

So a `%` with no number in front of it becomes `item.slice(0, 0)` - an empty
string - and `Number('') / 100` is `0`. The shipped guard blocks that after
`-` and at the start of input but not after `+`, `×` or `÷`, so `5×%`
evaluates silently to `0` instead of rejecting the keystroke. Replacing the
single-operator test with `CalculateUtil.isSymbol(last)` makes the rule
uniform, which is what the other four checks in this method already are.

## Permissions & config

**None.** The sample declares no `requestPermissions` and needs none - it
touches no files, no network and no sensors.

Window configuration is the usual boilerplate, in `EntryAbility`:
`setWindowLayoutFullScreen(true)`, then `getWindowAvoidArea` for
`TYPE_SYSTEM` and `TYPE_NAVIGATION_INDICATOR` into `AppStorage`, then an
`avoidAreaChange` subscription to keep them current. `HomePage` reads them
with `@StorageLink` and converts with `px2vp` at the point of use. Two notes:
`@StorageProp` would be the more accurate decorator since the page only reads
these values, and the `avoidAreaChange` subscription is never released in
`onWindowStageDestroy` - the same omission that recurs across this industry's
samples.

`onCreate` pins the app to
`ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT`, which is how the
hardcoded `Color.Black` / `rgb(255, 255, 255)` key colours stay legible.
Remove that line and the keypad is unreadable in dark mode.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- No parentheses, no memory keys, no history: the expression is a flat token
  list, so precedence is the only structure the evaluator understands.
- `parseExpression` consumes the array it is given. Always pass a copy.
- `NUM_MAX_LEN` is 16 - past that, results are shown in the sample's own
  `<mantissa>e<exponent>` form rather than JavaScript's `toExponential`.
- `numberToScientificNotation` uses `Math.log(result)` after forcing `result`
  positive, so a very small magnitude yields a negative exponent by design;
  `0` never reaches that branch because its string is one character.
- The layout is fixed-size: the display column is `width(360) height(224)`
  and each key is 64 vp, so the keypad does not adapt to tablet or 2in1
  widths.
- The font family is `'鸿蒙黑体'` (HarmonyOS Sans), assumed present.

## Pitfalls

- **`HW-15-0050`** (B/medium, confirmed): `0 ÷ 0` displays the literal string
  `null`. `numberToScientificNotation` guards only `±Infinity`; `NaN` passes
  through `JSON.stringify(NaN) === 'null'`, clears every later check and is
  shown as the answer, while the `'NaN'` sentinel that would have triggered
  the error message is never produced. Fix: add `Number.isNaN(result)` to the
  same guard.
- **`HW-15-0051`** (B/medium, confirmed): the thousands separator is a dead
  feature. `resultFormat` builds its `RegExp` from a string that still
  contains the `/` delimiters and the `g` flag, and `'\d'` in a plain string
  literal collapses to `d`, so `replace` never matches and no number is ever
  rendered as `1,234,567`. Its `@Watch` handler compounds this: the guard
  `this.inputValue !== null || this.inputValue !== undefined` is always true,
  and the handler reassigns the variable it watches - safe today only because
  the formatter is a no-op, so fixing the regex alone would loop the watcher
  and push commas into parsed values. Fix: use a regex literal, format only at
  render, and keep the token list comma-free.
- **`HW-15-0052`** (B/low, confirmed): `%` is accepted directly after `+`, `×`
  and `÷`. `validateEnter` blocks it only at the start of input and after `-`,
  and `parseExpression` turns the dangling `%` into `Number('') / 100 = 0`, so
  `5×%` silently evaluates to `0` instead of rejecting the keystroke. Fix:
  test `CalculateUtil.isSymbol(last)` instead of the single `-` case.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` -
  `@Watch` semantics and why writing the watched variable from its handler is
  hazardous
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `documentation/harmonyos-references/02_application-framework/js-apis-arkts-decimal.md` -
  `Decimal` and `toFixed`, the library alternative to the manual scaling used
  here
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkts-decimal
- `documentation/harmonyos-references/02_application-framework/arkts-apis-arkts-collections-arraybuffer.md` -
  `slice`, cited by the document for the backspace implementation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-arkts-collections-arraybuffer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textarea.md`
  and `.../ts-basic-components-textinput.md` - `customKeyboard` and
  `selectionMenuHidden`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textarea
- `huawei_industry_tree/15_utilities/docs/20_calculator.md` - the source
  architecture page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculator-0000002298744774
