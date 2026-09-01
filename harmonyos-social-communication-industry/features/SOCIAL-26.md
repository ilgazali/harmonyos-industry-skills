---
id: SOCIAL-26
title: Location card in chat - pick a POI with sceneMap, embed a MapComponent in the bubble, hand off to Petal Maps
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/26_chat_page_location_navigation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_location_navigation-0000002322113637
sample: huawei_industry_tree/14_social_communication/downloads/ChatPageLocationNavigation.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MapKit", "@kit.PerformanceAnalysisKit"]
apis: [bundleManager, common, hilog, map, mapCommon, petalMaps, sceneMap, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0060, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card for the **"send my location" flow in a chat**: the + button
opens a map picker, the chosen POI becomes a bubble with a live map preview,
tapping the bubble opens a full-screen detail page, and a button there hands
off to a real navigation app.

The pattern is worth studying because it uses **three different levels of Map
Kit in one screen**, and picking the right level per surface is the whole
design. `sceneMap.chooseLocation` is a complete system-provided picker page -
search box, nearby POI list, confirm button - which you launch and await;
you write no map UI for it at all. `MapComponent` is the embeddable map you
place inside your own layout, driven by a `MapComponentController` handed
back through an async callback. `petalMaps.openMapRoutePlan` is a pure
hand-off: you pass a destination and Petal Maps takes over.

It generalises past chat. Any app that lets a user attach a place - a
delivery address in an order flow, a meeting point in an event invite, a
check-in in a social post - wants exactly this triple: system picker in,
embedded preview in the feed, external navigation out.

**Read `HW-14-0060` before adopting the preview.** The sample shares one map
controller across every location bubble, which is fine with one card on
screen and wrong with two.

## Feature checklist

- Typing text shows a send button; the + button is hidden while text is
  present and reappears when the field is empty.
- Tapping + dismisses the keyboard and opens the system location picker with
  search and nearby-POI controls enabled.
- Confirming a place appends a card message carrying its name, address and
  coordinates.
- The card renders the name, the address and a live map beneath them, with a
  chat-bubble triangle on the correct side.
- Sender alternates on every send, so the demo shows both own and peer cards.
- Tapping a card pushes a detail page with a full-bleed map, a marker on the
  place, and a back arrow.
- Tapping the detail page's bottom bar opens a bottom sheet listing the
  installed map apps; picking Petal Maps opens it with the route planned.
- If no map app is installed, the sheet shows a toast instead of navigating.
- Leaving either page hides the map; returning shows it again.

## Architecture

One `entry` module, seven ArkUI files in the classic
components/constants/datas/dialogs/pages/utils split.

```
entry/src/main/ets
├── components/BottomActionBar.ets   input row: text, emoji, +, send; owns chooseLocation
├── constants/CommonConstant.ets     Constant + PageConstant (route names)
├── datas/MsgData.ets                MessageType enum + MsgContent interface
├── dialogs/SelectMapDialog.ets      @CustomDialog bottom sheet listing map apps
├── entryability/EntryAbility.ets    full-screen window, avoid areas -> AppStorage
├── entrybackupability/
├── pages
│   ├── ChatPage.ets                 the list, the text builder, the card builder
│   ├── LocationDetailPage.ets       full-screen map + name/address + navigate
│   └── NavigationPage.ets           @Entry: NavPathStack root and pageMap
└── utils/DisplayUtil.ets            px2vp over the AppStorage avoid-area values
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the routing. `NavigationPage` is
nineteen lines: it owns the `NavPathStack`, publishes it as
`@Provide('pageInfo')`, pushes `ChatPage` in `aboutToAppear`, and maps route
names to components in a `@Builder pageMap`. Both destinations consume the
stack with `@Consume(Constant.PAGE_INFOS)`, so neither needs a router import
and neither knows how it was reached. The card passes its whole `MsgContent`
as the route param -

```typescript
this.pageInfos.pushPath({ name: PageConstant.PAGE_LOCATION_DETAIL, param: item });
```

- and the detail page reads it back with
`getParamByIndex(this.pageInfos.size() - 1)`. No shared store, no id lookup:
the message *is* the navigation argument.

**The decision worth avoiding** is that `ChatPage` holds `mapOptions`,
`callback` and `mapController` as three single-valued fields while rendering
*N* map cards from them. That is `HW-14-0060`. `LocationDetailPage` has the
same three fields and is correct, because it only ever shows one map.

## Implementation steps

1. **Set up the AGC project first.** Map Kit requires the Map service to be
   switched on for the app in AppGallery Connect and a manual signing profile;
   without it `MapComponent` renders blank and `chooseLocation` fails. The doc
   lists the five steps.
2. **Root the app in a `Navigation`** with a `@Provide`d `NavPathStack` and a
   `pageMap` builder; push the chat page from `aboutToAppear`.
3. **Model messages as one interface with a type tag** (`MessageType.TEXT` /
   `CARD`) and branch in the list item, rather than two parallel arrays.
4. **Raise the picker from the input bar**, not the page: clear focus first,
   then call `sceneMap.chooseLocation` with `searchEnabled` and
   `showNearbyPoi` true, and `.catch` the `BusinessError`.
5. **Build the card's `mapOptions` from `item.location`, per item**, and keep
   one controller per card - do not share the page-level fields
   (`HW-14-0060`).
6. **Take the controller from the `mapCallback`,** check `err` first, and call
   `setZoomControlsEnabled(false)` on a preview-sized map.
7. **Add the marker inside the callback**, as the detail page does, instead of
   from a timer after the component appears.
8. **Show and hide the map with the page** using `NavDestination.onShown` /
   `onHidden`; an embedded map holds a render surface and must be released
   when the page is not visible.
9. **Probe for a map app with `bundleManager.canOpenLink`** before offering
   navigation, and toast when the list is empty.
10. **Hand off with `petalMaps.openMapRoutePlan`,** passing
    `destinationPosition` and `destinationName`, then close the sheet.

## Verified snippets

All snippets are from `ChatPageLocationNavigation.zip`. Corrected forms are
marked.

**Raising the system picker — `entry/src/main/ets/components/BottomActionBar.ets`** (as shipped)

```typescript
import { sceneMap } from '@kit.MapKit';
import { common } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';

naviChooseLocationPage() {
  let locationChoosingOptions: sceneMap.LocationChoosingOptions = {
    location: { longitude: Constant.TEMP_LONGITUDE, latitude: Constant.TEMP_LATITUDE },
    searchEnabled: true,   // 展示搜索控件
    showNearbyPoi: true    // 展示附近POI
  };
  // 拉起地点选取页
  sceneMap.chooseLocation(this.getUIContext().getHostContext() as common.UIAbilityContext,
    locationChoosingOptions)
    .then((data) => {
      this.sendLocationCard(data);
    }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'chooseLocation', `code: ${err.code}, message: ${err.message}`);
  });
}

private sendLocationCard(result: sceneMap.LocationChoosingResult) {
  this.isSelf = !this.isSelf;
  this.data.push({
    isSelf: this.isSelf,
    type: MessageType.CARD,
    text: '',
    name: result.name ?? '',
    address: result.address,
    location: result.location
  });
}
```

**Everything about the picker is in the options object.** `location` is only
the *initial* camera centre - here a hardcoded pair of coordinates near
Dongguan, which in a real app would be the device's last known position.
`searchEnabled` adds the search field, `showNearbyPoi` the POI list under the
map; both default to off, and with both off the user can only drag a pin.
The result is a `LocationChoosingResult` with `name`, `address` and
`location`, which maps one-to-one onto the message fields - `name` is
optional and is defaulted with `?? ''`.

The call takes a `common.UIAbilityContext`, not a UI context: it starts a
system page as a modal on top of the ability. That is also why the handler
calls `getFocusController().clearFocus()` first - leaving the soft keyboard up
while a system page launches produces a visible flicker.

**The card bubble with its embedded map — `entry/src/main/ets/pages/ChatPage.ets`** (corrected, see `HW-14-0060`)

```typescript
// FIX: shipped code keeps one mapOptions / callback / mapController on the page
//      and passes them to every card. Derive them per item instead.
@Builder
cardMsgBuilder(item: MsgContent) {
  Stack() {
    Row() {
      Image($r('app.media.icon_triangle_left'))
        .fitOriginalSize(true)
        .visibility(item.isSelf ? Visibility.None : Visibility.Visible);

      Column() {
        Column() {
          Text(item.name)
            .fontSize($r('app.float.fontsize_14'))
            .maxLines(Constant.NUM_1)
            .textOverflow({ overflow: TextOverflow.Ellipsis });
          Text(item.address)
            .fontSize($r('app.float.fontsize_10'))
            .opacity(Constant.OPACITY_07)
            .maxLines(Constant.NUM_1)
            .textOverflow({ overflow: TextOverflow.Ellipsis });
        }.padding({ left: $r('app.float.vp_12'), top: $r('app.float.vp_8') });

        MapComponent({
          mapOptions: { position: { target: item.location, zoom: Constant.NUM_15 } },
          mapCallback: async (err, controller) => {          // FIX: one controller per card
            if (!err) {
              controller.setZoomControlsEnabled(false);
              controller.addMarker({ position: item.location });
            }
          }
        })
          .width(Constant.FULL_PERCENT)
          .layoutWeight(Constant.NUM_1)
          .borderRadius({ bottomLeft: $r('app.float.vp_8'), bottomRight: $r('app.float.vp_8') })
          .clip(true);
      }.layoutWeight(Constant.NUM_1)
      .backgroundColor(Color.White);

      Image($r('app.media.icon_triangle_right'))
        .visibility(item.isSelf ? Visibility.Visible : Visibility.None);
    }.alignItems(VerticalAlign.Top);

    Stack() {
    }.width(Constant.FULL_PERCENT).height(Constant.FULL_PERCENT)
    .onClick(() => {                                          // 位置卡片点击事件
      this.pageInfos.pushPath({ name: PageConstant.PAGE_LOCATION_DETAIL, param: item });
    });
  }
  .width(Constant.NUM_233)
  .height(Constant.NUM_156);
}
```

**The outer `Stack` with an empty `Stack` on top is the tap target,** and it
is deliberate: `MapComponent` consumes gestures, so an `onClick` on the card
`Column` would never fire over the map area. An empty full-size sibling
stacked above it swallows the taps and pushes the detail page. It also means
the preview map is not pannable inside the bubble, which is the right
behaviour for a chat card.

**The two triangles are one pair of mirrored assets** toggled by
`visibility(item.isSelf ? ... )` rather than by a conditional in the layout,
so the bubble keeps a stable node structure whichever side it is on.

The shipped version passes `this.mapOptions` (built once in `aboutToAppear`
from the hardcoded `TEMP_LATITUDE`/`TEMP_LONGITUDE`) and `this.callback`
(which assigns `this.mapController = mapController`) to *every* card. With
two location cards on screen the last one to initialise wins the field, and
the page's `@Watch('onChangedData')` handler then moves that camera and adds
the marker:

```typescript
// as shipped, ChatPage.onChangedData - the consequence of the shared controller
if (msgContent.type === MessageType.CARD) {
  setTimeout(() => {
    let cameraUpdate = map.newCameraPosition({ target: msgContent.location, zoom: Constant.NUM_15 });
    this.mapController?.moveCamera(cameraUpdate);
    this.mapController?.addMarker({ position: msgContent.location, icon: 'icon_location.png' });
  }, Constant.NUM_200);
}
```

So the newest location is applied to whichever controller registered last -
often an older card - while the new card stays at the hardcoded default
coordinates. The 200 ms timer is a race against the new map's
initialisation, not a fix. Deriving options from `item.location` and adding
the marker inside the per-card callback removes both the timer and the
`@Watch` branch.

**The detail page and the hand-off — `entry/src/main/ets/pages/LocationDetailPage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  this.msgContent = this.pageInfos.getParamByIndex(this.pageInfos.size() - 1) as MsgContent;
  let cameraPosition: mapCommon.CameraPosition = {
    target: this.msgContent.location,
    zoom: Constant.NUM_15
  };
  this.mapOptions = { position: cameraPosition };
  this.callback = async (err, mapController) => {
    if (!err) {
      this.mapController = mapController;
      this.mapController.setZoomControlsEnabled(false);
      let markerOptions: mapCommon.MarkerOptions = { position: this.msgContent.location };
      this.mapController?.addMarker(markerOptions);
    }
  };
}

// 拉起花瓣地图导航
async naviMapGuide() {
  let params: petalMaps.RoutePlanParams = {
    destinationPosition: {
      latitude: this.msgContent.location.latitude,
      longitude: this.msgContent.location.longitude,
    },
    destinationName: this.msgContent.name
  };
  await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params);
  if (this.selectMapDialog) {
    this.selectMapDialog.close();
  }
}
```

**This is the shape the cards should have had.** Options are built from the
message's own coordinates before the component exists, and the marker is
added inside the callback, at the one moment the controller is guaranteed
valid - no timer, no ordering assumption. The `err` check comes first
because `mapCallback` is an `AsyncCallback`, so `mapController` is undefined
on failure.

`openMapRoutePlan` needs only a destination; Petal Maps supplies the origin
from the device. There is **no route drawn inside this app** - the hand-off
is the feature, and it is the reason the sample needs no location permission
of its own.

Both `NavDestination`s pair `.onShown(() => mapController.show())` with
`.onHidden(() => mapController.hide())`. An embedded `MapComponent` keeps a
render surface alive; without the pair, a backgrounded chat keeps drawing
maps.

**The map-app sheet — `entry/src/main/ets/dialogs/SelectMapDialog.ets`** (as shipped)

```typescript
@CustomDialog
export struct selectMapDialog {
  mapScheme: string = 'maps://navigation';
  mapName: string = '花瓣地图';               // "Petal Maps"
  @State mapWays: Array<string> = [];
  openMap: () => void = () => {};

  aboutToAppear(): void {
    if (bundleManager.canOpenLink(this.mapScheme)) {
      this.mapWays.push(this.mapName);
    }
  }

  build() {
    Column() {
      Text($r('app.string.map_name'))
        .onClick(() => {
          if (this.mapWays.length > Constant.NUM_0) {
            this.openMap();
          } else {
            this.promptAction.showToast({ message: $r('app.string.map_tip') });
          }
        });
      Divider();
      Text($r('app.string.cancel'))
        .onClick(() => { this.selectMapDialog?.close(); });
    }
    .padding({ bottom: DisplayUtil.getBottomRectHeight(this.getUIContext()) });
  }
}
```

**`bundleManager.canOpenLink` is the check that keeps the sheet honest.** It
tests whether any installed app handles `maps://navigation`, and the scheme
must also be declared in the app's `querySchemes` to be visible. The sheet
degrades to a toast rather than throwing when nothing handles it - the right
behaviour for an optional hand-off.

The dialog takes `openMap` as a callback property rather than importing the
page, so it stays a dumb sheet, and it pads its bottom by the navigation-bar
height from `DisplayUtil` because `customStyle: true` disables the system's
own inset handling.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` only, with no `reason`/`usedScene` - correct, because it is
  `system_grant` and those fields are only mandatory for `user_grant`
  permissions.
- **No location permission is requested.** That is deliberate and worth
  noticing: `chooseLocation` runs in the system picker's own process and
  returns a place, so the app never reads the device position itself. If you
  add a "use my current location" shortcut you inherit
  `ohos.permission.LOCATION` and `APPROXIMATELY_LOCATION` along with it.
- Map Kit additionally needs the AGC configuration and a manual signing
  profile described in the doc; those are project settings, not manifest
  entries.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The sample will not run unsigned.** Map service must be enabled for the
  app in AGC and the debug signature regenerated afterwards.
- The picker's initial camera and the chat's default card position are the
  same hardcoded constants (`TEMP_LATITUDE` 22.878538,
  `TEMP_LONGITUDE` 113.886642). Replace both with a real seed.
- `isSelf` simply flips on every send, so the conversation alternates sides
  regardless of who "sent" anything - it is a layout demo, not a transport.
- There is no message list key generator at all: `ForEach(this.data, (item: MsgContent) => {...})`
  is called without the third argument, so ArkUI falls back to its default
  key. Supply an explicit unique key before this list grows.
- The voice and emoji buttons raise a `tips` toast; the only working
  attachment is location.
- The chat page terminates the ability on back press
  (`onBackPressed` → `terminateSelf`), which is demo behaviour.

## Pitfalls

- **`HW-14-0060`** (B/medium, confirmed): all location cards share one
  `mapOptions`, one `mapCallback` and one `mapController` held on `ChatPage`.
  The callback overwrites `this.mapController` on each map's init, and
  `onChangedData` moves the camera and adds the marker on whichever
  controller registered last, 200 ms after the fact. With two or more cards,
  older cards re-centre on the newest place and the new card can stay stuck
  at the hardcoded `TEMP` coordinates. Fix: build `mapOptions` from
  `item.location` per item and keep the controller per card, adding the
  marker inside that card's callback.
- **The 200 ms `setTimeout` is a race, not a delay.** It exists to wait for a
  map that may not have initialised; on a slow device the `moveCamera` call
  lands on the previous controller. Removing the shared state removes the
  need for the timer.
- **`MapComponent` swallows gestures.** The empty overlay `Stack` is what
  makes the card tappable; if you restructure the bubble, keep it.
- **Missing `onShown`/`onHidden` leaks a render surface.** Both pages pair
  them correctly - preserve the pair when copying the card builder into
  another page.
- **`canOpenLink` needs the scheme declared** in `querySchemes` in
  `module.json5` to see third-party map apps; without it the sheet can report
  "no map app" on a device that has one.

## References

- `huawei_industry_tree/14_social_communication/docs/26_chat_page_location_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_location_navigation-0000002322113637
- `documentation/harmonyos-guides/07_application-services/map-location-selecting.md` - `sceneMap.chooseLocation`, `LocationChoosingOptions`, `LocationChoosingResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location-selecting
- `documentation/harmonyos-guides/04_application-services/map-presenting.md` - `MapComponent`, `MapComponentController`, the `mapCallback` contract, show/hide
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-presenting
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - `MarkerOptions` and `addMarker`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-guides/04_application-services/map-petalmaps.md` - opening Petal Maps for route planning
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-petalmaps
- `documentation/harmonyos-references/03_application-services/map-petal-maps.md` - `petalMaps.openMapRoutePlan` and `RoutePlanParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-petal-maps
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling the Map service for the project
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/08_coding-and-debugging/ide-signing.md` - the manual signing step Map Kit requires
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-signing
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `pushPath`, `getParamByIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
