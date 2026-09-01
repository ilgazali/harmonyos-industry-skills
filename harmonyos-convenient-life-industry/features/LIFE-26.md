---
id: LIFE-26
title: Find nearby service outlets and act on one - current location, reverse geocoding, keyword place search, dial and navigate
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
sample: huawei_industry_tree/02_convenient_life/downloads/ListOfNearbyOutlets.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.TelephonyKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["geoLocationManager.getCurrentLocation", "geoLocationManager.isGeocoderAvailable", "geoLocationManager.getAddressesFromLocation", "geoLocationManager.ReverseGeoCodeRequest", "geoLocationManager.GeoAddress", "site.searchByText", "site.SearchByTextParams", "site.SearchByTextResult", "site.Site", "site.Poi", "site.OpeningHours", "site.Period", "site.TimeOfWeek", "petalMaps.openMapRoutePlan", "petalMaps.RoutePlanParams", "mapCommon.LatLng", "call.hasVoiceCapability", "call.makeCall", "abilityAccessCtrl.createAtManager", "atManager.requestPermissionsFromUser", "atManager.requestPermissionOnSetting", "atManager.checkAccessToken", "bundleManager.getBundleInfoForSelf", "PermissionRequestResult.dialogShownResults", List, ListItem, ForEach, State, expandSafeArea, "resourceManager.getStringSync", "PromptAction.showToast"]
permissions: ["ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-02-0211, HW-02-0212, HW-02-0213, HW-02-0214, HW-02-0215, HW-02-0216, HW-02-0217, HW-02-0218, HW-02-0219, HW-02-0220, HW-02-0221, HW-02-0222, HW-02-0223, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **what is near me, and let me act on it**: nearby branches,
restaurants, ATMs, pharmacies - a list built from the device's own position,
each row offering a call and a route.

It is a four-API chain, and each link is worth taking separately:

```
geoLocationManager.getCurrentLocation()          -> a fix
geoLocationManager.getAddressesFromLocation()    -> a place name for the header
site.searchByText({ query, location, radius })   -> the POIs
   -> site.name / formatAddress / poi.phone / poi.openingHours / location
call.makeCall(phone)                             -> the system dialler
petalMaps.openMapRoutePlan(ctx, { destinationPosition }) -> the map app
```

The two hand-offs at the end are the reason this shape is worth copying: neither
dialling nor navigating needs a permission or a UI of your own. Only the
positioning half does.

**Two permissions, and they must be requested together.** `ohos.permission.LOCATION`
carries a documented prerequisite: "This permission must be requested with
`ohos.permission.APPROXIMATELY_LOCATION`". This sample declares and requests both,
which is the correct shape.

Take `LIFE-25` instead if both endpoints are already known and you only need the
distance and duration between them - that needs no permission at all.

## Feature checklist

- [ ] Both location permissions declared **and** requested together.
- [ ] `getCurrentLocation` inside a try, and its failure clearing the searching
      state (HW-02-0215).
- [ ] The reverse-geocode result checked for emptiness before indexing
      (HW-02-0219) and its catch actually logging (HW-02-0214).
- [ ] The paging loop bounded by `pageIndex * pageSize <= 500`
      (HW-02-0213).
- [ ] `ForEach` keyed on `site.siteId`, not on the row object
      (HW-02-0212).
- [ ] The list bound to the real fields, not to placeholder text
      (HW-02-0211).
- [ ] `openMapRoutePlan` wrapped in try/catch (HW-02-0218), and the
      no-voice-capability branch telling the user something (HW-02-0220).

## Architecture

Three files plus a model - there is no service layer, and one page holds the
whole flow.

| File | Role |
| --- | --- |
| `pages/NearbyOutletsListPage.ets` | `@Entry`. Location, reverse geocoding, place search, opening-hours formatting, the list, and both hand-offs. |
| `utils/PermissionsUtil.ets` | A module singleton exporting check / request / request-on-setting, keyed off `bundleManager.getBundleInfoForSelf`. |
| `dataModel/OutletInfo.ets` | The row model: `placeName`, `address`, `businessHour`, `phone`, `location`. |

Four `@State` fields drive the whole screen:

```ts
@State nearbyOutlets: OutletInfo[] = [];
@State placeName: ResourceStr = $r('app.string.fetching_location');
@State count: number = 0;
@State isSearchConcluded: boolean = false;
@State isBeginToSearch: boolean = false;
```

`isBeginToSearch` reveals the status row; `isSearchConcluded` swaps it from
"searching" to the result count. Both are set outside the render path -
`isBeginToSearch` in the click handler, `isSearchConcluded` in the `finally` of
the search - which is what makes the failure path in pitfall 5 leave the page
stuck.

The page is a single `@Entry` component with no `Navigation` and no
`routerMap`, and it handles the system bars declaratively rather than through
`AppStorage`:

```ts
.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM]);
```

That one line replaces the whole `setWindowLayoutFullScreen` plus
`getWindowAvoidArea` plus `on('avoidAreaChange')` dance that `LIFE-24` and
`LIFE-25` perform in their abilities - and it cannot leak a listener. Prefer it
whenever the page just needs to draw under the bars.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Declare both location permissions.**

   ```json5
   "requestPermissions": [
     { "name": "ohos.permission.LOCATION",
       "reason": "$string:EntryAbility_desc",
       "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
     { "name": "ohos.permission.APPROXIMATELY_LOCATION",
       "reason": "$string:EntryAbility_desc",
       "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
   ]
   ```

   `LOCATION` alone is not grantable - the permission list states the
   prerequisite explicitly.

2. **Request, then re-check, then act.** The utility runs the full three-step
   flow; the page verifies the outcome independently before doing any work:

   ```ts
   let permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
   await PermissionsRequest.commonRequestPermissions(permissions, this.getUIContext());
   let permissionAllowed = await PermissionsRequest.checkPermissions(permissions);
   if (permissionAllowed) {
     this.isBeginToSearch = true;
     this.nearbyOutlets = [];
     this.count = 0;
     let location = await this.getLocation();
     this.searchNearbyOutlets(location);
   }
   ```

   Inside `commonRequestPermissions` the decision of whether to fall back to the
   Settings page is made from `dialogShownResults`, not from the grant result:

   ```ts
   let checkOk: boolean = await this.checkPermissions(permissions);
   if (!checkOk) {
     let isDialogShown = await this.requestPermissions(context, permissions);
     if (isDialogShown !== true) {
       await this.requestPermissionsOnSetting(context, permissions);   // second grant
     }
   }
   ```

   No dialog shown means the system will not ask again, which is precisely when
   `requestPermissionOnSetting` is the right call.

3. **Get the fix, inside a try** (HW-02-0215). The shipped code awaits it on
   the line *before* the try opens:

   ```ts
   async getLocation(): Promise<mapCommon.LatLng> {
     try {
       let location = await geoLocationManager.getCurrentLocation();   // shipped: outside the try
       if (geoLocationManager.isGeocoderAvailable()) {
         // step 4
       }
       return { latitude: location.latitude, longitude: location.longitude } as mapCommon.LatLng;
     } catch (error) {
       let err = error as BusinessError;
       hilog.error(0x0000, 'NearbyOutlets', `getLocation failed. Code: ${err.code}`);
       throw error;                                                     // let the caller clear isBeginToSearch
     }
   }
   ```

4. **Reverse geocode for a display name** (HW-02-0219, HW-02-0214):

   ```ts
   let reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest =
     { 'latitude': location.latitude, 'longitude': location.longitude, 'maxItems': 1 };
   geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest, (err, data) => {
     if (err) {
       hilog.error(0x0000, 'NearbyOutlets', 'getAddressesFromLocation err: ' + JSON.stringify(err));
       return;
     }
     let geoAddress = data as geoLocationManager.GeoAddress[];
     if (!geoAddress || !geoAddress.length) {     // shipped code omits this
       return;
     }
     this.placeName = this.getLocationName(geoAddress[0].placeName ?? '', geoAddress[0].subLocality ?? '');
   });
   ```

   `maxItems: 1` is an upper bound, not a guarantee. `subLocality` is the
   district and `premises` the road number - the sample comments on the choice
   at `NearbyOutletsListPage.ets:51`, and the document's snippet quietly picks
   the other one (HW-02-0223).

5. **Trim the place name safely** (HW-02-0216). The shipped helper splits on a
   separator that may be empty:

   ```ts
   getLocationName(address: string, subLocality: string): string {
     if (!subLocality) {
       return address;                          // shipped code splits on '' instead
     }
     let placeString: string[] = address.split(subLocality);
     return placeString[placeString.length - 1];
   }
   ```

6. **Search by keyword around the fix.**

   ```ts
   const DISTANCE = 5000;
   let params: site.SearchByTextParams = {
     query: '华为授权',
     location: { latitude: location.latitude, longitude: location.longitude },
     radius: DISTANCE,          // 1 to 50000 metres; 50000 by default
     language: 'zh_CN',
     pageIndex: 1
   };
   let result = await site.searchByText(this.getUIContext().getHostContext(), params);
   ```

   `Site.distance` is the straight-line distance in metres and is only populated
   for keyword and nearby search - which is why the sample can filter on it:

   ```ts
   let distance = site.distance ?? -1;
   if (distance <= DISTANCE && distance >= 0) { /* keep */ }
   ```

   Note the `?? -1` drops sites with no distance rather than keeping them.

7. **Page through the rest, within the documented limit** (HW-02-0213):

   ```ts
   const PAGE_SIZE = 20;
   const MAX_PAGE = Math.floor(500 / PAGE_SIZE);          // pageIndex * pageSize <= 500
   for (let i = 2; i <= Math.min(Math.ceil(count / PAGE_SIZE), MAX_PAGE); i++) {
     // shipped: i <= Math.ceil(count / 20.0), unbounded
   }
   ```

   Set `pageSize` explicitly rather than relying on the default, so the two
   numbers in the limit calculation are the two numbers you sent.

8. **Format the opening hours from `Period` pairs** (HW-02-0217). `time` is a
   string in `hhmm`, `week` is 0 for Sunday through 6 for Saturday:

   ```ts
   let time: string = periods[i-1].open?.time?.toString().substring(0, 2) + ':' +
     periods[i-1].open?.time?.toString().substring(2) + '—' + /* ... close ... */;
   str = getStringSync($r('app.string.week').id) +
     getStringSync(this.getDayInChinese(periods[i]?.open?.week ?? -1).id);  // shipped: getDayInChinese(i)
   ```

   Read the weekday from `open.week` on both ends of a range - the shipped code
   reads it from the loop index on one of them.

9. **Build the rows from the real fields** (HW-02-0211, HW-02-0212):

   ```ts
   List() {
     ForEach(this.nearbyOutlets, (item: OutletInfo) => {
       ListItem() {
         Text(item.placeName);        // shipped: Text('xxxxxxxx')
         Text(item.address);          // shipped: Text('xxxxxxxxxxxxxxxx')
         Text(item.phone);            // shipped: Text('xxxxxxxx')
       }
     }, (item: OutletInfo) => item.siteId);   // shipped: (item: string) => item
   }
   ```

   `Site.siteId` is the only mandatory, unique field the search returns - carry
   it into `OutletInfo` and key on it.

10. **Dial through the system, with a branch for devices that cannot**
    (HW-02-0220):

    ```ts
    jumpToCallPage(phoneNumber: string) {
      if (!call.hasVoiceCapability()) {
        this.promptAction.showToast({ message: $r('app.string.no_voice_capability') });
        return;                                       // shipped: silently does nothing
      }
      call.makeCall(phoneNumber, (err: BusinessError) => {
        if (err) { hilog.info(0x0000, '', 'make call fail, err is:' + JSON.stringify(err)); }
      });
    }
    ```

    `makeCall` opens the dialler with the number filled in; it does not place
    the call, which is why no permission is involved. From API 15 the number may
    be given in `tel:` form.

11. **Hand off to the map application, guarded** (HW-02-0218):

    ```ts
    async jumpToLocationMap(location: mapCommon.LatLng) {
      let params: petalMaps.RoutePlanParams = { destinationPosition: location };
      try {
        await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params);
      } catch (error) {
        let err = error as BusinessError;
        hilog.error(0x0000, 'NearbyOutlets', `openMapRoutePlan failed. Code: ${err.code}`);
      }
    }
    ```

    Only `destinationPosition` is set - the map application uses the device's
    own position as the origin, so no second lookup is needed.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`ListOfNearbyOutlets.zip#entry/src/main/ets/pages/NearbyOutletsListPage.ets:38-62` -
the location and reverse-geocoding step as shipped:

```ts
  async getLocation(): Promise<mapCommon.LatLng> {
    let location = await geoLocationManager.getCurrentLocation();
    try {
      let isAvailable = geoLocationManager.isGeocoderAvailable();
      if (isAvailable) {
        let reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest =
          { 'latitude': location.latitude, 'longitude': location.longitude, 'maxItems': 1 };
        geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest, (err, data) => {
          if (err) {
            hilog.error(0x0000, '', 'getAddressesFromLocation err: ' + JSON.stringify(err));
          } else {
            hilog.info(0x0000, '', JSON.stringify(data));
            let geoAddress = data as geoLocationManager.GeoAddress[];
            // subLocality为区信息，premises为路序号信息，若设置为premises可显示到更详细的地点名
            this.placeName = this.getLocationName(geoAddress[0].placeName ?? '', geoAddress[0].subLocality ?? '');
          }
        });
      }
    } catch (err) {
    }
    return {
      latitude: location.latitude,
      longitude: location.longitude
    } as mapCommon.LatLng;
  }
```

Three findings live in those 25 lines: the await outside the try
(HW-02-0215), the unchecked `geoAddress[0]` (HW-02-0219), and the empty
catch (HW-02-0214).

`ListOfNearbyOutlets.zip#entry/src/main/ets/pages/NearbyOutletsListPage.ets:64-93` -
the keyword search and the distance filter:

```ts
  async searchNearbyOutlets(location: mapCommon.LatLng) {
    const DISTANCE = 5000;
    let params: site.SearchByTextParams = {
      query: '华为授权',
      location: {
        latitude: location.latitude,
        longitude: location.longitude
      },
      radius: DISTANCE,
      language: 'zh_CN',
      pageIndex: 1
    };
    try {
      let result = await site.searchByText(this.getUIContext().getHostContext(), params);
      hilog.info(0x0000, '', 'Succeeded in searching by text.' + JSON.stringify(result));
      let count = result.totalCount;
      let sites: site.Site[] = result.sites ?? [];
      sites.forEach(site => {
        let distance = site.distance ?? -1;
        if (distance <= DISTANCE && distance >= 0) {
          let outletInfo = {
            placeName: site.name,
            address: site.formatAddress,
            businessHour: this.getOpeningHours(site.poi?.openingHours?.periods),
            phone: site.poi?.phone,
            location: site.location
          } as OutletInfo;
          this.nearbyOutlets.push(outletInfo);
        }
      });
```

`ListOfNearbyOutlets.zip#entry/src/main/ets/pages/NearbyOutletsListPage.ets:121-128` -
the search's error and completion handling, which is the correct shape:

```ts
    } catch (error) {
      const ERR: BusinessError = error as BusinessError;
      hilog.error(0x0000, '', `Failed in searching nearby. Code is ${ERR.code}, message is ${ERR.message}`);
    } finally {
      this.isSearchConcluded = true;
      this.count = this.nearbyOutlets.length;
    }
```

`ListOfNearbyOutlets.zip#entry/src/main/ets/pages/NearbyOutletsListPage.ets:191-212` -
both hand-offs, neither of which needs a permission:

```ts
  jumpToCallPage(phoneNumber: string) {
    // 调用查询能力接口
    let isSupport = call.hasVoiceCapability();
    if (isSupport) {
      // 如果设备支持呼叫能力，则继续跳转到拨号界面，并显示拨号的号码
      // 从API15开始支持tel格式电话号码，如："tel:13xxxx"
      call.makeCall(phoneNumber, (err: BusinessError) => {
        if (!err) {
          hilog.info(0x0000, '', 'make call success.');
        } else {
          hilog.info(0x0000, '', 'make call fail, err is:' + JSON.stringify(err));
        }
      });
    }
  }

  async jumpToLocationMap(location: mapCommon.LatLng) {
    let params: petalMaps.RoutePlanParams = {
      destinationPosition: location
    };
    await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params);
  }
```

`ListOfNearbyOutlets.zip#entry/src/main/ets/utils/PermissionsUtil.ets:22-31` -
the three-step permission flow, decided on `dialogShownResults`:

```ts
  async commonRequestPermissions(permissions: Array<Permissions>, context: UIContext): Promise<void> {
    let checkOk: boolean = await this.checkPermissions(permissions);
    if (!checkOk) {
      let isDialogShown = await this.requestPermissions(context, permissions);
      if (isDialogShown !== true) {
        // 二次授权
        await this.requestPermissionsOnSetting(context, permissions);
      }
    }
  }
```

`ListOfNearbyOutlets.zip#entry/src/main/ets/utils/PermissionsUtil.ets:36-47` -
reading the dialog flag out of the request result:

```ts
  async requestPermissions(context: UIContext, permissions: Array<Permissions>): Promise<boolean | undefined> {
    let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
    try {
      let data =
        await atManager.requestPermissionsFromUser(context.getHostContext() as common.UIAbilityContext, permissions);
      hilog.info(0x000, 'testTag', 'requestPermissions1 success', JSON.stringify(data));
      return data.dialogShownResults ? data.dialogShownResults[0] : undefined; // 返回请求是否有弹窗
    } catch (e) {
      hilog.error(0x000, 'testTag', `requestPermissions1 err Code is ${e.code}, message is ${e.message}`);
      return undefined;
    }
  }
```

`ListOfNearbyOutlets.zip#entry/src/main/ets/pages/NearbyOutletsListPage.ets:379-382` -
the declarative alternative to the whole avoid-area dance:

```ts
    .width('100%')
    .height('100%')
    .backgroundColor('#F1F3F5')
    .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM]);
```

## Permissions & config

Two user-grant permissions, requested together:

```json5
// entry/src/main/module.json5
"requestPermissions": [
  { "name": "ohos.permission.LOCATION", "reason": "$string:EntryAbility_desc",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION", "reason": "$string:EntryAbility_desc",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
]
```

From the permission list: `ohos.permission.LOCATION` - "**Prerequisites**: This
permission must be requested with `ohos.permission.APPROXIMATELY_LOCATION`."
Both are permission level normal, authorization mode user_grant.

Neither `call.makeCall` nor `petalMaps.openMapRoutePlan` needs a permission -
both launch a system application with the data prefilled.

Account-side setup, quoted from the document: Map Service must be switched on
for the application under API management in AppGallery Connect, and the build
manually signed. `site.searchByText` and `openMapRoutePlan` both depend on it.

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **Map Service enabled in AGC plus manual signing** - the place search and the
  map hand-off both need it.
- **`ohos.permission.LOCATION` requires `ohos.permission.APPROXIMATELY_LOCATION`.**
- **Keyword search paging:** `pageSize` is 1 to 20 (default 20), `pageIndex` is
  1 to 500, and the two are bound together by `pageIndex * pageSize <= 500`.
- **`radius`** is 1 to 50000 metres, default 50000; fractions are ignored and an
  out-of-range value returns error code 401.
- **`query`** is 1 to 512 characters and is mandatory.
- **`Site.distance`** is optional, in metres, and populated only for keyword
  search and nearby search.
- **`Site.siteId`** is the only mandatory unique identifier the search returns -
  it is what a `ForEach` key should be built from.
- **`TimeOfWeek.time` is a string** in `hhmm` form, and `week` is 0 for Sunday
  through 6 for Saturday.
- **`getAddressesFromLocation` can resolve zero addresses** even on success;
  `maxItems` is an upper bound.
- **`call.hasVoiceCapability()` can be false** on the tablet and 2in1 device
  types this module declares.

## Pitfalls

1. **HW-02-0211 - the shipped sample shows no real data.** Four bindings are
   commented out and replaced with literals: `:229-230` `//Text(this.placeName)`
   then `Text('xxxxxxxx')`, `:290-291` the same for `item.placeName`, `:297-298`
   for `item.address` with sixteen x characters, and `:329-330` for
   `item.phone`. All four values are computed and stored (`:52`, `:85-88`); only
   `businessHour` reaches the screen (`:313`). Build and run the sample and
   every row shows placeholder text, while tapping the phone row still dials a
   number that was never displayed. Uncomment the four bindings and delete the
   literals.

2. **HW-02-0212 - the list key generator is typed as a string and handed an
   object.** `:285` opens `ForEach(this.nearbyOutlets, (item: OutletInfo) => {`
   and `:369` closes it with `}, (item: string) => item);`. The annotation is
   false at runtime, which is what stops the compiler from catching it, and the
   value returned is an `OutletInfo`, not the unique string key generation
   requires. The guide warns that "When the keys generated for different array
   items are the same, the behavior of the framework is undefined" and
   recommends "use a unique id property from the object data as the key". Carry
   `site.siteId` into the model and key on it.

3. **HW-02-0213 - the paging loop can exceed the documented page limit.**
   `:94` is `for (let i = 2; i <= Math.ceil(count / 20.0); i++)` where `count` is
   `result.totalCount`, and `pageSize` is never set so it defaults to 20. The
   reference caps the pair: "`pageIndex * pageSize <= 500`", which with the
   default page size means `pageIndex <= 25`. A dense area with more than 500
   matches drives the loop past that, one awaited request at a time, in the
   user-visible search path.

4. **HW-02-0214 - an entirely empty catch.** `:56-57` is
   `} catch (err) { }`, covering `isGeocoderAvailable`, the request
   construction and the dispatch. Everything else in the same method logs. The
   method still returns coordinates as though nothing happened.

5. **HW-02-0215 - `getCurrentLocation` is awaited outside the try, and the
   caller does not catch either.** `:39` is the statement before `try {` opens,
   and the click handler at `:253` awaits `getLocation()` bare after setting
   `isBeginToSearch = true` at `:250`. `isSearchConcluded` is only set in the
   `finally` of the search (`:125`), which the rejection never reaches - so a
   failed fix leaves the page showing `网点搜索中...` ("searching for
   outlets...") permanently.

6. **HW-02-0216 - the place name is split on a separator that can be empty.**
   `:52` passes `geoAddress[0].subLocality ?? ''` into `getLocationName`, and
   `:186` does `address.split(subLocality)` then takes the last element.
   Splitting on `''` yields one element per character, so the missing-district
   case - the ordinary one the `??` exists for - produces a single-character
   place name instead of falling back to the full one.

7. **HW-02-0217 - the opening-hours label reads the weekday from the loop
   index.** `:171` correctly uses `periods[i]?.open?.week ?? -1`, but `:178`
   uses `this.getDayInChinese(i)` - the array index - for the start of the next
   range. The two agree only when the array is a sorted, complete, one-entry-per-day
   week, which stops being true as soon as a venue is closed one day or opens
   twice on another.

8. **HW-02-0218 - `openMapRoutePlan` is awaited with nothing catching it.**
   `:211` is the last statement of `jumpToLocationMap`, and `:358-360` calls
   that method from an `onClick` without awaiting it. The two network calls in
   the same file are both guarded. Tapping 到这里 ("go here") on a device with
   no map application does nothing at all - no toast, no log.

9. **HW-02-0219 - `geoAddress[0]` is read without checking the array.**
   `:50-52`. The optional *fields* inside the element are handled with `??`; the
   emptiness of the array is not. A successful call that resolves no address
   returns an empty array, and the read throws inside a callback where the
   surrounding try has already returned.

10. **HW-02-0220 - the no-voice-capability branch is silent.** `:191-205`
    checks `call.hasVoiceCapability()` and simply does nothing when it is false.
    The module declares `phone`, `tablet` and `2in1`; on the latter two the
    phone row is a dead tap with no explanation.

11. **HW-02-0221 - a hilog call with the message in the tag slot.**
    `PermissionsUtil.ets:84` is
    ``hilog.error(0x0000, `Failed to check access token  ${err.code}`, ` message is ${err.message}`)``.
    The signature is `error(domain, tag, format, ...args)`, and the reference
    notes a tag "can contain a maximum of 31 bytes. If a tag exceeds this limit,
    it will be truncated." Every permission-check failure therefore arrives
    under a different, truncated tag. The same file gets it right at `:41`.

12. **HW-02-0222 - the status vocabulary is hardcoded Chinese.** `:266` builds
    the result count by concatenation, `:272` is the searching message, `:313`
    the business-hours prefix and `:350` the navigate label - while the same
    build method uses `$r('app.string.outlet')` and
    `$r('app.string.to_be_implemented')` and the file reaches into
    `resourceManager` six times inside `getOpeningHours`.

13. **HW-02-0223 - the document's first snippet does not compile.** Its `try`
    at `26_list_of_nearby_outlets.md:29` has no `catch`; `isAvailable` is tested
    at `:30` without being declared (the ZIP declares it at
    `NearbyOutletsListPage.ets:41`); and `:37` passes
    `geoAddress[0].premises ?? ''` where the ZIP passes `subLocality`, under a
    comment explaining the difference that appears only in the ZIP. Read the
    ZIP.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- site (keyword search - `SearchByTextParams`, paging limits, `Site`, `Poi`,
  `Period`, `TimeOfWeek`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-site
- @ohos.geoLocationManager (`getCurrentLocation`, `getAddressesFromLocation`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-geolocationmanager
- petalMaps (`openMapRoutePlan`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/map-petal-maps
- @ohos.telephony.call (`makeCall`, `hasVoiceCapability`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-call
- Requesting user authorization:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/request-user-authorization
- Permissions for all applications (`LOCATION` prerequisite):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all-user
- ForEach (key generation rules and uniqueness):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- Map Kit setup in AppGallery Connect:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-config-agc
- hilog (parameter order, tag length limit):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
