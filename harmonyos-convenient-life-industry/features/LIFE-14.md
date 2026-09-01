---
id: LIFE-14
title: Map with a docked half-modal - nested bindSheets that never dismiss, over a map resized to the detent
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/14_map_bind.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
sample: huawei_industry_tree/02_convenient_life/downloads/MapBindSheet.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: [bindSheet, SheetSize, SheetType, detents, enableOutsideInteractive, showClose, onWillDismiss, onDetentsDidChange, DismissSheetAction, MapComponent, "mapCommon.MapOptions", "mapCommon.MyLocationStyle", "MyLocationDisplayType", "map.MapComponentController", "controller.setMyLocationEnabled", "controller.setMyLocationStyle", "controller.setMyLocation", "controller.show", "controller.hide", "controller.getEventManager", "MapEventManager.on", "geoLocationManager.getCurrentLocation", "abilityAccessCtrl.createAtManager", "atManager.checkAccessToken", "atManager.requestPermissionsFromUser", "bundleManager.getBundleInfoForSelf", "window.getLastWindow", "window.getWindowProperties", Search, SearchController, Navigation, toolbarConfiguration, onPageShow, onPageHide]
permissions: ["ohos.permission.INTERNET", "ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-02-0100, HW-02-0101, HW-02-0102, HW-02-0103, HW-02-0104, HW-02-0105, HW-02-0106, HW-02-0149, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when a map (or any full-bleed canvas) needs a **panel that is
always on screen, draggable between heights, and never in the way** - the shape
every navigation app's home screen uses.

Three `bindSheet` options do the work, and each one is doing something
non-obvious:

- **`detents: [min, medium, max]`** gives the panel discrete resting heights
  instead of free dragging, so each stop can be sized to show exactly one
  section of content.
- **`enableOutsideInteractive: true`** removes the modal mask, so pinching and
  panning the map underneath still works while the panel is up.
- **registering `onWillDismiss` at all** is what stops the panel from being
  dismissed - the reference says a registered callback suppresses the automatic
  dismissal, so an empty body makes the sheet permanent.

The fourth idea is the map resize: `onDetentsDidChange` reports the settled
height, and the map is given the remaining space plus a negative top margin so
its centre stays visible above the panel rather than behind it.

Take this for maps, media players with a queue drawer, editors with an
inspector panel. If the panel should be dismissible, drop `onWillDismiss` and
this becomes an ordinary sheet.

## Feature checklist

- A full-screen map with the my-location control enabled and a custom
  my-location marker.
- A bottom panel at one of three heights - 140, 320 or 800 vp - draggable
  between them.
- The panel cannot be dismissed: dragging it below the smallest detent springs
  it back.
- No close button and no dimming mask; the map stays interactive at every
  detent.
- The map is resized and shifted so its centre stays above the panel.
- Inside the panel: a search box, five travel-mode shortcuts, and home/work
  cards.
- Below the panel, a second permanently docked 60 vp sheet carries the bottom
  toolbar.
- Location permissions are checked on entry and requested when missing.
- The map is suspended in `onPageHide` and resumed in `onPageShow`.

## Architecture

One `entry` module, one page. Everything is in `MapHome.ets`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/MapHome.ets      THE CARD: map, permissions, both sheets, the resize
└── util/LogUtil.ets       singleton hilog wrapper
```

The documented tree matches the zip.

**Two sheets, nested, both permanent.**

```
Stack
└── MapComponent .height(mapWindowHeight) .margin({ top: -heightVal })
    └── bindSheet(true, SheetBuilder, { detents: [140, 320, 800], ... })   the content panel
        └── Column                                                         the panel body
            └── bindSheet(true, NavigationBuilder, { detents: [60], ... }) the toolbar
```

The inner sheet is attached to the outer sheet's own root `Column`, so the
toolbar sits below the content panel and stays put while the panel is dragged.
The document shows neither the nesting nor the inner sheet (`HW-02-0106`).

**How the map is kept visible.** `onDetentsDidChange` reports the settled height
in **px**; the page converts to vp, stores it in `heightVal`, and derives:

```
mapWindowHeight = windowHeight - heightVal + 10      the map gets the space above the panel
margin.top      = -heightVal                          ...then slides up, re-centring the view
```

Both depend on `windowHeight`, which arrives asynchronously from
`window.getLastWindow` - the race in `HW-02-0105`.

**Permission flow**, in `aboutToAppear`:

```
checkPermissions()                 -> per-permission checkAccessToken
  false -> requestPermissions()    -> atManager.requestPermissionsFromUser
```

Both halves are defective: the check is an OR where it must be an AND
(`HW-02-0100`), and the request ignores its result (`HW-02-0101`).

## Implementation steps

1. **Enable Map Kit in AppGallery Connect and sign the build.** The document's
   说明 block lists the steps. Do **not** carry the sample's `client_id`
   metadata forward - it is obsolete from HarmonyOS 5.0.2(14) and belongs to
   someone else's project (`HW-02-0104`).
2. **Declare all three permissions** in `module.json5`, with `reason` and
   `usedScene` on the two user-grant location ones.
3. **Check every permission, not any of them** (`HW-02-0100`) - the reference
   requires `LOCATION` to be requested together with `APPROXIMATELY_LOCATION`.
4. **Inspect `authResults` from the request** and handle a denial
   (`HW-02-0101`); a rejected promise means the call failed, not that the user
   said no.
5. **Await the window height before anything computes against it**
   (`HW-02-0105`), and seed `mapWindowHeight` to something non-zero.
6. **Build the map with `MapComponent({ mapOptions, mapCallback })`** and take
   the controller from the callback; do the `setMyLocationEnabled` /
   `setMyLocationStyle` / `getEventManager` work there, not earlier.
7. **Point `MyLocationStyle.icon` at a file that exists** - the string form
   resolves under `resources/rawfile`, or use `$r()` (`HW-02-0102`).
8. **Attach the sheet with `bindSheet(true, ...)`** - a literal, because this
   panel is not toggleable - and set `detents`, `preferType: SheetType.BOTTOM`,
   `showClose: false`, `enableOutsideInteractive: true`.
9. **Register `onWillDismiss` and do not call `dismiss()` in it.** That is the
   documented way to make the sheet permanent.
10. **Resize the map in `onDetentsDidChange`,** converting the px argument with
    `px2vp`.
11. **Nest a second single-detent sheet** inside the panel for the toolbar.
12. **Suspend and resume the map** in `onPageHide` / `onPageShow`.

## Verified snippets

All snippets are from `MapBindSheet.zip`. Corrected forms are marked.

**The sheet - `MapBindSheet.zip#entry/src/main/ets/pages/MapHome.ets:191`** (as shipped)

```typescript
build() {
  Stack() {
    MapComponent({ mapOptions: this.mapOptions, mapCallback: this.callback })
      .height(this.mapWindowHeight)
      .margin({ top: 0 - this.heightVal })
      .bindSheet(true, this.SheetBuilder(), {          // literal true - the panel is permanent
        detents: [this.detentsMin, this.detentsMedium, this.detentsMax],   // 140 / 320 / 800
        preferType: SheetType.BOTTOM,
        showClose: false,
        enableOutsideInteractive: true,                // no mask - the map stays interactive
        onWillDismiss: (() => {                        // registered but never dismisses
          this.log.logInfo(TAG, `bindSheet scroll to bottom`);
        }),
        onDetentsDidChange: ((heightVal) => {
          if (this.heightVal > 0) {
            return;                                    // runs once - see HW-02-0105
          }
          this.heightVal = this.uiContext.px2vp(heightVal);      // the callback reports px
          if (this.heightVal >= (this.detentsMedium.valueOf() as number)) {
            return;
          }
          this.mapWindowHeight = this.windowHeight - this.heightVal + WINDOW_HEIGHT_OFFSET;
        })
      });
  }
  .height('100%');
}
```

**An empty `onWillDismiss` is not a stub - it is the mechanism.** The reference
is explicit: "If this callback is registered, the sheet is not dismissed
immediately when the user performs the above operations. Instead, you can use
the `reason` parameter ... to decide whether to dismiss the sheet. If this
callback is not registered, the sheet is dismissed immediately." So registering
it and never calling `dismissSheetAction.dismiss()` makes the panel undismissable
by pull-down, back gesture or Esc. Delete the callback and the panel disappears
the first time the user drags it down.

**`enableOutsideInteractive: true` also removes the mask.** The reference: "The
value true means that interactions are allowed, in which case no mask is not
displayed... If this parameter is set to true, the setting of `maskColor` does
not take effect." That is why the map beneath stays both visible and touchable.
Without it a bottom sheet defaults to non-interactive with a dimming mask.

**`onDetentsDidChange` returns px, not vp.** The reference notes "The return
value is in px" for `onDetentsDidChange`, `onHeightDidChange` and
`onWidthDidChange` alike - hence the `px2vp` on the next line. Forgetting it
gives a height roughly three times too large on a typical phone.

**`this.detentsMedium.valueOf() as number`** is needed because the detents are
typed `SheetSize | Length`; `valueOf()` narrows the union enough to compare.

**The map callback - same file, line 77** (as shipped)

```typescript
this.callback = async (err, mapController) => {
  if (!err) {
    this.mapController = mapController;
    this.mapController?.setMyLocationEnabled(true);
    this.mapController?.setMyLocationStyle(this.locationStyle);
    this.mapEventManager = this.mapController.getEventManager();

    let mapLoad = () => {
      this.log.logInfo(TAG, `on-mapLoad...`);
    };
    this.mapEventManager.on('mapLoad', mapLoad);

    let myLocationButtonClick = () => {
      geoLocationManager.getCurrentLocation().then((location: geoLocationManager.Location) => {
        if (location) {
          this.mapController?.setMyLocation(location);
        }
      });
    };
    this.mapEventManager.on('myLocationButtonClick', myLocationButtonClick);
  }
};
```

**The controller only exists inside this callback.** `MapComponent` hands it
back asynchronously once the map surface is ready, so every call that configures
the map has to live here - assigning `mapController` in `aboutToAppear` and
configuring it later would race the surface.

**The my-location button is not automatic.** `myLocationControlsEnabled: true`
in the options draws the button; `setMyLocationEnabled(true)` turns on the blue
dot; and the `myLocationButtonClick` handler is what actually fetches a fix and
feeds it back with `setMyLocation`. All three are needed.

**Page lifecycle - same file, line 176** (as shipped)

```typescript
onPageShow(): void {
  if (this.mapController) {
    this.mapController.show();          // resume rendering
  }
}

onPageHide(): void {
  if (this.mapController) {
    this.mapController.hide();          // stop rendering in the background
  }
}
```

**This pair is the map's power management** and is easy to omit. Without
`hide()` the map surface keeps rendering while the application is backgrounded.
`onPageShow`/`onPageHide` fire only on an `@Entry` component, which is why the
map lives on the page rather than in a child.

**Location style - same file, line 68** (corrected, see `HW-02-0102`)

```typescript
this.locationStyle = {
  anchorU: 0.5,
  anchorV: 1,
  radiusFillColor: 0xffff0000,          // ARGB - opaque red accuracy circle
  icon: $r('app.media.my_location'),    // FIX: the sample writes 'test.png', which does not exist
  displayType: mapCommon.MyLocationDisplayType.FOLLOW
};
```

The string form of `icon` resolves under `resources/rawfile` per the reference,
and this project has no `rawfile` directory. The `Resource` form has been
accepted since 5.0.0(12) and is checked at build time.

`MyLocationDisplayType.FOLLOW` keeps the map centred on the user as they move -
the right default for a navigation home screen, and different from `DEFAULT`,
which only marks the position.

**Permissions - same file, line 129** (corrected, see `HW-02-0100` and `HW-02-0101`)

```typescript
async checkPermissions(): Promise<boolean> {
  const permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
  for (const permission of permissions) {
    // FIX: the sample returns true on the FIRST granted permission - an OR, not an AND
    if (await this.checkAccessToken(permission) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return false;
    }
  }
  return true;
}

requestPermissions(): void {
  const atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(this.context,
    ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'])
    .then((data: PermissionRequestResult) => {          // FIX: the sample takes no parameter
      const denied = data.authResults.some((r: number) => r !== 0);
      if (denied) {
        // tell the user, and offer atManager.requestPermissionOnSetting(this.context, permissions)
      }
    })
    .catch((err: BusinessError) => {
      this.log.logError(TAG, `Failed to request permissions. Code ${err.code}, message ${err.message}`);
    });
}
```

**The two location permissions are one decision, not two.** The reference's
prerequisite line for `ohos.permission.LOCATION` reads "This permission must be
requested with `ohos.permission.APPROXIMATELY_LOCATION`", and the guidance adds
"Apply for the foreground precise location permission: Declare **both**". The
shipped OR means a user who allows only approximate location is treated as fully
authorised.

**A resolved promise is not a grant.** `authResults` carries `0` for granted,
`-1` for not granted and `2` for an invalid request; the `catch` fires only when
the call itself fails. The shipped code logs "Success to request permissions"
on a flat refusal.

`checkAccessToken` goes through `bundleManager.getBundleInfoForSelf(...
GET_BUNDLE_INFO_WITH_APPLICATION)` to obtain `appInfo.accessTokenId` - that is
the documented way to get the token for a self-check, and it is worth copying
verbatim.

**The nested toolbar sheet - same file, line 338** (as shipped)

```typescript
@Builder
SheetBuilder() {
  Column() {
    Search({ placeholder: $r('app.string.searchPlaceHolder'), controller: this.searchController })
    // ... travel-mode row, home/company cards ...
  }
  .bindSheet(true, this.NavigationBuilder(), {
    detents: [60],                       // a single detent: a fixed dock
    preferType: SheetType.BOTTOM,
    showClose: false,
    enableOutsideInteractive: true,
    onAppear: () => { this.log.logInfo(TAG, 'NavigationBuilder onAppear.'); },
    onWillDismiss: (() => {              // again: registered, never dismisses
      this.log.logInfo(TAG, `NavigationSheet scroll to bottom`);
    }),
  })
  .width('100%')
  .height('100%');
}
```

**A one-element `detents` array is how you dock something.** With a single
height there is nothing to drag to, so the sheet becomes a fixed bar - and
because it is attached to the *content panel's* root rather than to the map, it
layers beneath the panel and stays put while the panel moves.

Both sheets set `enableOutsideInteractive: true`, so neither adds a mask and
touches fall through to the map.

`Search`'s `controller` must be a `SearchController`; the sample passes a
`TextInputController` (`HW-02-0103`).

## Permissions & config

`MapBindSheet.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    // FIX: obsolete since HarmonyOS 5.0.2(14), and this value is the sample author's - HW-02-0104
    // "metadata": [{ "name": "client_id", "value": "1563631964248907072" }],
    "requestPermissions": [
      { "name": "ohos.permission.INTERNET" },
      {
        "name": "ohos.permission.LOCATION",
        "reason": "$string:location_permission",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      },
      {
        "name": "ohos.permission.APPROXIMATELY_LOCATION",
        "reason": "$string:approximately_location_permission",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      }
    ]
  }
}
```

The permission block is the most complete in this industry and worth copying:

- `ohos.permission.INTERNET` - system-grant, needs no `reason`.
- The two location permissions are **user-grant**, so each carries a `reason`
  string resource (shown in the system dialog) and a `usedScene` naming the
  ability and `"when": "inuse"`. Omitting `reason` on a user-grant permission is
  a build error.
- `LOCATION` may not be requested alone; `APPROXIMATELY_LOCATION` is its
  documented prerequisite.

Both must also be requested at runtime with `requestPermissionsFromUser` -
declaration alone is not enough.

Setting up Map Kit additionally requires enabling the service in AppGallery
Connect and a manual signing configuration, as the document's 说明 block at
lines 55-60 describes.

Root `build-profile.json5` targets `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 64-66).
- **Map Kit must be enabled in AppGallery Connect and the build manually
  signed** before the map renders at all. This sample cannot be run from the zip
  without an AGC project.
- Map Kit is a Chinese-mainland service; availability is regional.
- The detents are fixed vp numbers (140 / 320 / 800). On a screen shorter than
  800 vp the largest detent is clamped, and the panel content is not
  re-flowed for it.
- `onDetentsDidChange` reports **px**; every consumer must convert.
- The map resize runs exactly once, on the first detent change - later drags do
  not resize it.
- The initial camera is hard-coded to latitude 32, longitude 119 at zoom 15
  (`MapHome.ets:56-65`); there is no "start at the user's position" path.
- Nothing in the sample unsubscribes the two `MapEventManager` listeners
  (`HW-02-0149`). `MapEventManager.off` does exist - `LIFE-20`'s sample uses it
  for all five of its events - so the omission here is a leak, not a limitation.

## Pitfalls

- **`HW-02-0100` - the permission check is an OR.** It returns `true` on the
  first granted permission, so granting only approximate location suppresses the
  request for precise location - which the reference says must be requested
  alongside it.
- **`HW-02-0101` - `requestPermissionsFromUser` ignores `authResults`** and logs
  success on a denial. The `catch` only fires when the call itself fails.
- **`HW-02-0102` - `MyLocationStyle.icon: 'test.png'` cannot resolve.** The
  string form is looked up under `resources/rawfile`, and the project has no
  such directory. Use `$r()`.
- **`HW-02-0103` - `Search` is given a `TextInputController`** where
  `SearchOptions.controller` is typed `SearchController`.
- **`HW-02-0104` - `module.json5` ships a `client_id` metadata entry** that the
  Map Kit guide says is unnecessary from HarmonyOS 5.0.2(14), holding the sample
  author's own AGC project identifier.
- **`HW-02-0105` - `windowHeight` arrives asynchronously** while
  `onDetentsDidChange` reads it synchronously, and the `heightVal > 0` guard
  makes the first, possibly-zero computation permanent. The map can be laid out
  at zero or negative height.
- **`HW-02-0106` - the document binds `$$this.isShow`,** a field that does not
  exist in the sample (which binds a literal `true`), and never shows the nested
  toolbar sheet that produces the effect in the preview.
- **Do not delete the empty `onWillDismiss`.** Registering it is what makes the
  panel permanent; removing it lets the first downward drag dismiss the panel
  for good.
- **Do not forget `px2vp` on the detent callback.** The value is px.
- **Do not configure the controller before the map callback fires.** It does not
  exist until then.
- **`HW-02-0149` - the `mapLoad` and `myLocationButtonClick` listeners are never
  released.** `MapEventManager.off` exists - `LIFE-20` pairs registration and
  release for all five of its events - and both callbacks here close over the
  page, so they keep it reachable after it is destroyed.
- **Do not omit `enableOutsideInteractive: true`.** A bottom sheet defaults to
  modal, and the map underneath stops responding.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `detents`, `SheetType`, `enableOutsideInteractive`, `showClose`, `onWillDismiss` and its suppression semantics, `onDetentsDidChange` (returns px)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-guides/03_application-framework/arkts-sheet-page.md` - sheet detents and the secondary-confirmation pattern
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-sheet-page
- `documentation/harmonyos-references/03_application-services/map-common.md` - `MapOptions`, `MyLocationStyle` and the `resources/rawfile` rule for a string `icon`, `MyLocationDisplayType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Kit, and the note that client ID is not needed from HarmonyOS 5.0.2(14)
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `MapEventManager.on` for `mapLoad` and `myLocationButtonClick`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.LOCATION` and its `APPROXIMATELY_LOCATION` prerequisite
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` - `authResults` values 0 / -1 / 2
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - `SearchOptions.controller: SearchController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `LIFE-08` - the same industry's other `bindSheet`, used as a dismissible editor rather than a docked panel
- `LIFE-16`, `LIFE-20`, `LIFE-26`, `LIFE-29` - the industry's other Map Kit scenarios
