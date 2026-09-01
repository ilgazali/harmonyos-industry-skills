---
id: UTIL-15
title: Auto-advancing form - chain TextInput fields with key(), enterKeyType and focusControl
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/15_jump_to_next_input_text.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/jump_to_next_input_text-0000002326705889
sample: huawei_industry_tree/15_utilities/downloads/JumpToNextInPutText.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [TextInput, "TextInput.key", "TextInput.enterKeyType", EnterKeyType, InputType, onSubmit, onChange, "focusControl.requestFocus", "@Link", "@StorageProp", hilog, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0037, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card for a **long data-entry form** - registration, KYC, an
insurance application, an address book entry - where the user should be able to
type a field, press the keyboard's action key, and land on the next field
without reaching for the screen. It is also the pattern behind segmented
verification-code inputs, where each box hands off to its neighbour.

The mechanism is three attributes working together on one `TextInput`:
`.key(id)` names the field, `.enterKeyType(EnterKeyType.Next)` changes the
keyboard's bottom-right button to "next", and `.onSubmit()` calls
`focusControl.requestFocus(nextId)`. There is no form controller, no field
registry and no index arithmetic - each field only needs to know the id of the
one after it, so the chain is expressed entirely in the call site that builds
the row.

That linked-list shape is what makes it generalise. Reordering the form means
reordering the constructor calls; inserting a field means changing one `next`
value. It also degrades honestly: `requestFocus` returns a boolean and does not
throw, so an id that no longer exists costs a lost hop, not a crash. **But the
sample gets the two ends of the chain wrong** - the final field and the two
password fields (`HW-15-0037`) - and those are exactly the parts a real form
cannot ship without.

## Feature checklist

- A single-screen 个人信息 (personal information) form with ten labelled rows,
  visually grouped: name/gender/birthday in one white card, then each remaining
  field as its own rounded row.
- Every row is a label on the left and a text field on the right, filling the
  remaining width.
- The keyboard's action key reads "next" and moves focus to the following
  field when pressed.
- Each field writes back to its own page-level state as the user types.
- The phone field raises a numeric phone keypad rather than the general one.
- The 职业 (occupation) row shows a chevron on the right.
- A submit button at the bottom logs the collected values.
- The page is inset by the status bar and navigation indicator heights.

## Architecture

One `entry` module, one page file holding two structs, and a constants class.
No model layer - the form's data is ten `@State` strings on the page.

```
entry/src/main/ets
├── constants/StyleConstants.ets   the field labels and the key ids, as string constants
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
└── pages/JumpNextInput.ets        struct InputStyle (the reusable row, 22-62)
                                   struct JumpNextInput (@Entry, the form, 65-203)
```

The documented tree matches the zip.

**The design decision worth copying** is that the row component is a pure
function of four inputs and one link:

```typescript
@Component
struct InputStyle {
  text: string = '';            // the visible label
  placeholder: ResourceStr = '';
  index: string = '';           // this field's key
  next: string = '';            // the key to jump to on submit
  @Link value: ResourceStr;     // two-way binding into the page's state
}
```

`index` and `next` are what turn a list of independent rows into a chain. The
component never sees the form, never counts fields and never looks its
neighbour up; the parent expresses the order simply by what it passes. Adding a
field between two others is a two-line edit at the call site with no change to
`InputStyle` at all.

`@Link value` is the other half: each row is bound to a distinct `@State` on
the page (`this.name`, `this.gender`, …), so `onChange` writes straight through
and the submit handler reads ordinary component state. Ten separate `@State`
strings is verbose but explicit - the alternative, one object plus `@Observed`,
buys nothing at this size.

**Worth avoiding:** the ids are the Chinese label strings themselves
(`'姓名'`, `'性别'`), taken from the same constants class as the labels. That
couples the focus graph to display text; renaming a label silently breaks the
hop. Use opaque ids (`'field_name'`) and keep labels in resources.

## Implementation steps

1. **Build one row component** taking `text`, `placeholder`, `index`, `next`
   and an `@Link` for the value. Do not let it know the form.
2. **Give every `TextInput` a `.key(this.index)`.** `focusControl.requestFocus`
   resolves against `key(value)` or `id(value)`; without it the field cannot be
   a jump target.
3. **Set `.enterKeyType(EnterKeyType.Next)`** so the keyboard shows a "next"
   action instead of a return key - but only on fields that *have* a next
   (`HW-15-0037`).
4. **On the last field use `EnterKeyType.Done`** and submit the form from
   `onSubmit` instead of requesting focus on an empty key (`HW-15-0037`).
5. **Hop in `onSubmit`,** not in `onChange`: `onSubmit` fires on the action
   key press, which is the user saying "this field is finished".
6. **Give password fields `InputType.Password`** - the sample leaves 设置账户密码
   and 确认账户密码 as `InputType.Normal`, so both render in clear text
   (`HW-15-0037`).
7. **Choose `InputType` per field**, not just for the phone: `PhoneNumber` for
   the phone, `Number` for the income, `Password` for the two password fields,
   `Normal` for the rest.
8. **Wrap the form in a `Scroll`.** Ten 50 vp rows plus margins plus a raised
   keyboard do not fit a phone screen; the sample's bare `Column` cannot
   scroll, so the last fields are unreachable exactly when the keyboard that
   is supposed to walk you to them is open.
9. **Validate before hopping** if a field has a format (id number, phone). The
   hop is the natural moment to reject bad input, and `onSubmit` can simply not
   call `requestFocus`.

## Verified snippets

All snippets are from `JumpToNextInPutText.zip`,
`entry/src/main/ets/pages/JumpNextInput.ets`.

**The row component — as shipped**

```typescript
@Component
struct InputStyle {
  text: string = '';
  placeholder: ResourceStr = '';
  index: string = '';
  next: string = '';
  @Link value: ResourceStr;

  build() {
    Row() {
      Text(this.text)
        .fontSize(16)
        .fontWeight(500);
      TextInput({ placeholder: this.placeholder })
        .key(this.index)                      // 定义当前填写项的key值
        .fontSize(16)
        .width('60%')
        .layoutWeight(1)
        .type(this.text === StyleConstants.PHONE ? InputType.PhoneNumber : InputType.Normal)
        .placeholderColor('#66000000')
        .placeholderFont({ size: 16, weight: 400 })
        .backgroundColor(Color.White)
        .enterKeyType(EnterKeyType.Next)
        .onSubmit(() => {
          focusControl.requestFocus(this.next); // 填写跳转目标填写项的key值
        })
        .onChange((value: string) => {
          this.value = value;
        });

      Image($r('app.media.right'))
        .width(6)
        .visibility(this.text === StyleConstants.CAREER ? Visibility.Visible : Visibility.None);
    }
    .width('100%')
    .height(50)
    .backgroundColor(Color.White)
    .padding({ left: 15, right: 15 })
    .borderRadius(18);
  }
}
```

**Three attributes carry the design.** `.key(this.index)` publishes the field
under a name; `focusControl.requestFocus` is the *global* asynchronous focus
API and resolves its string argument against exactly this attribute (or
`id()`), transferring focus on the next frame. `.enterKeyType(EnterKeyType.Next)`
is purely the keyboard's action-key label - it does not itself move focus.
`.onSubmit` is the bridge: it fires when that action key is pressed, and that
is where the actual hop happens. Remove any one of the three and the chain
breaks silently.

`.width('60%')` followed by `.layoutWeight(1)` is redundant - the weight wins
and the field takes all the space the label leaves. The two conditionals both
compare against `this.text`, so the row's *label* decides its input type and
whether a chevron appears; that is the same display-text coupling as the ids,
and it is why the phone row must keep its exact label to keep its numeric
keypad.

**The row component — corrected, see `HW-15-0037`**

```typescript
@Component
struct InputStyle {
  text: string = '';
  placeholder: ResourceStr = '';
  index: string = '';
  next: string = '';                            // '' on the last field
  inputType: InputType = InputType.Normal;      // FIX: passed in, not inferred from the label
  submit: () => void = () => {};                // FIX: what the last field does instead of hopping
  @Link value: ResourceStr;

  build() {
    Row() {
      Text(this.text)
        .fontSize(16)
        .fontWeight(500);
      TextInput({ placeholder: this.placeholder })
        .key(this.index)
        .fontSize(16)
        .layoutWeight(1)
        .type(this.inputType)                   // FIX: InputType.Password for the two password rows
        .placeholderColor('#66000000')
        .placeholderFont({ size: 16, weight: 400 })
        .backgroundColor(Color.White)
        .enterKeyType(this.next ? EnterKeyType.Next : EnterKeyType.Done)   // FIX: Done at the end
        .onSubmit(() => {
          if (this.next) {
            focusControl.requestFocus(this.next);
          } else {
            this.submit();                      // FIX: last field submits, does not hop to ''
          }
        })
        .onChange((value: string) => {
          this.value = value;
        });
    }
    .width('100%')
    .height(50)
    .backgroundColor(Color.White)
    .padding({ left: 15, right: 15 })
    .borderRadius(18);
  }
}
```

**Both halves of `HW-15-0037` come from the same cause: the row cannot see
where it sits.** The shipped component hardcodes `EnterKeyType.Next` and
derives the input type from the label, so the last field advertises a "next"
action it cannot honour - `next` defaults to `''`, and
`focusControl.requestFocus('')` matches no component and returns `false`. The
user presses next on the final field of a ten-field form and nothing happens.
Making `enterKeyType` depend on `this.next` fixes the label and the behaviour
together, from one piece of information the parent already supplies.

Lifting `InputType` into a parameter fixes the second half. `InputType.Password`
is what masks characters and suppresses predictive text; with `Normal`, both
设置账户密码 (set account password) and 确认账户密码 (confirm account password)
render in the clear on screen, which is a shoulder-surfing exposure on a form
that also collects an id number and a phone number.

**The chain, expressed at the call sites — as shipped**

```typescript
InputStyle({
  text: StyleConstants.INCOME,
  placeholder: $r('app.string.enter_income'),
  index: StyleConstants.INCOME,
  next: StyleConstants.SET_PASSWORD,          // -> the next field's key
  value: this.income
})
  .margin({ bottom: 8 });
InputStyle({
  text: StyleConstants.SET_PASSWORD,
  placeholder: $r('app.string.enter_set_password'),
  index: StyleConstants.SET_PASSWORD,
  next: StyleConstants.CONFIRM_PASSWORD,
  value: this.password
})
  .margin({ bottom: 8 });
InputStyle({
  text: StyleConstants.CONFIRM_PASSWORD,
  placeholder: $r('app.string.enter_confirm_password'),
  index: StyleConstants.CONFIRM_PASSWORD,
  next: StyleConstants.EMERGENCY_CONTACT_NAME,
  value: this.confirmPassword
})
  .margin({ bottom: 8 });
InputStyle({
  text: StyleConstants.EMERGENCY_CONTACT_NAME,
  placeholder: $r('app.string.enter_emergency_contact_name'),
  index: StyleConstants.EMERGENCY_CONTACT_NAME,
  value: this.emergencyContactName               // no `next` — end of the chain
})
  .margin({ bottom: 90 });
```

**`index` on one row equals `next` on the row before it, and that is the whole
form graph.** There is no array of fields to keep in sync and no index maths;
the order on screen and the order of the focus chain are the same list of
constructor calls, so they cannot drift apart. The last call simply omits
`next`, which is why the corrected component can use its emptiness as the
end-of-chain signal.

The visual grouping is done purely with containers: the first three rows sit
inside a shared white `Column` with `borderRadius(18)`, and the remaining seven
are individually rounded rows separated by `margin({ bottom: 8 })`. The final
`margin({ bottom: 90 })` is the gap before the submit button.

**The constants that double as keys — `entry/src/main/ets/constants/StyleConstants.ets`** (as shipped)

```typescript
export class StyleConstants {
  static readonly TEXT_NAME: string = '姓名:';          // unused
  static readonly TEXT_SET_PASSWORD: string = '设置账户密码:';   // unused
  // ...
  static readonly NAME: string = '姓名';
  static readonly GENDER: string = '性别';
  static readonly PHONE: string = '联系电话';
  static readonly CAREER: string = '职业';
  static readonly INCOME: string = '年收入';
  static readonly SET_PASSWORD: string = '请设置密码';
  static readonly CONFIRM_PASSWORD: string = '确认账户密码';
  static readonly EMERGENCY_CONTACT_NAME: string = '紧急联系人姓名';
}
```

**The class holds two parallel sets of labels and ships only one.** The ten
`TEXT_*` constants carry the trailing colon that a form label wants
(`'姓名:'`) and are never imported; the page passes the bare forms
(`'姓名'`) as both the visible label *and* the key. So the rendered labels have
no colons, and the password row is labelled 请设置密码 ("please set a
password") rather than the 设置账户密码 ("set account password") its `TEXT_`
twin intends.

More importantly, these strings are hardcoded rather than resources, unlike
every placeholder on the page, which uses `$r('app.string.…')`. Translating the
form would change the labels - and, because the labels are the keys, would
break the focus chain and the phone keypad conditional at the same time. Split
the two roles: opaque ids in the constants class, labels in
`resources/base/element/string.json`.

## Permissions & config

**None.** The sample declares no `requestPermissions`, which is worth noting
given that it collects a name, gender, birth date, id number, phone number,
income, passwords and an emergency contact - none of that is read from the
device, all of it is typed by the user, and none of it is persisted or sent
anywhere. The submit handler logs it:

```typescript
hilog.info(0x0000, 'testTag', `提交信息${this.name} ${this.gender} ${this.birthday}`);
```

Logging personal data at `info` level with no `%{private}s` masking is
demo-only behaviour; do not carry it into an app.

`module.json5` lists `deviceTypes: ["phone", "tablet", "2in1"]` and a single
`EntryAbility` with no `EntryBackupAbility` and no `routerMap` - the form is
the launch page.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`focusControl.requestFocus` is asynchronous**: it requests the transfer for
  the *next* frame render. It returns `true` only if the target exists, is
  mounted and is focusable, and `false` otherwise - it never throws, so a
  broken chain is silent. For an immediate transfer the reference points to
  `FocusController.requestFocus` on the `UIContext`.
- Focus effects only render on a real device, not in the previewer.
- **The form cannot scroll.** The root is a `Column` with
  `justifyContent(FlexAlign.Start)`; ten 50 vp rows, the gaps, the 90 vp
  spacer and the button exceed a phone screen, and there is no `Scroll`
  ancestor, so the lower fields are cut off rather than reachable.
- The back arrow at the top of the page has no `onClick`.
- 职业 (occupation) is a plain `TextInput` that merely *shows* a chevron; it
  looks like a picker row and behaves like a text field.
- The `@Link value` is typed `ResourceStr` while every bound `@State` is a
  `string` and `onChange` assigns a `string`. It works, but the declared type
  is wider than anything the component can actually receive.

## Pitfalls

- **`HW-15-0037` — the last field's Next key requests focus on an empty key,
  and both password fields render unmasked.** (B/low, confirmed) The final
  `InputStyle` (`:171-177`) omits `next`, so `onSubmit` calls
  `focusControl.requestFocus('')`, which matches nothing and returns `false`;
  the field still advertises `EnterKeyType.Next` (`:44`). Separately, the only
  `InputType` special case is the phone (`:40`), so 请设置密码 and
  确认账户密码 (`:155-170`) use `InputType.Normal` and display in clear text.
  **Fix:** derive `enterKeyType` from whether `next` is set, submit instead of
  hopping on the last field, and pass `InputType.Password` for the password
  rows.
- **The keys are the display labels.** `.key(this.index)` receives the same
  Chinese string that `Text(this.text)` renders. Localising or editing a label
  breaks the focus chain, and the phone row loses its numeric keypad, with no
  compile error.
- **`InputType` and the chevron are inferred from the label text**
  (`this.text === StyleConstants.PHONE`, `=== StyleConstants.CAREER`) rather
  than passed in. Two string comparisons standing in for two parameters.
- **No validation anywhere.** The id number, phone and income accept any
  characters, and the two password fields are never compared, on a form whose
  point is to walk the user through to the end.
- **`hilog.info` prints the entered personal data unmasked** on submit.
- **`TEXT_NAME` … `TEXT_EMERGENCY_CONTACT_NAME` are dead constants** and the
  password label that ships (请设置密码) is not the one they intend
  (设置账户密码).

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInput`, `type`/`InputType`, `enterKeyType`/`EnterKeyType`, `onSubmit`, `onChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-focus.md` - `focusControl.requestFocus`, its boolean return, and which components accept focus
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-focus
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-component-id.md` - `key()` and `id()` as `requestFocus` targets
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-component-id
- `huawei_industry_tree/15_utilities/docs/15_jump_to_next_input_text.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jump_to_next_input_text-0000002326705889
- `UTIL-13` - the other text-entry card in this industry, on the keyboard's appearance rather than its action key
