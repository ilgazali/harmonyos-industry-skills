---
id: OFFICE-09
title: Visitor management - register a guest from the system contacts and auto-generate the pass QR code
industry: 05_office
doc: huawei_industry_tree/05_office/docs/09_guest_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
sample: huawei_industry_tree/05_office/downloads/GuestDemo.zip
kits: ["@kit.ContactsKit", "@kit.ScanKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: ["contact.selectContacts", "contact.Contact", "generateBarcode.createBarcode", "generateBarcode.CreateOptions", "generateBarcode.ErrorCorrectionLevel", "scanCore.ScanType", "UIContext.showTextPickerDialog", "UIContext.showDatePickerDialog", TextPickerResult, "UIContext.getMeasureUtils", "MeasureUtils.measureText", "UIContext.px2vp", Navigation, NavPathStack, NavDestination, "NavPathStack.pushPath", "NavPathStack.pushPathByName", "NavPathStack.pop", PopInfo, routerMap, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.AvoidAreaType.TYPE_SYSTEM", "window.on('avoidAreaChange')", "AppStorage.setOrCreate", "@StorageProp", "@Provide", "@Consume", "util.format", "resourceManager.getStringArrayByNameSync"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0051, HW-05-0052, HW-05-0053, HW-05-0054, HW-05-0055, HW-05-0056]
status: verified-with-fixes
---

## When to use

Load this card when an office app has to **book a visitor and issue them a
pass**: pre-register who is coming, when, why and where, then hand them a QR
code the door reader can scan.

Two capabilities carry the scenario:

- **`contact.selectContacts`** - the system contact picker fills in the guest's
  name and phone number, so the host does not retype them. It needs **no
  permission**: the picker is a system UI and the app only receives what the user
  selected.
- **`generateBarcode.createBarcode`** - the pass QR code is produced on device at
  submit time from the guest's identity and stored on the invitation record.

Everything else is `UIContext` dialogs (`showTextPickerDialog`,
`showDatePickerDialog`) and a `routerMap`-driven `Navigation` stack that returns
the completed record to the list through `pop(result, true)`.

## Feature checklist

The application must be able to:

- Show the registered invitations sorted by visit time.
- Open an "add invitation" page from the list, and receive the finished record
  back through the navigation stack's `onPop`.
- Fill the guest name and phone from the system contact picker, and let the host
  type them instead.
- Pick the phone area code and the visit reason from text pickers backed by
  string-array resources.
- Pick the visit date and time from a date picker bounded by a start/end range.
- Pick the visit location from a separate address page that returns its result
  through `pop`.
- Enable the submit button only when every field is filled, and validate the
  phone number against `^1\d{10}$`.
- Generate the pass QR code on submit and attach it to the record.
- Show the pass page with the guest name, a masked phone number and the QR code,
  regenerating the code if the record does not carry one.

## Architecture

Single `entry` HAP, four pages plus a model and a util:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | immersive layout, publishes `topRectHeight` / `bottomRectHeight` into `AppStorage`, loads `pages/GuestListPage` |
| `pages/GuestListPage.ets` | `@Entry`; owns the `NavPathStack` (`@Provide('pageStack')`), the invitation list and the sort |
| `pages/AddViewPage.ets` | the registration form: contact import, area-code / reason / date pickers, address jump, submit |
| `pages/AddressPage.ets` | address chooser, returns its selection via `pop` |
| `pages/GuestViewPage.ets` | the pass: guest details, masked phone, QR code |
| `model/DataModel.ets` | `GuestData`, `AddressData` and the two seed arrays |
| `utils/CommonUtil.ets` | `generateScanCode` and `formatDate` |
| `common/CommonConstants.ets` | layout and validation constants |

All three destinations are declared in `route_map.json` and reached by name; each
page file exports a global `@Builder` (`addViewPageBuilder`, `addressPageBuilder`,
`guestViewPageBuilder`) that the route entry names.

Result plumbing uses two different `NavPathStack` idioms, both worth knowing:

- **`pushPath` with an `onPop` callback** - `GuestListPage` pushes `AddViewPage`
  with `onPop: (info: PopInfo) => { ... info.result as GuestData ... }`, and the
  form calls `pop(this.guest, true)`.
- **`pushPathByName` with a trailing callback** - `AddViewPage` pushes
  `AddressPage` and reads `popInfo.result` in the callback.

Registration flow:

```
GuestListPage  + icon
  -> pushPath({ name: 'AddViewPage', onPop })
       AddViewPage.aboutToAppear
         resourceManager.getStringArrayByNameSync('area_code' | 'area_name' | 'visit_reasons')
         seed guest.areaCode / reasons / selectedTime
       contact row  -> contact.selectContacts() -> name + phoneNumbers[0]
       area code    -> showTextPickerDialog(areaNameList)   -> areaCodeList[index]
       reason       -> showTextPickerDialog(reasonList)     -> value.value
       visit time   -> showDatePickerDialog({start, end})   -> getTime() + formatDate
       location     -> pushPathByName('AddressPage')        -> popInfo.result
       submit       -> CommonUtil.generateScanCode(name) -> guest.scanCode
                    -> pop(this.guest, true)
  -> onPop: dataArr.push(guest); sortGuestList()
```

## Implementation steps

1. **Declare no permission.** `contact.selectContacts` opens the system picker
   and returns only the chosen contact, and `generateBarcode` is a local
   computation. The sample's `module.json5` has no `requestPermissions` block and
   the document has no 权限说明 section - consistent and correct.
2. **Declare the routes.** `"routerMap": "$profile:route_map"` in `module.json5`,
   with one entry per destination naming the page file and its global `@Builder`.
3. **Publish the insets.** In `onWindowStageCreate`, `await
   setWindowLayoutFullScreen(true)`, then read the **status-bar** inset from
   `AvoidAreaType.TYPE_SYSTEM` (not `TYPE_CUTOUT`, HW-05-0053), convert with
   `px2vp` **in the ability** so pages can use the value directly, and subscribe
   to `avoidAreaChange` with a matching `off` in `onWindowStageDestroy`
   (HW-05-0056).
4. **Seed the list without aliasing the module constant.** Spread the exported
   seed array into `@State` so the in-place `sort` and `push` do not mutate shared
   module data (HW-05-0054).
5. **Load the picker ranges from resources.**
   `resourceManager.getStringArrayByNameSync('area_code' | 'area_name' | 'visit_reasons')`
   in `aboutToAppear`, and pre-fill the guest with the first area code and reason
   so the record is never half-empty.
6. **Import from contacts.** `contact.selectContacts()` returns
   `Promise<Array<Contact>>`; read `data[0].name?.fullName` and
   `data[0].phoneNumbers[0].phoneNumber`, guarding both. Attach a `.catch()`
   (HW-05-0052) and do **not** log the returned payload (HW-05-0051).
7. **Drive the pickers from `UIContext`.** `showTextPickerDialog({ range,
   selected, onAccept })` for the area code and the reason, each with **its own**
   remembered index (HW-05-0055); `showDatePickerDialog({ start, end, selected,
   showTime, useMilitaryTime, onDateAccept })` for the visit time. Note the
   two-list pattern for the area code: `areaNameList` is what the picker shows and
   `areaCodeList[index]` is what gets stored.
8. **Gate the submit button.** Recompute `canTouch` from every field on each
   `onChange`, and validate the phone with `^1\d{10}$` - warning only once the
   input has reached full length so the user is not nagged mid-typing.
9. **Generate the pass code on submit.** `generateBarcode.createBarcode(name,
   options)` with `ScanType.QR_CODE` and `ErrorCorrectionLevel.LEVEL_H`, wrapped so
   a failure returns `undefined` rather than rejecting; attach it to the record and
   `pop(guest, true)`.
10. **Regenerate defensively on the pass page.** `onReady` reads
    `ctx.pathInfo.param as GuestData` and, when `scanCode` is absent, generates it
    again - so a record that arrived from storage still shows a code.
11. **Mask the phone on display.** Keep the raw number in the model and mask only
    at render time (`substring(0, 3) + '******' + substring(9)`).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The invitation list and its sort

`GuestDemo.zip#entry/src/main/ets/pages/GuestListPage.ets`

```ts
@Entry
@Component
struct GuestListPage {
  @State dataArr: GuestData[] = SAMPLE_GUEST_LIST;      // aliases the module constant - HW-05-0054
  @Provide('pageStack') pageStack: NavPathStack = new NavPathStack();
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;

  aboutToAppear() {
    this.sortGuestList();
  }

  /**
   * 对访客邀请列表数据排序
   */
  sortGuestList() {
    this.dataArr.sort((a: GuestData, b: GuestData) => {
      return b.selectedTimeMills - a.selectedTimeMills;
    });
  }

  // + icon: push the form and take its result back through onPop
  Image($r('app.media.ic_add'))
    .onClick(() => {
      this.pageStack.pushPath({
        name: 'AddViewPage', onPop: (info: PopInfo) => {
          if (info.result) {
            let guest = info.result as GuestData;
            this.dataArr.push(guest);
            this.sortGuestList();
          }
        }
      }, true);
    });
}
```

Corrected seeding:

```ts
@State dataArr: GuestData[] = [...SAMPLE_GUEST_LIST];
```

### Importing a contact

`GuestDemo.zip#entry/src/main/ets/pages/AddViewPage.ets`

```ts
import { contact } from '@kit.ContactsKit';

Row({ space: 10 }) {
  Image($r('app.media.ic_contact'));
  Text($r('app.string.contact_text'));
}
.onClick(() => {
  // 拉起通讯录信息
  let promise = contact.selectContacts();
  promise.then((data) => {
    let jsonString = JSON.stringify(data);
    hilog.info(0x0001, '%s', jsonString);          // PII in the log - HW-05-0051
    if (data.length > 0) {
      let contact: contact.Contact = data[0];
      this.guest.name = contact.name?.fullName ?? '';
      if (contact.phoneNumbers && contact.phoneNumbers.length > 0) {
        this.guest.phone = contact.phoneNumbers[0].phoneNumber;
      }
    }
  });                                               // no .catch - HW-05-0052
});
```

Corrected form:

```ts
contact.selectContacts()
  .then((data) => {
    if (data.length > 0) {
      const picked: contact.Contact = data[0];
      this.guest.name = picked.name?.fullName ?? '';
      if (picked.phoneNumbers && picked.phoneNumbers.length > 0) {
        this.guest.phone = picked.phoneNumbers[0].phoneNumber;
      }
      this.getButtonEnabled();
    }
  })
  .catch((err: BusinessError) => {
    hilog.error(0x0001, 'GuestDemo', 'selectContacts failed: %{public}s', JSON.stringify(err));
  });
```

Note the shadowing in the shipped code: `let contact: contact.Contact = data[0];`
reuses the module name as a local binding. Renaming the local (as above) keeps the
import readable.

### Text and date pickers from UIContext

`GuestDemo.zip#entry/src/main/ets/pages/AddViewPage.ets`

```ts
private select: number | number[] = 0;   // shared by both pickers - HW-05-0055
private reasonList: string[] = [];
private areaCodeList: string[] = [];
private areaNameList: string[] = [];

aboutToAppear(): void {
  let context = this.uiContext.getHostContext();
  if (context) {
    let resManager = context.resourceManager;
    this.areaCodeList = resManager.getStringArrayByNameSync('area_code');
    this.areaNameList = resManager.getStringArrayByNameSync('area_name');
    this.reasonList = resManager.getStringArrayByNameSync('visit_reasons');
  }
  if (this.areaCodeList && this.areaCodeList.length > 0) {
    this.guest.areaCode = this.areaCodeList[0];
  }
  this.guest.selectedTime = CommonUtil.formatDate(this.selectedDate);
  this.guest.selectedTimeMills = this.selectedDate.getTime();
  if (this.reasonList && this.reasonList.length > 0) {
    this.guest.reasons = this.reasonList[0];
  }
}

// area code: show the country names, store the dialling code at the same index
this.uiContext.showTextPickerDialog({
  range: this.areaNameList,
  selected: this.select,
  onAccept: (value: TextPickerResult) => {
    this.select = value.index;
    if (typeof value.index === 'number') {
      let selectedIndex = value.index as number;
      this.guest.areaCode = this.areaCodeList[selectedIndex];
    }
  }
});

// visit reason: store the displayed value directly
this.uiContext.showTextPickerDialog({
  range: this.reasonList,
  selected: this.select,
  onAccept: (value: TextPickerResult) => {
    this.select = value.index;
    this.guest.reasons = value.value as string;
  }
});

// visit time: bounded date + time picker
this.uiContext.showDatePickerDialog({
  start: new Date(CommonConstants.START_DATE),
  end: new Date(CommonConstants.END_DATE),
  selected: this.selectedDate,
  showTime: true,
  useMilitaryTime: false,
  onDateAccept: (value: Date) => {
    this.selectedDate = value;
    this.guest.selectedTimeMills = value.getTime();
    this.guest.selectedTime = CommonUtil.formatDate(value);
  }
});
```

### Submit gate and phone validation

`GuestDemo.zip#entry/src/main/ets/pages/AddViewPage.ets`

```ts
TextInput({ placeholder: $r('app.string.input_phonenumber_tip'), text: $$this.guest.phone })
  .maxLength(CommonConstants.MAX_PHONE_NUMBER_LENGTH)
  .type(InputType.PhoneNumber)
  .onChange(() => {
    if (this.guest.phone.match('^1\\d{10}$')) {
      this.getButtonEnabled();
    } else if (this.guest.phone.length === CommonConstants.MAX_PHONE_NUMBER_LENGTH &&
      !this.guest.phone.match('^1\\d{10}$')) {
      this.uiContext.getPromptAction().showToast({
        message: $r('app.string.invalid_phone_input_tip'),
        duration: CommonConstants.NORMAL_TIP_DURATION
      });
    }
  });

getButtonEnabled(): boolean {
  if (this.guest.selectedTime !== '' && this.guest.phone !== '' && this.guest.areaCode !== '' &&
    this.guest.name !== '' && this.guest.address !== '' && this.guest.reasons !== '') {
    this.canTouch = true;
  } else {
    this.canTouch = false;
  }
  return this.canTouch;
}

Button($r('app.string.submit_text'))
  .enabled(this.canTouch)
  .onClick(() => {
    // 提交数据时生成访客二维码
    CommonUtil.generateScanCode(this.guest.name)
      .then((scanCode) => {
        if (scanCode) {
          this.guest.scanCode = scanCode;
        }
        this.pageStack.pop(this.guest, true);
      });
  });
```

### QR generation helper

`GuestDemo.zip#entry/src/main/ets/utils/CommonUtil.ets`

```ts
import { generateBarcode, scanCore } from '@kit.ScanKit';

export class CommonUtil {
  static async generateScanCode(name: string): Promise<PixelMap | undefined> {
    let options: generateBarcode.CreateOptions = {
      scanType: scanCore.ScanType.QR_CODE,
      width: CommonConstants.DEFAULT_CODE_LENGTH,
      height: CommonConstants.DEFAULT_CODE_LENGTH,
      margin: CommonConstants.DEFAULT_CODE_MARGIN,
      level: generateBarcode.ErrorCorrectionLevel.LEVEL_H,
      backgroundColor: CommonConstants.DEFAULT_WHITE_COLOR,
      pixelMapColor: CommonConstants.DEFAULT_BLACK_COLOR,
    };
    try {
      return await generateBarcode.createBarcode(name, options);
    } catch (error) {
      hilog.error(DOMAIN_ID, TAG, `Failed to createBarCode. Code: ${error.code}, message: ${error.message}`);
      return undefined;
    }
  }
}
```

Swallowing the failure into `undefined` is deliberate here: the caller
(`AddViewPage` submit) treats a missing code as "attach nothing and continue",
and the pass page regenerates it later.

### The pass page

`GuestDemo.zip#entry/src/main/ets/pages/GuestViewPage.ets`

```ts
/**
 * 隐藏手机号信息
 */
hidePhoneNumber(originPhone: string) {
  if (!originPhone || originPhone === '') {
    return '';
  }
  let pre: string = originPhone.substring(0, 3);
  let suffix: string = originPhone.substring(9);
  return `${pre}******${suffix}`;
}

// ...
.onReady((ctx: NavDestinationContext) => {
  this.pageStack = ctx.pathStack;
  if (ctx.pathInfo.param) {
    // 判断之前是否已经生成二维码数据，若没有则再次尝试生成
    this.dataclass = ctx.pathInfo.param as GuestData;
    if (!this.dataclass.scanCode) {
      CommonUtil.generateScanCode(this.dataclass.name)
        .then((scanCode) => {
          if (scanCode) {
            this.dataclass.scanCode = scanCode;
            this.mScanCode = scanCode;
          }
        });
    }
  }
});
```

Text width measured once so the label column lines up:

```ts
aboutToAppear(): void {
  let uiContext = this.getUIContext();
  let measureUtil = uiContext.getMeasureUtils();
  this.maxWidth = uiContext.px2vp(measureUtil.measureText({
    textContent: $r('app.string.visit_location_text'),
    fontSize: $r('app.float.normal_font_size')
  }));
}
```

### Route declaration

`GuestDemo.zip#entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    { "name": "AddViewPage",   "pageSourceFile": "src/main/ets/pages/AddViewPage.ets",   "buildFunction": "addViewPageBuilder" },
    { "name": "AddressPage",   "pageSourceFile": "src/main/ets/pages/AddressPage.ets",   "buildFunction": "addressPageBuilder" },
    { "name": "GuestViewPage", "pageSourceFile": "src/main/ets/pages/GuestViewPage.ets", "buildFunction": "guestViewPageBuilder" }
  ]
}
```

## Permissions & config

`GuestDemo.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "routerMap": "$profile:route_map",
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages"
    // no requestPermissions block
  }
}
```

Notes on the config:

- **No permission is required or declared.** `contact.selectContacts` is
  documented with no "Required permissions" line - the system picker returns only
  what the user selected - and `generateBarcode.createBarcode` runs locally. The
  document correspondingly has no 权限说明 section, which is consistent.
- Reading the address book directly (`contact.queryContacts` and friends) *would*
  need `ohos.permission.READ_CONTACTS`; the picker route exists precisely to avoid
  that.
- `routerMap` is mandatory for the three `pushPath` / `pushPathByName` calls, and
  each `buildFunction` must be a **global** `@Builder` exported from its page file.
- Picker ranges live in string-array resources: `area_code`, `area_name` and
  `visit_reasons`.
- `build-profile.json5` pins `compatibleSdkVersion` / `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Devices.** `deviceTypes` is `["phone", "tablet", "2in1"]`; `selectContacts` is
  documented for Phone, PC/2in1, Tablet and Wearable.
- **The pass code encodes only the guest name.** `generateScanCode(this.guest.name)`
  puts the plain name into the QR payload - there is no token, signature or
  expiry, so it is a demonstration of the mechanism, not a usable access
  credential.
- **Nothing is persisted.** The invitation list lives only in `@State` over a
  module-level seed array; closing the app loses every added guest.
- **Phone validation is mainland-China specific.** `^1\d{10}$` accepts only an
  11-digit number starting with 1, even though the form offers an area-code
  picker for other countries.
- **`hidePhoneNumber` assumes an 11-digit number.** `substring(0, 3)` +
  `'******'` + `substring(9)` reconstructs 11 characters; a shorter imported
  number produces a mask of the wrong length (it does not leak digits).
- **`showTextPickerDialog` / `showDatePickerDialog` are `UIContext` methods.**
  They must be called from a component that can reach a `UIContext`, which is why
  the page caches `uiContext: UIContext = this.getUIContext()`.

## Pitfalls

- **Serializing the picked contact into a log line is incorrect.**
  `hilog.info(0x0001, '%s', JSON.stringify(data))` writes the contact's name and
  phone numbers to the system log, and because the payload lands in the *format*
  slot the `{private}` filtering never applies. Log a count, or nothing.
  (HW-05-0051)
- **`contact.selectContacts().then(...)` with no `.catch()` is incorrect.**
  Dismissing the picker rejects the promise; the fields silently stay empty and
  nothing is reported. (HW-05-0052)
- **Reading the top inset from `AvoidAreaType.TYPE_CUTOUT` is incorrect** when the
  comment and the usage both mean the status bar. `TYPE_SYSTEM` "usually indicates
  the status bar area"; `TYPE_CUTOUT` is the notch and is zero on devices without
  one, collapsing every page's top padding to the hard-coded `+ 15`.
  (HW-05-0053)
- **`@State dataArr = SAMPLE_GUEST_LIST` is incorrect.** It aliases the exported
  module constant, and the subsequent `sort()` and `push()` mutate it in place, so
  the "sample data" is permanently reordered and grown. Spread it:
  `[...SAMPLE_GUEST_LIST]`. (HW-05-0054)
- **Sharing one `select` field between two `showTextPickerDialog` calls is
  incorrect.** The area-code picker and the reason picker index different lists,
  so each one reopens on the other's last choice - and can be handed an index out
  of its own range. Give each dialog its own field. (HW-05-0055)
- **`on('avoidAreaChange')` without `off`, and an un-awaited
  `setWindowLayoutFullScreen`, are incorrect.** Release the subscription in
  `onWindowStageDestroy` and await the layout switch before reading the avoid
  area. (HW-05-0056)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/03_application-services/js-apis-contact.md` -
  `contact.selectContacts10+` (both overloads), its device list, and the absence
  of any required permission.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-contact
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType`: `TYPE_SYSTEM` as the status-bar area and `TYPE_CUTOUT` as the
  cutout area.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on('avoidAreaChange')` and
  `off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` -
  `info(domain, tag, format, ...args)` and the `{public}` / `{private}` filtering
  rules.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `showTextPickerDialog`, `showDatePickerDialog`, `getMeasureUtils` and `px2vp`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/04_media/scan-generatebarcode.md` -
  `createBarcode`, `CreateOptions` and `ErrorCorrectionLevel`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-generatebarcode
