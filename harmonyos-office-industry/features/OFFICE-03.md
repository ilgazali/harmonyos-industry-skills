---
id: OFFICE-03
title: Attendance check-in location - two-stage location permission request plus reverse geocoding
industry: 05_office
doc: huawei_industry_tree/05_office/docs/03_location_permissions.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
sample: huawei_industry_tree/05_office/downloads/LocationPermissions.zip
kits: ["@kit.AbilityKit", "@kit.LocationKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["abilityAccessCtrl.createAtManager", "AtManager.checkAccessToken", "AtManager.requestPermissionsFromUser", "AtManager.requestPermissionOnSetting", "abilityAccessCtrl.GrantStatus", PermissionRequestResult, "bundleManager.getBundleInfoForSelf", "bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION", "geoLocationManager.isLocationEnabled", "geoLocationManager.getCurrentLocation", "geoLocationManager.getAddressesFromLocation", "geoLocationManager.ReverseGeoCodeRequest", "windowStage.getMainWindowSync", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "AppStorage.setOrCreate", "@StorageProp", "UIContext.px2vp", "UIContext.getPromptAction"]
permissions: ["ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-05-0013, HW-05-0014, HW-05-0015, HW-05-0016, HW-05-0017, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app needs the user's **current physical location
at the moment of an action** - attendance check-in and check-out being the
canonical case - and therefore needs a location-permission flow that survives a
denial.

Two things make this scenario different from a plain "get my coordinates" task:

- The permission has **two stages**. The first tap raises the system
  authorization dialog. Once the user has answered it, the system no longer
  shows that dialog, so a later attempt must open the **permission settings
  dialog** (`requestPermissionOnSetting`) instead.
- The record must store a **human-readable address**, not coordinates, so the
  latitude/longitude has to be reverse-geocoded before it is written to the
  attendance row.

## Feature checklist

The application must be able to:

- Check the system location switch before doing anything else, and tell the user
  when it is off.
- Check whether the location permissions are already granted, and skip the
  request entirely when they are.
- Raise the first-time authorization dialog for
  `ohos.permission.LOCATION` **and** `ohos.permission.APPROXIMATELY_LOCATION`
  together.
- Detect that no dialog was shown (the permission had already been answered) and
  open the permission settings dialog instead.
- Re-verify the grant state after the request and only run the business logic
  when it succeeded.
- Obtain the current coordinates and reverse-geocode them into a place name.
- Record check-in and check-out separately, each with its own timestamp and its
  own address.
- Lay out under the status bar and navigation indicator by reading the avoid-area
  heights.

## Architecture

Single `entry` HAP, four source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/CheckInPage`, switches the window to full-screen layout, reads the status-bar and navigation-indicator avoid areas into `AppStorage` (`topRectHeight`, `bottomRectHeight`) and subscribes to `avoidAreaChange` to keep them current |
| `util/PermissionsRequest.ets` | the whole permission state machine, exported as a singleton (`export default new PermissionsRequest()`) |
| `pages/CheckInPage.ets` | the attendance UI, the location calls and the check-in/check-out records |
| `constants/StyleConstants.ets` | two layout constants |

`PermissionsRequest` exposes four methods that form a deliberate ladder:

1. `checkAccessToken(permission)` - resolves the app's `accessTokenId` through
   `bundleManager.getBundleInfoForSelf` and asks `atManager.checkAccessToken` for
   one permission.
2. `checkPermissions(permissions)` - returns `true` only when every permission in
   the list is `PERMISSION_GRANTED`.
3. `requestPermissions(context, permissions)` - first-stage dialog; returns
   `dialogShownResults[0]`, i.e. **whether a dialog was displayed**, not whether
   the user agreed.
4. `requestPermissionsOnSetting(context, permissions)` - second-stage settings
   dialog.

`commonRequestPermissions` composes them: check, then stage one, then stage two
only when stage one showed no dialog.

Data flow on a tap:

```
Button.onClick
  -> geoLocationManager.isLocationEnabled()        (switch off -> toast, stop)
  -> PermissionsRequest.commonRequestPermissions() (check -> stage 1 -> stage 2)
  -> PermissionsRequest.checkPermissions()         (re-verify, gate the business logic)
  -> getLocation()
       -> geoLocationManager.getCurrentLocation(cb)
            -> convertLatToPosition(lon, lat)
                 -> geoLocationManager.getAddressesFromLocation(req)
                      -> this.address = data[0].placeName
  -> record firstDate (check-in) or lastDate + lastAddress (check-out)
```

The last two steps run in the wrong order in the shipped sample - see
HW-05-0014.

## Implementation steps

1. **Declare both location permissions** in `entry/src/main/module.json5`.
   `ohos.permission.LOCATION` alone is not enough: the official Map Kit guide
   applies for `LOCATION` and `APPROXIMATELY_LOCATION` together, and requesting
   `LOCATION` on its own is one of the documented causes of `authResults == 2`
   ("the conditions for requesting the permission are not met").
2. **Gate on the system location switch.** Call
   `geoLocationManager.isLocationEnabled()` first; when it returns `false` no
   permission flow will help, so show a toast and stop.
3. **Check before you ask.** Resolve the app's `accessTokenId` from
   `bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION)`
   and call `atManager.checkAccessToken(tokenId, permission)` for each
   permission. Skip the dialog entirely when all are already granted.
4. **Stage one - the authorization dialog.** Call
   `atManager.requestPermissionsFromUser(context.getHostContext() as common.UIAbilityContext, permissions)`
   and read `data.dialogShownResults[0]`.
5. **Stage two - the settings dialog.** When `dialogShownResults[0]` is not
   `true`, the system showed no dialog because the permission has already been
   answered; only then call
   `atManager.requestPermissionOnSetting(context.getHostContext() as common.UIAbilityContext, permissions)`.
   Cast the context - the shipped sample and the document both omit the cast
   (HW-05-0013). Note that the document describes this branch as "the user
   refused", which is a different and wrong condition (HW-05-0016).
6. **Re-verify.** `requestPermissionOnSetting` returns
   `Promise<Array<GrantStatus>>`, but the sample instead calls
   `checkPermissions` again after the whole ladder and gates the business logic
   on that boolean. Either is fine; do not skip the re-check.
7. **Get the coordinates.** `geoLocationManager.getCurrentLocation(callback)`;
   the callback receives `(err, location)` and must test both. Wrap the call in
   `try/catch` and **log** the caught error - the document's snippet discards it
   (HW-05-0017).
8. **Reverse-geocode.** Build a
   `geoLocationManager.ReverseGeoCodeRequest` with `latitude`, `longitude` and
   `maxItems: 1`, pass it to `getAddressesFromLocation`, and take
   `data[0].placeName`.
9. **Write the attendance row only after the address resolves.** Assign
   `lastAddress` inside the `getAddressesFromLocation` resolution handler, not on
   the statement after `getLocation()` (HW-05-0014).
10. **Handle the insets.** In `onWindowStageCreate`, call
    `setWindowLayoutFullScreen(true)`, read both avoid areas, publish them into
    `AppStorage`, and subscribe to `avoidAreaChange`. Unsubscribe with
    `off('avoidAreaChange')` in `onWindowStageDestroy` - the sample does not
    (HW-05-0015). The page reads them with `@StorageProp` and converts with
    `this.getUIContext().px2vp(...)`.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The permission ladder

`LocationPermissions.zip#entry/src/main/ets/util/PermissionsRequest.ets`

```ts
import { abilityAccessCtrl, bundleManager, common, Permissions } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

class PermissionsRequest {
  async commonRequestPermissions(context: UIContext, permissions: Array<Permissions>): Promise<void> {
    let isPermission: boolean = await this.checkPermissions(permissions);
    if (!isPermission) {
      // stage 1
      let isDialogShown = await this.requestPermissions(context, permissions);
      if (isDialogShown !== true) {
        // stage 2 - no dialog was shown, so the permission has already been answered
        await this.requestPermissionsOnSetting(context, permissions);
      }
    }
  }

  async checkPermissions(permissions: Array<Permissions>): Promise<boolean> {
    for (let permission of permissions) {
      let grantStatus: abilityAccessCtrl.GrantStatus = await this.checkAccessToken(permission);
      if (grantStatus !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
        return false;
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

  async requestPermissions(context: UIContext, permissions: Array<Permissions>): Promise<boolean | undefined> {
    let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
    try {
      let data =
        await atManager.requestPermissionsFromUser(context.getHostContext() as common.UIAbilityContext, permissions);
      return data.dialogShownResults ? data.dialogShownResults[0] : undefined; // whether a dialog was shown
    } catch (e) {
      hilog.error(0x000, 'testTag', `requestPermissions1 err Code is ${e.code}, message is ${e.message}`);
      return undefined;
    }
  }

  async requestPermissionsOnSetting(context: UIContext, permissions: Array<Permissions>): Promise<void> {
    let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
    // as shipped - the context is not cast; see HW-05-0013
    await atManager.requestPermissionOnSetting(context.getHostContext(), permissions);
  }
}

export default new PermissionsRequest();
```

Corrected stage-two call:

```ts
await atManager.requestPermissionOnSetting(context.getHostContext() as common.UIAbilityContext, permissions);
```

### Location switch, coordinates and reverse geocoding

`LocationPermissions.zip#entry/src/main/ets/pages/CheckInPage.ets`

```ts
import { geoLocationManager } from '@kit.LocationKit';

isLocationEnabled(): boolean {
  let isLocationEnabled = geoLocationManager.isLocationEnabled();
  return isLocationEnabled;
}

getLocation() {
  let locationChange = (err: BusinessError, location: geoLocationManager.Location): void => {
    if (err) {
      hilog.error(0x000, 'testTag', 'locationChanger: err=' + JSON.stringify(err));
    }
    if (location) {
      let longitude = location.longitude;
      let latitude = location.latitude;
      this.convertLatToPosition(longitude, latitude);
    }
  };
  try {
    geoLocationManager.getCurrentLocation(locationChange);
  } catch (err) {
    hilog.error(0x000, 'testTag', 'errCode:' + JSON.stringify(err));
  }
}

convertLatToPosition(longitude: number, latitude: number) {
  let reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest = {
    'latitude': latitude, 'longitude': longitude, 'maxItems': 1
  };
  try {
    geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest).then((data) => {
      this.address = data[0].placeName ? data[0].placeName : '';
    })
      .catch((error: BusinessError) => {
        hilog.error(0x000, 'testTag', 'promise, getAddressesFromLocation: error=' + JSON.stringify(error));
      });
  } catch (err) {
    hilog.error(0x000, 'testTag', 'errCode:' + JSON.stringify(err));
  }
}
```

### The check-in / check-out tap handler

`LocationPermissions.zip#entry/src/main/ets/pages/CheckInPage.ets`

```ts
.onClick(async () => {
  if (this.isLocationEnabled()) {
    let permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
    await PermissionsRequest.commonRequestPermissions(this.getUIContext(), permissions);
    let permissionAllowed = await PermissionsRequest.checkPermissions(permissions);
    if (permissionAllowed) {
      this.getLocation();
      if (!this.firstDate) {
        this.firstDate = new Date();
      } else {
        this.lastDate = new Date();
        this.lastAddress = this.address; // stale - see HW-05-0014
      }
      this.getUIContext().getPromptAction().showToast({
        message: '打卡成功',
        alignment: Alignment.Bottom,
        textColor: '#64BB5C',
        offset: { dx: 0, dy: -300 }
      });
    }
  } else {
    this.getUIContext().getPromptAction().showToast({
      message: '位置开关未打开',
      alignment: Alignment.Bottom,
      offset: { dx: 0, dy: -300 }
    });
  }
})
```

### Full-screen layout and avoid areas

`LocationPermissions.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.loadContent('pages/CheckInPage', (err) => {
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
  });
  let windowClass: window.Window = windowStage.getMainWindowSync();
  let isLayoutFullScreen = true;
  windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(() => {
    hilog.info(DOMAIN, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag',
      'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
  });

  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });
  // missing: windowClass.off('avoidAreaChange') in onWindowStageDestroy - see HW-05-0015
}
```

Page side:

```ts
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;

// ...
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight) + StyleConstants.INDEX_TOP_MARGIN,
  bottom: this.getUIContext().px2vp(this.bottomRectHeight) + StyleConstants.INDEX_BOTTOM_MARGIN
});
```

## Permissions & config

`LocationPermissions.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "requestPermissions": [
      {
        "name": "ohos.permission.LOCATION",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      },
      {
        "name": "ohos.permission.APPROXIMATELY_LOCATION",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      }
    ]
  }
}
```

The document's 权限说明 section lists exactly these two permissions, and the
sample declares exactly these two - verified consistent.

Both are user_grant permissions with `"when": "inuse"`, so they cover foreground
use only; background positioning would additionally need
`ohos.permission.LOCATION_IN_BACKGROUND`, which this scenario does not use.

`LocationPermissions.zip#build-profile.json5` pins
`"compatibleSdkVersion": "6.0.0(20)"` and enables
`strictMode: { caseSensitiveCheck: true, useNormalizedOHMUrl: true }`.

## Constraints

- **Versions.** The document states API Version 20 Release or later, HarmonyOS
  6.0.0 Release SDK or later, and DevEco Studio 6.0.0 Release or later; the
  sample's `compatibleSdkVersion` is `6.0.0(20)`, which matches.
- **Devices.** `deviceTypes` is `["phone", "tablet", "2in1"]`.
- **System location switch.** `isLocationEnabled()` returning `false` cannot be
  fixed by any permission request; the app must send the user to the system
  toggle.
- **Second-stage dialog precondition.** Per the reference,
  `requestPermissionOnSetting` may only be called after `requestPermissionsFromUser`
  has run, and it displays nothing if the user already granted the permission in
  the first dialog.
- **Location permission pair.** `ohos.permission.LOCATION` must be requested
  together with `ohos.permission.APPROXIMATELY_LOCATION`; requesting the precise
  permission alone is one of the documented causes of an invalid request
  (`authResults == 2`).
- **Reverse geocoding is a network operation.** `getAddressesFromLocation`
  resolves asynchronously and can reject; `placeName` may be absent, which is why
  the sample falls back to an empty string.
- **Map imagery is static.** The page draws `app.media.map`; there is no Map Kit
  component in this sample, so nothing here proves a map-rendering flow.

## Pitfalls

- **The document says the settings dialog opens "if the user refuses
  authorization", which is incorrect.** It opens when `dialogShownResults[0]` is
  `false`, i.e. when the system showed no dialog at all because the permission
  had already been answered. Denying the first dialog leaves
  `dialogShownResults[0] === true` and the sample deliberately does not escalate
  then. (HW-05-0016)
- **`requestPermissionOnSetting(context.getHostContext(), ...)` is incorrect.**
  `getHostContext()` is typed `Context | undefined` while the parameter is a
  mandatory `Context`; cast it as the same file already does for
  `requestPermissionsFromUser` and as the official guide does. (HW-05-0013)
- **`this.lastAddress = this.address` immediately after `getLocation()` is
  incorrect.** The address is filled in by an asynchronous reverse-geocoding
  callback, so the check-out row stores the check-in address, or an empty string
  on the first tap; assign it inside the `getAddressesFromLocation` handler.
  (HW-05-0014)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect.**
  Call `windowClass.off('avoidAreaChange', callback)` in `onWindowStageDestroy`.
  (HW-05-0015)
- **The document's `catch (err) {}` is incorrect.** `getCurrentLocation` throws
  synchronously for an invalid request or a disabled location service; log the
  error as the sample does, otherwise "no permission" and "location off" are
  indistinguishable at runtime. (HW-05-0017)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionOnSetting12+` signature, mandatory `Context` parameter and
  the precondition that `requestPermissionsFromUser` must have run first.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  the meaning of `authResults` and `dialogShownResults`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `getHostContext(): Context | undefined`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on('avoidAreaChange')` / `off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  first-stage authorization flow and `authResults` checking.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_system/request-user-authorization-second.md` -
  second-stage `requestPermissionOnSetting` flow with the context cast.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization-second
- `documentation/harmonyos-guides/04_application-services/map-location.md` -
  requesting `LOCATION` together with `APPROXIMATELY_LOCATION` and the
  `module.json5` fragment for both.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
