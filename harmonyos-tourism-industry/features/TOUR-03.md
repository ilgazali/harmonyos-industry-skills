---
id: TOUR-03
title: Location permission bubble - prompt for location with a Popup instead of a dialog
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
sample: huawei_industry_tree/09_tourism/downloads/LocationPermissionPrompt.zip
kits: ["@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit", "@kit.ArkUI"]
apis: [bindPopup, PopupOptions, Placement, "abilityAccessCtrl.createAtManager", checkAccessToken, requestPermissionsFromUser, requestPermissionOnSetting, dialogShownResults, "bundleManager.getBundleInfoForSelf", Tabs, TabContent, onContentWillChange, "@StorageProp", "window.getWindowAvoidArea"]
permissions: ["ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-09-0016, HW-09-0017, HW-09-0018, HW-09-0019, HW-09-0020, HW-09-0021, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card when a feature **needs a permission but the app still works
without it**, and an unsolicited system dialog on first launch would be too
much. The pattern: check quietly on entry, and if the permission is missing,
attach a small bubble to the UI element that depends on it - here the city
name in the top left - carrying one sentence and a button that raises the real
request.

It is the right shape for location in travel and tourism, where the city
picker has a sensible default and precise location only makes the nearby list
better. It generalises to any "enhance, don't block" permission: calendar for
itinerary import, photos for a review, notifications for a booking reminder.

The complementary piece is `requestPermissionOnSetting`: once the user has
refused permanently, the ordinary request no longer shows a dialog, and this
sample correctly detects that and opens the settings sheet instead.

## Feature checklist

- On entering the home page, check whether both location permissions are
  granted, without showing anything if they are.
- If either is missing, show a bubble anchored under the location text in the
  top left of the search bar.
- The bubble carries an explanation, a 去开启 (go and enable) button, and a
  close cross.
- The button raises the system permission request.
- If the user has already refused permanently, open the permission settings
  sheet instead of a request that would not appear.
- Once the permission is granted, the bubble goes away and does not come back.
- Closing the bubble by the cross, by tapping outside, or by pressing back all
  dismiss it for this visit.

## Architecture

One `entry` module, four ArkUI files and a constants file. No model layer -
the whole feature is a boolean.

```
entry/src/main/ets
├── components/LocationPopup.ets   the location text + the bubble + the whole permission flow
├── constant/StyleConstant.ets     numeric literals kept out of the layout
├── entryability/EntryAbility.ets  full-screen layout, avoid areas -> AppStorage
├── entrybackupability/
├── pages/TabIndex.ets             @Entry, the four bottom tabs
└── view/
    ├── HomePageView.ets           the home tab: LocationPopup + Search + TabMainView
    └── TabMainView.ets            the home tab's own inner tabs and content
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `LocationPopup` owns both the
anchor and the permission logic. It is not a dialog raised by a page; it is
the location label itself, which happens to know whether it is allowed to
show a real location. That keeps the permission concern next to the UI that
needs it, and the host (`HomePageView`) just places it:

```typescript
LocationPopup({ 'locationText': $r('app.string.location_text') });
```

`@Prop locationText: Resource` is the only input. Everything else -
`handlePopup`, the check, the request, the settings fallback - is internal.

## Implementation steps

1. **Declare both permissions** in `module.json5` with `reason`, `usedScene`
   and `when: "inuse"`. `LOCATION` is the **precise** permission and
   `APPROXIMATELY_LOCATION` the **approximate** one - the document has these
   two the wrong way round (`HW-09-0018`).
2. **Check in `aboutToAppear`,** `await`ed, so the bubble state is settled
   before the first frame that could show it.
3. **Accumulate the check across permissions**, do not overwrite per
   iteration (`HW-09-0017`).
4. **Bind the bubble to the anchor text** with `bindPopup(this.handlePopup, {...})`,
   `placement: Placement.Bottom` and `enableArrow: true` so the arrow points
   at the element the permission is about.
5. **Use `showInSubWindow: true`** so the bubble can extend past the page
   bounds near the top edge.
6. **Reset the flag from `onStateChange`** when the popup becomes invisible,
   otherwise a dismissal by tap-outside leaves `handlePopup` true and the
   bubble cannot be shown again.
7. **On the button, request; then re-check** so the bubble closes on success
   (`HW-09-0019`).
8. **Read `dialogShownResults`** to detect a permanent refusal and fall back
   to `requestPermissionOnSetting`.

## Verified snippets

All snippets are from `LocationPermissionPrompt.zip`. Corrected forms are marked.

**The bubble — `entry/src/main/ets/components/LocationPopup.ets`** (as shipped)

```typescript
@Component
export default struct LocationPopup {
  @Prop locationText: Resource;
  @State handlePopup: boolean = false;

  @Builder
  locationPopupBuilder() {
    Row({ space: StyleConstant.SPACE_TWO }) {
      Text($r('app.string.popup_text'))
        .fontSize($r('app.float.vp_fourteen'))
        .fontColor(Color.White)
        .height($r('app.float.forty'))
        .textOverflow({ overflow: TextOverflow.MARQUEE })
      Button($r('app.string.pop_button_text'))
        .backgroundColor('#0A59F7')
        .controlSize(ControlSize.SMALL)
        .onClick(() => { this.requestPermissions(); })
      Image($r('app.media.ic_fork'))
        .width($r('app.float.vp_fifteen'))
        .height($r('app.float.vp_fifteen'))
        .onClick(() => { this.handlePopup = false; })
    }
    .justifyContent(FlexAlign.Center)
    .borderRadius(StyleConstant.SEVEN_BORDER_RADIUS)
    .zIndex(StyleConstant.FIVE_HUNDRED)
  }

  build() {
    Column() {
      Text(this.locationText)
        .width($r('app.float.vp_fifty'))
        .textOverflow({ overflow: TextOverflow.Ellipsis })
        .maxLines(StyleConstant.TEXT_MAX_LINES)
        .bindPopup(this.handlePopup, {
          builder: this.locationPopupBuilder,
          placement: Placement.Bottom,
          offset: { x: StyleConstant.TWELVE, y: StyleConstant.MINUS_FIVE },
          enableArrow: true,
          showInSubWindow: true,
          autoCancel: true,
          popupColor: 'rgba(56, 55, 54,0.9)',
          backgroundBlurStyle: BlurStyle.NONE,
          width: '91%',
          radius: StyleConstant.TEN,
          onStateChange: (e) => {
            if (!e.isVisible) {
              this.handlePopup = false;      // keeps the flag in step with a tap-outside dismissal
            }
          }
        })
    }
  }
}
```

**Three options carry the design.** `enableArrow` is what makes it a pointer
rather than a floating toast - the arrow names the element the permission is
for. `showInSubWindow` lets a bubble anchored near the status bar draw outside
the page. `onStateChange` is not optional: `bindPopup` reads the boolean but
does not write it back, so without this handler a dismissal by tap-outside
leaves `handlePopup` true and no later assignment can re-show the bubble.

The bubble body being a `@Builder` rather than a string is what allows the
button and the close cross inside it.

**The permission check — same file** (corrected, see `HW-09-0017`, `HW-09-0020`)

```typescript
import AbilityAccessCtrl, { Permissions } from '@ohos.abilityAccessCtrl';
import { bundleManager } from '@kit.AbilityKit';

class CommonConstants {
  static readonly LOCATION: Permissions = 'ohos.permission.LOCATION';
  static readonly APPROXIMATELY_LOCATION: Permissions = 'ohos.permission.APPROXIMATELY_LOCATION';
}

private permissions: Array<Permissions> = [
  CommonConstants.LOCATION,
  CommonConstants.APPROXIMATELY_LOCATION
];

async aboutToAppear() {
  await this.checkPermissions();
}

async checkPermissions(): Promise<void> {
  let missing = false;                                   // FIX: shipped loop overwrites per pass
  for (const permission of this.permissions) {           // FIX: shipped loop is `i < 2`
    if (await this.checkAccessToken(permission) !== AbilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      missing = true;
      break;
    }
  }
  this.handlePopup = missing;
}

async checkAccessToken(permission: Permissions): Promise<AbilityAccessCtrl.GrantStatus> {
  const atManager = AbilityAccessCtrl.createAtManager();
  let grantStatus: AbilityAccessCtrl.GrantStatus = AbilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenID: number = 0;                               // FIX: shipped code leaves both unassigned
  try {
    const bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    tokenID = bundleInfo.appInfo.accessTokenId;
  } catch (err) {
    hilog.error(0x0000, 'getBundleInfoForSelf failed', `error: ${err}`);
    return grantStatus;                                  // FIX: shipped code falls through with tokenID!
  }
  try {
    grantStatus = await atManager.checkAccessToken(tokenID, permission);
  } catch (err) {
    hilog.error(0x0000, 'checkAccessToken failed', `error: ${err}`);
  }
  return grantStatus;
}
```

`checkAccessToken` needs the app's own token ID, and the only way to it is
`getBundleInfoForSelf` with `GET_BUNDLE_INFO_WITH_APPLICATION` - the flag
matters, without it `appInfo` is not populated. **Accumulate, never assign
per iteration**: the two location permissions are granted independently, so
the loop must be an OR over "missing", not a running assignment.

**The request, with the permanent-refusal fallback — same file** (corrected, see `HW-09-0019`)

```typescript
async requestPermissions() {
  const atManager = AbilityAccessCtrl.createAtManager();
  const context = this.getUIContext().getHostContext() as Context;
  await atManager.requestPermissionsFromUser(context, this.permissions).then((data) => {
    const showResults = data.dialogShownResults as [];
    for (let i = 0; i < showResults?.length; i++) {
      if (showResults[i] === false) {          // no dialog appeared: refused for good
        atManager.requestPermissionOnSetting(context, this.permissions)
          .then((data: Array<AbilityAccessCtrl.GrantStatus>) => {
            hilog.info(0x0000, 'succeed', `requestPermissionOnSetting success, result: ${data}`);
          })
          .catch((err: BusinessError) => {
            hilog.error(0x0000, 'error',
              `requestPermissionOnSetting fail, code: ${err.code}, message: ${err.message}`);
          });
        break;
      }
    }
  }).catch((error: Error) => {
    hilog.error(0x0000, 'failed', `Request permissions failed, error is: ${error}`);
  });
  await this.checkPermissions();               // FIX: absent in the sample - the bubble never closes
}
```

**`dialogShownResults` is the part most implementations miss.** After a
permanent refusal `requestPermissionsFromUser` still resolves successfully and
shows nothing, so a naive implementation looks like it worked and the user
sees no dialog at all. The reference says it plainly: with `authResults` `-1`
and `dialogShownResults` `false`, "the permission has been set and no dialog
box is displayed" - that is the cue to call `requestPermissionOnSetting`,
which opens the settings sheet directly.

**Avoid areas into padding — `entry/src/main/ets/pages/TabIndex.ets`** (as shipped)

```typescript
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;

// ...
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

Compare with `TOUR-01`, which publishes the same two values through
`@Provide` / `@Consume` and gets the types wrong. `@StorageProp` with a typed
default of `0` is the simpler and safer form when the value is only read: the
ability writes px into `AppStorage`, the page converts with `px2vp` at the
point of use, and a missing key falls back to `0` instead of `undefined`.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  {
    "name": "ohos.permission.LOCATION",                    // PRECISE location
    "reason": "$string:Location_Reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",      // APPROXIMATE location
    "reason": "$string:Location_Reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory and the
  reason resource must exist in `resources/base/element/string.json`.
- `LOCATION` is refused unless `APPROXIMATELY_LOCATION` is requested in the
  same array.
- `when: "inuse"` is right for this scenario: the app has no continuous task
  and must not hold location in the background.
- The document's 权限说明 has the two descriptions swapped (`HW-09-0018`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `compatibleSdkVersion` in the sample is `6.0.0(20)`, so `bindPopup` options
  used here (`showInSubWindow`, `backgroundBlurStyle`, `onStateChange`) are
  all available; `requestPermissionOnSetting` needs API 12 or later.
- The sample never actually resolves a location: it demonstrates the prompt,
  not the fix. The location text is a static string resource. For the fix
  itself see `TOUR-01`.
- The three non-home tabs are placeholders holding a single `Text` - and are
  unreachable anyway until `HW-09-0016` is fixed.

## Pitfalls

- **`HW-09-0016` — `TabIndex` returns `false` from `onContentWillChange`,**
  which vetoes every tab switch. Three of the four bottom tabs are dead; only
  the bar highlight moves. Remove the callback or return `true`.
- **`HW-09-0017` — `checkPermissions` assigns `handlePopup` on every loop
  pass,** so the last permission wins. With the approximate permission granted
  and the precise one denied, the bubble never appears - the exact case the
  feature is for. Accumulate into a local and assign once.
- **`HW-09-0018` — the document's 权限说明 swaps the two permissions.**
  `LOCATION` is precise, `APPROXIMATELY_LOCATION` is approximate. The document
  says the opposite, and contradicts `TOUR-01` on the same pair.
- **`HW-09-0019` — nothing re-checks after the grant,** so the bubble stays up
  offering to enable a permission that is already enabled. The document
  promises 开启权限后气泡不再提示 (after enabling, the bubble no longer
  prompts); that only holds from the next launch. Call `checkPermissions()`
  again after the request resolves.
- **`HW-09-0020` — `tokenID!` and `grantStatus!` are asserted non-null on
  variables that stay unassigned** when their try blocks throw, so a failed
  `getBundleInfoForSelf` returns `undefined` typed as a `GrantStatus`.
  Initialise both and return early.
- **`HW-09-0021` — `avoidAreaChange` is never released and
  `setWindowLayoutFullScreen` drops its promise.** Same boilerplate defect as
  `TOUR-01`'s `HW-09-0013`; it recurs across this industry's samples.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-popup.md` - `bindPopup` and `PopupOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-popup
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `onContentWillChange` and its return value
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` - `authResults` and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the user_grant request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_application-services/map-location.md` - which of the two location permissions is which
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
- `TOUR-01` - the same avoid-area boilerplate, and the location fix this bubble unlocks
