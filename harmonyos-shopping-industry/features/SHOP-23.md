---
id: SHOP-23
title: Delivery address book - list, edit, and fill the form from contacts, location or the clipboard
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/23_address_manager.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_manager-0000002366628616
sample: huawei_industry_tree/16_shopping/downloads/AddressManager.zip
kits: ["@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ContactsKit", "@kit.CoreFileKit", "@kit.LocationKit", "@kit.PerformanceAnalysisKit"]
apis: ["contact.selectContacts", "geoLocationManager.getCurrentLocation", "geoLocationManager.getAddressesFromLocation", ReverseGeoCodeRequest, "pasteboard.createData", "pasteboard.getSystemPasteboard", MIMETYPE_TEXT_PLAIN, "abilityAccessCtrl.createAtManager", requestPermissionsFromUser, requestPermissionOnSetting, checkAccessToken, "bundleManager.getBundleInfoForSelf", List, ListItem, ForEach, Navigation, NavPathStack, NavPathInfo, NavDestination, CustomDialogController, LocalStorage, "@LocalStorageLink", "@Link", Checkbox, TextArea, "@Extend"]
permissions: [ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0025, HW-16-0013, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a checkout flow needs a **delivery address book**: a list
with a default entry, an add/edit form, delete behind a confirmation, and copy
to the clipboard. It is the least glamorous screen in any commerce app and the
one users abandon carts over, because typing a full address on a phone is
miserable.

The interesting part is the three shortcuts the form offers instead of typing.
A contacts icon on the phone row raises the system contact picker and fills
name and number in one tap. A location icon on the region row reverse-geocodes
the current position into province/city/district plus street. A copy action on
each list row puts the whole address on the system clipboard as plain text.
Each is a small, independent API, and each transfers directly to any other
"long form the user should not have to type" - a profile, a delivery note, an
insurance claim.

The card also covers the permission utility the sample ships alongside: it
checks first, requests once, and falls back to the settings sheet when the
system will no longer show a dialog. It is the most careful permission code in
this industry and worth lifting on its own.

## Feature checklist

- Address list showing name, phone, region + detailed address for each entry.
- A checkbox per row marks the default address; ticking one clears the others.
- Each row carries 删除 (delete), 复制 (copy) and 修改 (modify) actions.
- Delete opens a custom confirmation dialog; only 确定 removes the entry.
- Copy writes a four-line labelled address to the system clipboard and toasts
  success or failure.
- Modify pushes the edit page with the row index; the create button pushes it
  with `-1` for a blank form.
- The edit form has four required fields plus a home/company tag and a
  "set as default" checkbox; save stays disabled until all four are non-empty.
- The contacts icon on the phone row raises the system picker and fills name
  and phone from the chosen contact.
- The location icon on the region row requests location permission, resolves
  the current position and fills region and detailed address from it.
- An empty list shows a 暂无数据 (no data) placeholder.

## Architecture

One `entry` module. Two pages joined by a `Navigation` stack, with the form
body factored out into a view so the create and edit cases are one component.

```
entry/src/main/ets
├── common/Constants.ets              layout constants + the Chinese field labels used by copy
├── entryability/EntryAbility.ets     full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model
│   ├── AddressInfoModel.ets          AddressInfoModel + 3 seed addresses + DetailAddressItem
│   └── ContactsModel.ets             the shape selectContacts actually returns
├── pages
│   ├── AddressManagerMainPage.ets    @Entry: the list, copy, delete, default-address logic
│   └── EditAddressPage.ets           NavDestination: hosts the form, owns save
├── utils
│   ├── CustomDialog.ets              the delete confirmation
│   ├── Logger.ets
│   └── PermissionsRequest.ets        check -> request -> requestPermissionOnSetting
└── view/DetailAddressView.ets        the form body: four rows, contacts, location (272 lines)
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is that the address list is a
`LocalStorage` owned by the entry page and reached from the edit page by key:

```typescript
let para: Record<string, AddressInfoModel[]> = { 'addressInfoList': ADDRESS_INFO_MODEL };
let storage: LocalStorage = new LocalStorage(para);

@Entry(storage)
struct AddressManagerMainPage {
  @LocalStorageLink('addressInfoList') addressInfosList: AddressInfoModel[] = [];
```

`EditAddressPage` and the delete dialog declare the same
`@LocalStorageLink('addressInfoList')`. So the destination edits the list
directly and the list page re-renders when it pops - no result callback, no
event bus, and nothing global: the storage is scoped to this `@Entry` page and
its children, unlike `AppStorage`. For a master-detail pair inside one
`Navigation` that is the right amount of sharing.

The form is data-driven: `DetailAddressView` builds its four rows from a
`DetailAddressItem[]` (index, title, placeholder, optional icon) and dispatches
on `item.index` in `getText`, `getAddressInfoModel` and the icon's `onClick`.
The rows stay identical for free, at the cost of an index-based `if/else`
ladder in three places - fine at four fields, a problem at fourteen.

## Implementation steps

1. **Seed a `LocalStorage` with the address array and attach it to the entry
   page** with `@Entry(storage)`; link it from every participating component
   with `@LocalStorageLink('addressInfoList')`.
2. **Push the edit page with the row index as the param**
   (`pushPath(new NavPathInfo('editAddressPage', index))`), using `-1` for
   create, and read it back in `onReady` from `context.pathInfo.param`.
3. **Deep-copy the model before editing** (`JSON.parse(JSON.stringify(...))`),
   so cancelling an edit leaves no half-typed values in the list.
4. **Validate on every `onChange`** and drive the save button's `enabled` from
   the resulting boolean rather than checking on submit.
5. **Write back with `splice(index, 1, model)`, not element assignment** -
   ArkUI observes array mutations through the container methods, and a plain
   `list[i] = x` does not re-render.
6. **Fill name and phone with `contact.selectContacts({ isMultiSelect: false })`**;
   the picker runs in the Contacts app, so you need no permission of your own.
7. **Request location, then check `authResults`, then locate** - both
   `LOCATION` and `APPROXIMATELY_LOCATION`, because the precise permission is
   refused unless the approximate one is requested with it. The document's
   step-4 snippet drops this check and puts an `await` inside a non-`async`
   callback, so it does not compile (`HW-16-0025`, an instance of the
   corpus-wide excerpt defect `HW-16-0013`).
8. **Reverse-geocode with `maxItems: 1`** and assemble the region from
   `administrativeArea + subAdministrativeArea + subLocality`, defaulting the
   optional parts to `''`.
9. **Copy with the ArkTS pasteboard**, not the C API the document links
   (`HW-16-0025`): `pasteboard.createData(MIMETYPE_TEXT_PLAIN, text)` then
   `getSystemPasteboard().setData(...)`, toasting on both branches.
10. **Await the permission utility before checking its result.** The shipped
    `determineLocationPermissions` does not - see the snippet.

## Verified snippets

All snippets are from `AddressManager.zip`. Corrected forms are marked.

**Permission, then location, then address - `entry/src/main/ets/view/DetailAddressView.ets`** (corrected)

```typescript
const REQUEST_PERMISSIONS: Array<Permissions> = [
  'ohos.permission.APPROXIMATELY_LOCATION',
  'ohos.permission.LOCATION'
];

async determineLocationPermissions() {
  let permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
  await PermissionsRequest.commonRequestPermissions(this.getUIContext(), permissions);  // FIX: shipped call
  let permissionAllowed = await PermissionsRequest.checkPermissions(permissions);       // is not awaited
  if (permissionAllowed) {
    this.getCurrentLocation();
  }
}

async getCurrentLocation() {
  let atManager = abilityAccessCtrl.createAtManager();
  try {
    atManager.requestPermissionsFromUser(this.context, REQUEST_PERMISSIONS).then((data) => {
      if (data.authResults[0] !== 0 || data.authResults[1] !== 0) {
        LOGGER.info('request data is null');
        return;                                   // absent from the document's snippet
      }
      let locationChange = async (err: BusinessError, location: geoLocationManager.Location) => {
        if (location.latitude === 0 && location.longitude === 0) {
          LOGGER.error('latitude or longitude is null, err ' + JSON.stringify(err));
          return;
        }
        let reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest = {
          'latitude': location.latitude,
          'longitude': location.longitude,
          'maxItems': 1
        };
        const result = await geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest);
        if (result[0].placeName) {
          const obj = result[0];
          let city = obj.subAdministrativeArea || '';
          this.addressInfoModel.region = obj.administrativeArea + city + obj.subLocality;
          let roadName = obj.roadName || '';
          let subRoadName = obj.subRoadName || '';
          this.addressInfoModel.detailAddress = roadName + subRoadName;
        }
      };
      geoLocationManager.getCurrentLocation(locationChange);
    }).catch((err: Error) => {
      LOGGER.error(`req permissions failed, error.code: ${err.name}, error message: ${err.message}.`);
    });
  } catch (err) {
    LOGGER.error(`req permissions failed, error.code: ${err.code}, error message: ${err.message}.`);
  }
}
```

**The callback must be `async`, and that is the whole point of `HW-16-0025`.**
`getCurrentLocation` takes an `AsyncCallback`, and reverse geocoding inside it
is itself asynchronous, so the callback has to carry the `async` modifier
before it can `await getAddressesFromLocation`. The zip declares it
(`let locationChange = async (err, location) => {...}`); the document's excerpt
drops the modifier while keeping the `await`, which is not valid TypeScript.

**`authResults` is the only real answer.** `requestPermissionsFromUser`
resolves successfully even when the user refuses, so the array of `0`/`-1`
codes is the one thing that says whether locating is allowed - the sample
checks both entries and returns early, and the document's excerpt omits the
check entirely.

Two things here are worth not copying. The method **requests the permissions a
second time**, although `determineLocationPermissions` has just run the full
check/request/settings sequence through `PermissionsRequest`. And
`commonRequestPermissions` is `async` but is called **without `await`**, so
`checkPermissions` on the next line runs while the system dialog is still up:
on a first cold tap the check answers "denied" and nothing happens until the
user taps the icon again. Awaiting it, as above, is a one-word fix.

**The permission utility - `entry/src/main/ets/utils/PermissionsRequest.ets`** (as shipped)

```typescript
async commonRequestPermissions(context: UIContext, permissions: Array<Permissions>) {
  let isPermission: boolean = await this.checkPermissions(permissions);
  if (!isPermission) {
    // First authorization
    let isDialogShown = await this.requestPermissions(context, permissions);
    if (isDialogShown !== true) {
      // Second authorization: no dialog was shown, so the user has refused for good
      this.requestPermissionsOnSetting(context, permissions);
    }
  }
}

async checkPermissions(permissions: Array<Permissions>) {
  for (let permission of permissions) {
    let grantStatus: abilityAccessCtrl.GrantStatus = await this.checkAccessToken(permission);
    if (grantStatus !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return false;                       // all-or-nothing: one denial fails the set
    }
  }
  return true;
}

async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenId: number = 0;
  try {
    let bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    let appInfo: bundleManager.ApplicationInfo = bundleInfo.appInfo;
    tokenId = appInfo.accessTokenId;
    grantStatus = await atManager.checkAccessToken(tokenId, permission);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x000, 'testTag', `Failed to check access token  ${err.code}, message is ${err.message}`);
  }
  return grantStatus;
}
```

**This is the correct shape of a user_grant permission flow, and it is worth
lifting wholesale.** `checkPermissions` accumulates across the set and returns
`false` on the first denial rather than letting the last permission win.
`checkAccessToken` initialises both `tokenId` and `grantStatus` before the
`try`, so a failed `getBundleInfoForSelf` degrades to `PERMISSION_DENIED`
instead of returning `undefined` typed as a `GrantStatus` - compare
`TOUR-03`'s `HW-09-0020`, which is the same code with the initialisers
missing. And `GET_BUNDLE_INFO_WITH_APPLICATION` is required: without that flag
`appInfo` is not populated and there is no token to check.

The `dialogShownResults` branch is the piece most implementations miss. After
a permanent refusal `requestPermissionsFromUser` still resolves and shows
nothing, so an app that only looks at the result appears to work while the
user sees no dialog. `isDialogShown !== true` is the cue to call
`requestPermissionOnSetting`, which opens the settings sheet directly.

**Copy to clipboard - `entry/src/main/ets/pages/AddressManagerMainPage.ets`** (as shipped)

```typescript
import { BusinessError, pasteboard } from '@kit.BasicServicesKit';

splicingAddress(addressModel: AddressInfoModel): void {
  this.copyText =
    Constants.CONSIGNEE + '：' + addressModel.name + '\n' +           // 姓名 (name)
    Constants.PHONE_NUMBER + '：' + addressModel.phone + '\n' +       // 手机号 (phone)
    Constants.REGION + '：' + addressModel.region + '\n' +            // 所在地区 (region)
    Constants.DETAILED_ADDRESS + '：' + addressModel.detailAddress;   // 详细地址
}

copyAddress(index: number): void {
  this.splicingAddress(this.addressInfosList[index]);
  let pasteData = pasteboard.createData(pasteboard.MIMETYPE_TEXT_PLAIN, this.copyText);
  let systemPasteboard = pasteboard.getSystemPasteboard();
  systemPasteboard.setData(pasteData).then(() => {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.copy_success') });
  }).catch((err: BusinessError) => {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.copy_failed') });
    LOGGER.error('selectContact callback, errCode:' + err.code + ', errMessage:' + err.message);
  });
}
```

**The import line is the finding.** `pasteboard` comes from
`@kit.BasicServicesKit` - the ArkTS module - while the document links
"Pasteboard" twice to `capi-pasteboard`, the NDK C API, which is a different
surface with different types (`HW-16-0025`). An ArkTS reader following those
links finds `OH_Pasteboard_*` functions that do not apply.

The labelled four-line `label：value` format is deliberate - a bare
concatenation is unreadable when pasted into a chat, and the labels give the
receiving app something to split on. They are plain string constants in
`Constants.ets`, not resources, which is the one thing to change before
shipping: the copied text is user-visible content in a fixed language.

**The list row and its actions - same file** (as shipped, with the row's
redundant `Flex`/`Row` nesting collapsed for reading; attributes are unchanged)

```typescript
ForEach(this.addressInfosList, (item: AddressInfoModel, index: number) => {
  ListItem() {
    Column() {
      Row() {
        Text(item.name);
        Text(item.phone);
      }
      Text(`${item.region} ${item.detailAddress}`)
        .opacity(Constants.OPACITY);
      Row() {
        Checkbox()
          .select(item.isDefaultAddress)
          .onChange(() => { this.setDefaultAddress(item, index); });
        Text(item.isDefaultAddress === true ? $r('app.string.already_default') :
          $r('app.string.set_default'));
        Blank();
        Text($r('app.string.delete')).textStyle()
          .onClick(() => { this.deleteAddress(index); });
        Text($r('app.string.copy')).textStyle()
          .onClick(() => { this.copyAddress(index); });
        Text($r('app.string.modify')).textStyle()
          .onClick(() => { this.navPathStack.pushPath(new NavPathInfo('editAddressPage', index)); });
      }
      .width(Constants.FULL_PERCENT);
      Divider();
    }
    .alignItems(HorizontalAlign.Start);
  }
});
```

Two details carry the row. `Blank()` between the default-address cluster and
the action cluster is what pins the three actions to the right edge without a
width calculation. And the three actions are `Text` with a shared `@Extend`
(`textStyle`) rather than `Button`s - which gives a flat, low-emphasis strip
appropriate for row-level actions, at the cost of the accessibility and press
feedback a `Button` would bring for free.

`setDefaultAddress` enforces the single-default invariant by walking the list
and clearing every other entry with `splice(int, 1, model)`. The `splice` is
not stylistic: assigning `this.addressInfosList[int] = model` would not be
observed and the other rows' checkboxes would not update.

Note the `ForEach` here has **no key generator**. With rows identified by
index only, a delete of any but the last entry re-renders more of the list
than it needs to; add `(item, index) => index + item.phone` or a stable id.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",   // approximate location
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.LOCATION",                 // precise location
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory and the
  `reason` resource must exist in `resources/base/element/string.json`.
- `LOCATION` (precise) is refused unless `APPROXIMATELY_LOCATION` is requested
  in the same array. The document's 权限说明 labels them correctly.
- `when: "inuse"` is right: the address form needs a position once, on tap,
  and must not hold location in the background.
- **No permission is declared for contacts, and none is needed.**
  `contact.selectContacts` raises the system picker in the Contacts app and
  returns only what the user chose - that is the whole reason to prefer it
  over the contacts query API. The clipboard write is unprivileged too.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Nothing persists.** The list lives in a `LocalStorage` seeded from three
  hardcoded entries in `AddressInfoModel.ets`; closing the app restores the
  seed. A real address book needs a relational store or preferences behind the
  same `@LocalStorageLink` surface.
- Reverse geocoding requires a network connection and the fields it returns
  (`subAdministrativeArea`, `subLocality`, `roadName`, `subRoadName`) are all
  optional - the sample defaults three of them to `''` but concatenates
  `administrativeArea` and `subLocality` unguarded, so an incomplete result can
  produce `undefined` inside the region string.
- `ContactsModel.ets` re-declares the picker's return shape by hand and the
  data is converted with `JSON.parse(JSON.stringify(data))`, so any shape
  change in Contacts Kit is accepted silently.
- The home/company tag round-trips through a `tag: number` and an
  `isHome: boolean` with an inverted mapping in `EditAddressPage`'s save
  handler; it is the most fragile part of the form.
- `setColorMode` is forced to light; the row colours are `Color.White` and
  `Color.Black` literals with no dark palette.

## Pitfalls

- **`HW-16-0025`** (E/low, confirmed): the document's step-4 snippet puts
  `await geoLocationManager.getAddressesFromLocation(...)` inside a
  non-`async` arrow callback, which does not compile, and drops the
  `data.authResults` check the zip performs before locating. Separately, the
  document links "Pasteboard" to `capi-pasteboard` (the native C API) in both
  the step-2 text and the reference list, although the sample imports the
  ArkTS `pasteboard` module from `@kit.BasicServicesKit`. Fix: regenerate the
  step-4 snippet from `DetailAddressView.ets` (keeping `async` and the
  `authResults` guard) and repoint both links at `js-apis-pasteboard`.
- **`HW-16-0013`** (E/medium, confirmed): this document is one of the
  enumerated instances of a corpus-wide defect - the excerpt pipeline
  abridges snippets by removing structurally necessary code (closing braces,
  `catch` blocks, `async` modifiers, `return`s) rather than eliding only
  bodies, so a large share of published snippets no longer parse. Here it is
  the missing `async` in step 4. Treat every snippet in these documents as
  illustrative and take the zip as the source of truth.
- `determineLocationPermissions` calls the `async`
  `commonRequestPermissions` without `await`, so `checkPermissions` runs while
  the system dialog is still open and the first tap on the location icon
  silently does nothing. Add the `await`.
- `getCurrentLocation` then requests both permissions again, producing a second
  dialog in the denied case; and the address `ForEach` has no key generator, so
  a mid-list delete re-renders more of the list than it should.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItem`, `ForEach` keying
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `getCurrentLocation`, `getAddressesFromLocation`, `ReverseGeoCodeRequest` and its optional fields
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-references/03_system/js-apis-pasteboard.md` - the ArkTS pasteboard: `createData`, `MIMETYPE_TEXT_PLAIN`, `getSystemPasteboard().setData`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pasteboard
- `documentation/harmonyos-references/03_system/capi-pasteboard.md` - the C API the document links by mistake; here for comparison only
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-pasteboard
- `documentation/harmonyos-references/03_application-services/js-apis-contact.md` - `selectContacts` and the picker's return shape
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-contact
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the user_grant request flow and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - which of the two location permissions is which
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `@Entry(storage)` and `@LocalStorageLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `@CustomDialog` and `CustomDialogController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `huawei_industry_tree/16_shopping/docs/23_address_manager.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_manager-0000002366628616
- `TOUR-03` - the same permission utility with the initialisers missing (`HW-09-0020`)
