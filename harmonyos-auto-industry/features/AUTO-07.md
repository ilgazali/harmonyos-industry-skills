---
id: AUTO-07
title: Smart autofill of owner details, with a custom licence-plate keyboard
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/07_car_maintenance.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
sample: huawei_industry_tree/01_auto/downloads/CarMaintenance.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI"]
apis: [TextInput, contentType, ContentType.PERSON_FULL_NAME, ContentType.ID_CARD_NUMBER, ContentType.PHONE_NUMBER, autoFillManager.requestAutoSave, AutoSaveCallback, "@Observed", "@ObjectLink", "@Extend", LengthMetrics, PromptAction.showToast]
permissions: []
min_api: 24
modules: [entry, features/vehicleKeyboard]
findings: [HW-01-0043, HW-01-0044, HW-01-0045, HW-01-0046, HW-01-0047, HW-01-0048, HW-01-0050]
status: verified-with-fixes
---

## When to use

Load this card whenever the app asks the user to **type personal details they
have already given some other app** - name, ID number, phone, address - and you
want the system to offer them back instead of making the user retype. In this
industry it is the "bind your vehicle" form; the same mechanism applies to
delivery addresses, insurance applications and account registration.

It also carries the industry's **custom licence-plate keyboard**, which is a
separate reusable piece.

**This is the only practice in `01_auto` that requires API 24.** Every other
document in the industry targets API 20. Check `HW-01-0044` before mixing this
into a project built on the industry architecture.

## Feature checklist

- A form with name, ID card number and phone number.
- Each field declares what kind of data it holds, so the system offers
  previously saved values automatically - no code runs to fetch them.
- On submit, the entered values are offered back to the user's Huawei account so
  the next form is prefilled.
- A licence-plate field with a purpose-built keyboard (province characters, then
  alphanumerics) instead of the system keyboard.
- Empty-field validation before submitting.

## Architecture

Two modules: the `entry` HAP and a `vehicleKeyboard` feature HAR.

```
entry/src/main/ets
├── components/
│   ├── LicensePlateComponent.ets   plate field, hosts the custom keyboard
│   ├── OwnerInfo.ets               the three autofill-enabled fields
│   └── TitleBar.ets
├── constants/StyleConstants.ets
├── entryability/EntryAbility.ets   publishes topAvoid / bottomAvoid
├── model/OwnerInfoModel.ets        @Observed UserInfo
└── pages/MainPage.ets              owns UserInfo, submits the form

features/vehicleKeyboard/src/main/ets
├── components/Keyboard.ets         the keyboard UI
├── components/VehicleInput.ets     the input that raises it
└── constants/KeyboardConstant.ets
```

State: `MainPage` holds an `@Observed UserInfo` and passes it down as
`@ObjectLink`, so the child form mutates the same object and the parent sees the
values when the submit button is pressed.

**The autofill itself requires no code.** There is no API call to populate the
fields - declaring `contentType` on each `TextInput` is the whole integration.
The system recognises the field's purpose and offers matching saved data. Code
is only needed for the *save* direction.

## Implementation steps

1. **Model the form as an `@Observed` class** and pass it to the field component
   with `@ObjectLink`.
2. **Give every `TextInput` a `contentType`** matching what it collects. This is
   what activates smart fill.
3. **Bind two-way with `$$`** so edits land back on the model:
   `text: $$this.userInfo.userName`.
4. **Validate before submitting**, and return early with a message.
5. **Call `autoFillManager.requestAutoSave(uiContext, callback)`** inside a
   `try/catch`, and report success only from `onSuccess` (`HW-01-0043`).
6. **For the licence plate, use a custom keyboard** via
   `customKeyboard` on the input rather than filtering the system keyboard.

## Verified snippets

All snippets are from `CarMaintenance.zip`. Corrected forms are marked.

**The observed form model — `model/OwnerInfoModel.ets`** (as shipped)

```typescript
@Observed
export class UserInfo {
  userName: string;
  userID: string;
  phoneNumber: string;

  constructor(userName: string, userID: string, phoneNumber: string) {
    this.userName = userName;
    this.userID = userID;
    this.phoneNumber = phoneNumber;
  }
}
```

`@Observed` on the class plus `@ObjectLink` in the child is what makes `$$`
two-way binding to a **nested property** (`this.userInfo.userName`) propagate
back to the parent. Without `@Observed`, edits would not be seen.

**Autofill-enabled fields — `components/OwnerInfo.ets`** (corrected, see `HW-01-0045`)

```typescript
@Component
export struct OwnerInfo {
  @ObjectLink userInfo: UserInfo;

  build() {
    Column() {
      Row() {
        Text($r('app.string.name')).fontWeight(500);
        TextInput({ placeholder: $r('app.string.name_placeholder'), text: $$this.userInfo.userName })
          .contentType(ContentType.PERSON_FULL_NAME)
          .width('60%')
          .textAlign(TextAlign.End);
      }.textInputBorder();

      Row() {
        Text($r('app.string.id_card')).fontWeight(500);
        // FIX: shipped code hardcodes '请输入身份证号' here
        TextInput({ placeholder: $r('app.string.id_card_placeholder'), text: $$this.userInfo.userID })
          .contentType(ContentType.ID_CARD_NUMBER)
          .width('60%')
          .textAlign(TextAlign.End);
      }.textInputBorder();

      Row() {
        Text($r('app.string.phone_number')).fontWeight(500);
        // FIX: shipped code hardcodes '请输入手机号' here
        TextInput({ placeholder: $r('app.string.phone_placeholder'), text: $$this.userInfo.phoneNumber })
          .contentType(ContentType.PHONE_NUMBER)
          .width('60%')
          .textAlign(TextAlign.End);
      }.textInputBorder(true);
    }
    .alignItems(HorizontalAlign.Start);
  }
}
```

`.contentType(...)` is the entire smart-fill integration on the read side.
Useful values beyond these three: `ContentType.FULL_STREET_ADDRESS`,
`ContentType.EMAIL_ADDRESS`, `ContentType.NEW_PASSWORD`,
`ContentType.FORMAT_ADDRESS`. Pick the most specific one - the system matches
saved entries by content type, so a wrong or missing type means no suggestions.

**A reusable border style — `components/OwnerInfo.ets`** (as shipped)

```typescript
@Extend(Row)
function textInputBorder(isLast: boolean = false) {
  .border({
    width: { bottom: isLast ? LengthMetrics.vp(0) : LengthMetrics.vp(1) },
    color: { bottom: $r('app.color.textinput_border') }
  })
  .width(StyleConstants.FULL_WIDTH)
  .justifyContent(FlexAlign.SpaceBetween);
}
```

`@Extend(Row)` with a parameter is the clean way to express "all rows look like
this, except the last one has no divider". `LengthMetrics.vp(...)` is the typed
way to give a border width its unit.

**Saving the form back — `pages/MainPage.ets`** (corrected, see `HW-01-0043`)

```typescript
import { autoFillManager } from '@kit.AbilityKit';

.onClick(() => {
  if (!this.userInfo.userName || !this.userInfo.userID || !this.userInfo.phoneNumber) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.incomplete') });
    return;
  }
  // FIX: shipped code passes no callback, no try/catch, and toasts success
  // unconditionally on the next line.
  try {
    autoFillManager.requestAutoSave(this.getUIContext(), {
      onSuccess: () => {
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.bind_success') });
      },
      onFailure: () => {
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.bind_failed') });
      }
    });
  } catch (error) {
    Logger.error(`requestAutoSave threw: ${JSON.stringify(error)}`);
  }
});
```

`requestAutoSave` raises a **system prompt** asking the user whether to save -
it is not a silent write. The user can decline, so the outcome genuinely has two
branches and both need handling.

**Publishing insets — `entryability/EntryAbility.ets`** (as shipped)

```typescript
AppStorage.setOrCreate('topAvoid', avoidArea.topRect.height);
AppStorage.setOrCreate('bottomAvoid', avoidArea.bottomRect.height);
```

Read them with `@StorageProp`, not with `AppStorage.get(...)` inside `build()`
(`HW-01-0047`):

```typescript
@StorageProp('topAvoid') topAvoid: number = 0;
// ...
.padding({ top: this.getUIContext().px2vp(this.topAvoid) })
```

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`. Smart
fill runs in the system's autofill service, not in the app, so the app never
sees the user's stored data unless the user chooses to fill it in.

`build-profile.json5`:

```json5
"compatibleSdkVersion": "6.1.1(24)",
"targetSdkVersion": "6.1.1(24)"
```

## Constraints

- **API Version 24 Release and DevEco Studio 6.1.1 Release** - four API versions
  above the rest of this industry (`HW-01-0044`).
- Smart fill depends on the device's autofill service and a signed-in Huawei
  account; on a device without one, `contentType` is inert and the fields behave
  as ordinary inputs. Nothing in the sample detects that.
- `requestAutoSave` shows a system prompt; it cannot save silently.
- The fields collect an ID card number and a phone number. Treat both as
  sensitive: do not log them, and do not persist them outside the autofill
  service without saying so.

## Pitfalls

- **`HW-01-0043` — the save result is never checked.** `requestAutoSave` is
  called with no `AutoSaveCallback` and no `try/catch`, and "binding successful"
  is toasted on the next line regardless of whether the user accepted the system
  prompt.
- **`HW-01-0044` — this practice alone requires API 24.** It also renames
  `约束与限制` to `环境准备` and drops the SDK line, so the raised floor is easy
  to miss when combining it with the other practices in the industry.
- **`HW-01-0045` — two of three placeholders and both toasts are hardcoded
  Chinese** in a module that ships `en_US` resources and uses `$r` for the
  first field.
- **`HW-01-0046` — the project tree omits `entrybackupability`** and gives
  `EntryAbility.ets` the comment belonging to `StyleConstants.ets`.
- **`HW-01-0047` — `px2vp(AppStorage.get('topAvoid')) as number`** puts the type
  assertion on the return value rather than the argument, and runs both lookups
  inside `build()`.
- **`HW-01-0048` — `OwnerInfoOption` is declared and never used.** It describes
  exactly the row-driven design the form should have had; building the rows by
  hand instead is what let the placeholders drift apart.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-autofillmanager.md` - `requestAutoSave`, `AutoSaveCallback.onSuccess` / `onFailure`, and the reference's own `try/catch` example
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-autofillmanager
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `contentType` and the `ContentType` enum
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed` / `@ObjectLink`, required for `$$` on a nested property
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- Document's own pointer: `scenario-fusion-introduction-to-smart-fill`
- `AUTO-01` - the industry architecture; note the API version gap before combining
