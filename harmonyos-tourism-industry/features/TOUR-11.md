---
id: TOUR-11
title: Traveller details form - phone and Chinese ID card validation with inline error styling
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/11_check_information.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
sample: huawei_industry_tree/09_tourism/downloads/CheckInformation.zip
kits: ["@kit.ArkUI"]
apis: [TextInput, InputType, maxLength, inputFilter, onFocus, Checkbox, Divider, Button, "UIContext.getPromptAction", showToast, "@State", "@StorageProp", "$$", RegExp, Date]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0066, HW-09-0067, HW-09-0068, HW-09-0069, HW-09-0070, HW-09-0071, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for a **traveller or identity form that must be validated
before it is submitted** - the passenger details step of a booking, a
real-name registration, a check-in form. Two things are worth taking from it:

- **The Chinese resident ID card check.** Length, province prefix, birth date
  and the GB 11643 checksum, in twenty lines. This is the piece nobody should
  write from memory - the weight sequence and the check-digit mapping are
  fiddly and this implementation gets both right.
- **Inline error styling driven by one boolean per field**: the input's
  `fontColor` and its `Divider`'s `color` both flip, and `onFocus` clears the
  error so the field goes black again the moment the user returns to it.

The form-level lesson is the one the sample gets wrong: **one error flag per
field, not per message** (`HW-09-0066`).

## Feature checklist

- A traveller form: name, phone number, document type (fixed to ID card),
  document number.
- A warning banner explaining the details must match the booking.
- An agreement checkbox gating the confirm button, which is greyed out until
  it is ticked.
- Confirm validates the ID card, then the phone, and toasts the first failure.
- The offending field and its divider turn red; focusing the field clears the
  error styling.
- A success toast when everything passes.

## Architecture

One `entry` module, one page, one utility, one constants file.

```
entry/src/main/ets
├── constants/StyleConstants.ets   every literal dimension, weight and colour name
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage as px
├── entrybackupability/
├── pages/CheckInfoMainPage.ets    @Entry - the whole form, 229 lines
└── utils/CheckMethods.ets         checkPhoneNo, isValidDate, checkIDCardNo
```

The document's tree calls the utility `CheckInformation.ets`; the file is
`CheckMethods.ets` (`HW-09-0070`).

**The validation layer is pure and testable.** `CheckMethods.ets` imports
nothing, holds no state, and exports three functions over strings. That is
what lets the checksum be unit-tested without a UI, and it is the right shape
for any format validator - keep it out of the component.

**The page holds one boolean per error**, not per field, which is the defect:
`isNameCorrect` and `isPhoneNoCorrect` for three inputs, so the ID card
borrows the name's flag.

## Implementation steps

1. **Write the validators as free functions** over plain strings, with no
   component or context dependency.
2. **Anchor the phone regex at both ends** (`HW-09-0068`).
3. **Validate the ID card in four stages** - length, province prefix, birth
   date, checksum - cheapest first, so a malformed string never reaches the
   arithmetic.
4. **Parse the birth date in local time**, not through `new Date(string)`
   (`HW-09-0069`).
5. **Give every input its own error flag**, and bind both the input's
   `fontColor` and its `Divider`'s `color` to it (`HW-09-0066`).
6. **Clear the flag in `onFocus`**, so returning to a field resets it without
   a second validation pass.
7. **Constrain the input at entry** with `type` and `maxLength`, and leave the
   validators to catch what only they can (`HW-09-0071`).
8. **Let one component own the agreement flag** - either the checkbox or the
   row, never both (`HW-09-0067`).

## Verified snippets

All snippets are from `CheckInformation.zip`. Corrected forms are marked.

**The ID card checksum — `entry/src/main/ets/utils/CheckMethods.ets`** (as shipped)

```typescript
// leading two digits: the province codes issued on the mainland
const PROVINCE_ID = ['11', '12', '13', '14', '15', '21', '22', '23', '31', '32', '33', '34',
  '35', '36', '37', '41', '42', '43', '44', '45', '46', '50', '51', '52', '53', '54',
  '61', '62', '63', '64', '65'];

export function checkIDCardNo(id: string): boolean {
  // stage 1-3: length, province, birth date - all cheap, all before the arithmetic
  if (id.length !== 18 || !PROVINCE_ID.includes(id.slice(0, 2)) ||
    !isValidDate(`${id.slice(6, 10)}-${id.slice(10, 12)}-${id.slice(12, 14)}`)) {
    return false;
  }

  // stage 4: the GB 11643 weighted checksum over the first 17 digits
  let ans: number = 0;
  for (let i = 0; i < 17; i++) {
    if (id[i].charCodeAt(0) === 32) {
      return false;                       // a space would make Number() return 0, not NaN
    }
    ans += (2 ** (17 - i) % 11) * Number(id[i]);
  }
  const res = (12 - ans % 11) % 11;
  if (res === 10) {
    return id[17] === 'X' || id[17] === 'x';
  }
  return id[17] === res.toString();
}
```

**Two details make this correct.** The weight for position `i` is
`2^(17-i) mod 11`, which generates the standard sequence
7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2 without a lookup table.
The check digit is `(12 - sum mod 11) mod 11`, which maps a remainder of 2 to
the value 10 - the only case rendered as `X`.

The explicit space test is not decoration: `Number(' ')` is `0`, not `NaN`, so
a space inside the number would silently contribute zero and could produce a
valid checksum. Every other non-digit yields `NaN`, which poisons the sum and
fails the comparison.

The card layout is `AABBCCYYYYMMDDXXXC`: two province digits, four more
region digits, eight birth-date digits, three sequence digits, one check
character.

**Validating the birth date — same file** (corrected, see `HW-09-0069`)

```typescript
export function isValidDate(dateStr: string) {
  const parts = dateStr.split('-');
  const y = Number(parts[0]);
  const m = Number(parts[1]);
  const d = Number(parts[2]);
  const date = new Date(y, m - 1, d);          // FIX: sample uses new Date(dateStr) - UTC
  if (date.getTime() >= new Date().getTime()) {
    return false;                              // a birth date in the future is not one
  }
  // the round trip rejects overflow dates: 1990-02-31 comes back as 1990-03-03
  return date.getFullYear() === y && date.getMonth() + 1 === m && date.getDate() === d;
}
```

**The round-trip comparison is the technique worth keeping.** `Date` silently
normalises out-of-range components, so constructing the date and checking that
its parts came back unchanged is a complete calendar validation - leap years
included - in one line and with no month-length table.

What the sample gets wrong is the construction: `new Date('1990-01-01')`
parses as UTC midnight while `getFullYear()` reads local time, so the round
trip fails everywhere west of Greenwich.

**The phone regex — same file** (as shipped)

```typescript
export function checkPhoneNo(num: string) {
  const PHONE_REGEXP = new RegExp('^(?:(?:\\+|00)86)?1[3-9]\\d{9}$');
  return PHONE_REGEXP.test(num);
}
```

Optional `+86` or `0086` country code, then a mainland mobile number: leading
`1`, second digit 3 to 9, nine more digits. **Both anchors matter** - the
document's copy of this line omits the `$` (`HW-09-0068`), which turns it into
"contains a phone number at the start" and accepts trailing junk.

**One flag per field — `entry/src/main/ets/pages/CheckInfoMainPage.ets`** (corrected, see `HW-09-0066`)

```typescript
@State isNameCorrect: boolean = true;
@State isPhoneNoCorrect: boolean = true;
@State isIdCardCorrect: boolean = true;          // FIX: absent in the sample

// the ID card row
TextInput({ placeholder: $r('app.string.input_card_no'), text: $$this.idCardNo })
  .type(InputType.Normal)
  .maxLength(18)                                 // FIX: sample sets no length limit
  .fontColor(this.isIdCardCorrect ? $r('app.color.black_font_color')
                                  : $r('app.color.red_font_line_color'))
  .onFocus(() => {
    this.isIdCardCorrect = true;                 // returning to the field clears the error
  })

Divider()
  .strokeWidth(StyleConstants.ZERO_POINT_FIVE)
  .color(this.isIdCardCorrect ? $r('sys.color.mask_fourth') : $r('app.color.red_font_line_color'))
```

**Colouring the divider as well as the text** is what makes the error read as
"this row", not "this word" - the underline runs the full width of the field
and is visible even when the input is empty. `onFocus` as the reset is the
other half: the user does not have to resubmit to clear the red.

**Validating on submit — same file** (corrected, see `HW-09-0066`)

```typescript
Button($r('app.string.confirm'))
  .fontColor(this.isAgree ? Color.White : $r('app.color.button_disabled_font_color'))
  .backgroundColor(this.isAgree ? $r('app.color.button_normal_background_color')
                                : $r('app.color.button_disabled_background_color'))
  .onClick(() => {
    if (!this.isAgree) {
      this.promptAction.showToast({ message: $r('app.string.check_real_name_notice') });
      return;
    }
    if (!checkIDCardNo(this.idCardNo.trim())) {
      this.isIdCardCorrect = false;              // FIX: sample sets isNameCorrect = false
                                                 // FIX: sample also forces isPhoneNoCorrect = true
      this.promptAction.showToast({ message: $r('app.string.input_id_card_no_error_error') });
      return;
    }
    if (!checkPhoneNo(this.phoneNo.trim())) {
      this.isPhoneNoCorrect = false;
      this.promptAction.showToast({ message: $r('app.string.input_phone_no_error') });
      return;
    }
    this.promptAction.showToast({ message: $r('app.string.success') });
  })
```

`.trim()` before validating is right and easy to forget - a trailing space
pasted from a contact card would otherwise fail the anchored regex.

The button is styled as disabled rather than actually disabled, so tapping it
un-agreed produces an explanatory toast instead of silence. That is the better
choice for a gate the user can act on.

**The agreement row — same file** (corrected, see `HW-09-0067`)

```typescript
Row() {
  Checkbox({ name: 'checkbox', group: 'checkboxGroup' })
    .select(this.isAgree)                       // FIX: sample uses $$this.isAgree
    .hitTestBehavior(HitTestMode.None)          // FIX: let the tap reach the Row
  Text($r('app.string.i_agree'))
}
.onClick(() => {
  this.isAgree = !this.isAgree;
})
```

In the shipped form the `$$` two-way binding and the row's `onClick` both flip
`isAgree` on a single tap of the box, so it toggles twice and appears dead.
One owner per piece of state.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

No routing configuration - a single `@Entry` page.

Resource directories: `base`, `dark`, `en_US`, `zh_CN`. All user-visible text
goes through `$r`, including the error messages - this is the best-behaved
sample in the industry on that point.

`StyleConstants.ets` holds every numeric literal under a self-describing name
(`WIDTH_40`, `MARGIN_TOP_280`, `ZERO_POINT_FIVE`). It keeps the page free of
magic numbers, at the cost of constants whose names are their values.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **Mainland China only.** The phone regex accepts `+86` numbers and the ID
  card check accepts 18-character second-generation resident ID cards. It
  rejects passports, Hong Kong / Macau / Taiwan permits, and the older
  15-digit cards, and the province list stops at 65 - no 71, 81 or 82. The
  document type field is a fixed label, not a picker, so there is nowhere to
  select another document.
- The name field is **never validated** - it is collected and ignored.
- Nothing is submitted anywhere; a successful check is a toast.
- The confirm button's top margin is a fixed 280 vp
  (`StyleConstants.MARGIN_TOP_280`), so the layout assumes a phone-height
  screen and does not adapt.

## Pitfalls

- **`HW-09-0066` — an invalid ID card turns the *name* field red.** There is
  no `isIdCardCorrect`; the ID card input reuses `isNameCorrect`, and the same
  branch also forces `isPhoneNoCorrect` back to `true`, clearing a phone error
  the user has not fixed.
- **`HW-09-0067` — the agreement checkbox toggles twice per tap,** because
  `$$this.isAgree` and the parent `Row`'s `onClick` both flip it. Tapping the
  box does nothing, and the confirm button stays gated.
- **`HW-09-0068` — the document's phone regex is missing the `$`** that the
  shipped code has, so the documented version accepts trailing text.
- **`HW-09-0069` — `isValidDate` parses `YYYY-MM-DD` as UTC and reads local
  components,** so in any timezone behind UTC every genuine ID card is
  rejected before the checksum runs.
- **`HW-09-0070` — the documented tree names `utils/CheckInformation.ets`;**
  the file is `utils/CheckMethods.ets`.
- **`HW-09-0071` — no `type` and no `maxLength` on any input,** so a form
  built around format validation gives the user an alphabetic keyboard for a
  phone number and no length limit on either field.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInput`, `InputType`, `maxLength`, `inputFilter`, `onFocus`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-checkbox.md` - `Checkbox` and `select`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-checkbox
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-click.md` - click dispatch and bubbling
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-click
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-hit-test-behavior.md` - `hitTestBehavior` and `HitTestMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-hit-test-behavior
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - the `$$` two-way binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `showToast`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `TOUR-04` - the date range step before this one; `TOUR-10` - the order this form produces
