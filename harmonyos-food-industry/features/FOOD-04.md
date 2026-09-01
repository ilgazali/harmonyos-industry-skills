---
id: FOOD-04
title: Store address and route - a static map thumbnail that hands off to Petal Maps
industry: 17_food
doc: huawei_industry_tree/17_food/docs/04_map_navigation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
sample: huawei_industry_tree/17_food/downloads/MapNavigation.zip
kits: ["@kit.MapKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["staticMap.getMapImage", StaticMapOptions, StaticMapMarker, "staticMap.IconSize", "petalMaps.openMapPoiDetail", "petalMaps.openMapNavi", "petalMaps.openMapRoutePlan", PoiDetailParams, NaviParams, RoutePlanParams, "UIContext.getHostContext", "UIContext.px2vp", "@StorageProp", Rating, "window.getWindowAvoidArea"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-17-0022, HW-17-0023, HW-17-0024, HW-17-0025, HW-17-0029]
status: verified-with-fixes
---

## When to use

Load this card when a detail page has to **show where a place is and get the
user walking or driving there**, and you do not want to embed a live,
interactive map. The pattern: render the location as a single static image
fetched from Map Kit, and treat every gesture on it as a handoff into the
Petal Maps app - POI details on tap, turn-by-turn navigation and route
planning on two buttons.

This is the cheap and correct shape for a store detail page. A `MapComponent`
costs a map instance, a lifecycle, and a camera to keep in sync; a static
image is one `PixelMap` in `@State` and no ongoing cost. The app never owns
the map experience - it owns a picture of it, and Petal Maps owns everything
after the tap.

It generalises to any address card: a hotel, a clinic, a station, a venue in a
ticketing app. The one thing to decide up front is **which handle identifies
the destination** - a coordinate pair or a POI id. The sample gets this wrong
in the way most first implementations do (`HW-17-0023`), and the mistake is
invisible until you use the component for a second store.

## Feature checklist

- A store detail page: name, star rating, price band, hero image, open/closed
  badge, opening hours, address, map, buttons, reviews.
- A square map thumbnail centred on the store, with a marker pin on it,
  fetched over the network and shown as an ordinary `Image`.
- Tapping the thumbnail opens the store's POI detail page inside Petal Maps.
- A 导航 (navigate) button opens Petal Maps in turn-by-turn navigation to the
  store.
- A 路线 (routes) button opens Petal Maps route planning to the store.
- The page runs full-screen, with the status bar and navigation indicator
  heights padded out rather than overlapped.

## Architecture

One `entry` module, two ArkUI files. No model layer, no network layer - Map
Kit is the network layer.

```
entry/src/main/ets
├── components/StaticMapComponent.ets   the StaticMap struct: fetch + render + POI tap
├── entryability/EntryAbility.ets       full-screen layout, avoid areas -> AppStorage
└── pages/StoreDetailPage.ets           @Entry, the whole detail page in three @Builders
```

The documented tree matches the zip.

`StoreDetailPage` is three builders - `mapTriggerCard`, `commentsArea`,
`funcButton` - and two literal coordinates in `@State`, carrying the comment
"变量值仅用作功能展示，使用时根据实际地址信息设置" (values are for demonstration
only; set them from the real address in production).

**The design decision worth copying** is the split of responsibility along the
Petal Maps API boundary. `StaticMap` owns exactly one thing: turning a
coordinate into a picture, and turning a tap on that picture into
`openMapPoiDetail`. The two buttons that leave the app for navigation and
route planning live in the page, not in the component, because they belong to
the page's action bar and not to the map. That is the right cut: the component
is reusable in a list row or a card, and it does not drag two buttons with it.

**The part worth avoiding** is what that component takes as input. It receives
`latitude` and `longitude` as `@Prop` but pins `destinationPoiId` to a string
literal, so the coordinates are parameterised and the identity is not - and
the reference is explicit that "when both the POI ID and coordinates are
passed in, the POI ID will be used preferentially", so the hardcoded id wins
over the props every time (`HW-17-0023`). It also takes `UIContext` as a
`@Prop`, which deep-copies a framework handle (`HW-17-0025`).

## Implementation steps

1. **Enable the map service in AppGallery Connect** before writing any code:
   My Projects → your app → API Management → turn on 地图服务 (Map Service),
   then complete manual signing. Without this every Map Kit call fails with
   `1002600004`, "The Map permission is not enabled".
2. **Declare `ohos.permission.INTERNET`** in `entry/src/main/module.json5`.
   The static map image is a network fetch; the sample ships with no
   `requestPermissions` block at all although its own document requires this
   permission (`HW-17-0022`).
3. **Build the marker array first,** then the options object. The marker is
   what puts a pin on the store; without it the image is just a map centred
   near the store with nothing identifying it.
4. **Keep `zoom` inside the documented range.** `StaticMapOptions.zoom` must
   be an integer from 2 to 17 - note this is a *different* range from
   `PoiDetailParams.zoom` (3 to 20). The sample passes 20.
5. **Match `imageWidth`/`imageHeight` to `scale`:** with `scale: 2` each
   dimension is capped at 512 px and the returned image is 1024 px on a side.
6. **Attach a `.catch` to `getMapImage`** and a `try/catch` to every
   `petalMaps` call, and log the error code (`HW-17-0024`). The first-run
   failure mode is exactly the setup step in step 1, and unhandled it shows as
   a blank pink box with no message.
7. **Pass the POI id in as a prop** alongside the coordinates, or omit it
   entirely and let the coordinates drive the detail page (`HW-17-0023`).
8. **Do not pass `UIContext` as `@Prop`.** Call `this.getUIContext()` inside
   the child, or accept it as a plain undecorated property (`HW-17-0025`).
9. **Pad for the avoid areas** with `@StorageProp('topRectHeight')` /
   `('bottomRectHeight')` published by `EntryAbility` and converted with
   `px2vp` at the point of use.

## Verified snippets

All snippets are from `MapNavigation.zip`. Corrected forms are marked.

**Fetching the static image — `entry/src/main/ets/components/StaticMapComponent.ets`**
(corrected, see `HW-17-0024`)

```typescript
import { petalMaps, staticMap } from '@kit.MapKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

@Component
export struct StaticMap {
  @State image?: PixelMap = undefined;
  @Prop latitude: number;
  @Prop longitude: number;

  aboutToAppear(): void {
    this.init()
  }

  init() {
    // 设置静态图标记参数 - the pin drawn on the image
    let markers: Array<staticMap.StaticMapMarker> = [{
      location: {
        latitude: this.latitude,
        longitude: this.longitude
      },
      defaultIconSize: staticMap.IconSize.SMALL
    }];
    // 拼装静态图参数
    let option: staticMap.StaticMapOptions = {
      location: {
        latitude: this.latitude,
        longitude: this.longitude
      },
      zoom: 20,          // as shipped; the reference caps StaticMapOptions.zoom at 17
      imageWidth: 512,   // px, capped at 512 because scale is 2
      imageHeight: 512,
      scale: 2,          // returned image is 1024 x 1024 px
      markers: markers,
    };
    // 获取静态图
    staticMap.getMapImage(option).then((value) => {
      this.image = value;
    }).catch((err: BusinessError) => {          // FIX: absent in the sample
      hilog.error(0x0000, 'StaticMap',
        'getMapImage failed, code: %{public}d, message: %{public}s', err.code, err.message);
    });
  }
}
```

**Three options carry the design.** `markers` is what makes the image
*about* a store rather than about an area - the location is passed twice, once
to centre the camera and once to place the pin, and both are needed.
`scale` is a resolution multiplier, not a size: the width and height limits
halve when it is 2, so `512 × scale 2` is the largest square this API returns
and is what keeps the thumbnail sharp on a high-density panel. `zoom` is the
one value the sample gets wrong; the reference says it "must be an integer
ranging from 2 to 17. If a decimal is passed, it will be rounded down", and an
out-of-range value falls back rather than clamping the way you would expect.

The `.catch` is not defensive decoration. `getMapImage` rejects with
`1002600004` when the map service has not been switched on in AGC - the very
step the document devotes a four-point checklist to - and without a handler
that becomes an unhandled promise rejection while the UI silently shows the
component's `Color.Pink` background.

**The POI handoff — same file** (corrected, see `HW-17-0023`, `HW-17-0025`)

```typescript
@Prop poiId: string;          // FIX: was a literal '563233191438217472' in the body

@Builder
mapShow() {
  // 展示获取的静态图
  Image(this.image)
    .width($r('app.string.width_or_height_fullsize'))
    .height($r('app.float.map_height'))
    .fitOriginalSize(false)
    .objectFit(ImageFit.Cover)
    .onClick(async () => {
      let params: petalMaps.PoiDetailParams = {
        destinationPosition: {
          latitude: this.latitude,
          longitude: this.longitude
        },
        destinationPoiId: this.poiId
      };
      try {
        // FIX: sample uses this.context, a @Prop-copied UIContext
        await petalMaps.openMapPoiDetail(this.getUIContext().getHostContext(), params);
      } catch (err) {
        hilog.error(0x0000, 'StaticMap', 'openMapPoiDetail failed: %{public}s', JSON.stringify(err));
      }
    })
}
```

**A POI id is not a decoration on the coordinates, it overrides them.** The
`PoiDetailParams` reference states that "when both the POI ID and coordinates
are passed in, the POI ID will be used preferentially", and adds that "when
`destinationPoiId` is set, the map zoom level cannot be customized". So the
shipped component, given any other store's latitude and longitude, still opens
the one POI whose id is baked in - and the coordinates it was parameterised
with are ignored. Either the id is a prop, as here, or it is dropped so the
coordinates take effect.

`objectFit(ImageFit.Cover)` with `fitOriginalSize(false)` is what lets a
1024×1024 square fill a wide, short map strip without letterboxing: the image
is cropped to the box rather than scaled to fit it. The centre of the crop is
the centre of the map, which is where the marker is - so the pin survives the
crop.

**Leaving for navigation and route planning — `entry/src/main/ets/pages/StoreDetailPage.ets`**
(corrected, see `HW-17-0024`)

```typescript
@Builder
funcButton() {
  Row({ space: 16 }) {
    Button($r('app.string.button_nav'))                  // 导航 - turn-by-turn
      .layoutWeight(1)
      .fontColor($r('sys.color.mask_secondary'))
      .backgroundColor($r('app.color.button_nav_background'))
      .onClick(async () => {
        let params: petalMaps.NaviParams = {
          destinationPosition: {
            latitude: this.latitude,
            longitude: this.longitude
          },
          // vehicleType: 0  可选择导航出行方式
        };
        try {                                            // FIX: absent in the sample
          await petalMaps.openMapNavi(this.context.getHostContext(), params);
        } catch (err) {
          hilog.error(0x0000, 'StoreDetail', 'openMapNavi failed: %{public}s', JSON.stringify(err));
        }
      })
    Button($r('app.string.button_routes'))               // 路线 - route planning
      .layoutWeight(1)
      .onClick(async () => {
        let params: petalMaps.RoutePlanParams = {
          destinationPosition: {
            latitude: this.latitude,
            longitude: this.longitude
          }
        };
        try {
          await petalMaps.openMapRoutePlan(this.context.getHostContext(), params);
        } catch (err) {
          hilog.error(0x0000, 'StoreDetail', 'openMapRoutePlan failed: %{public}s', JSON.stringify(err));
        }
      })
  }
  .width($r('app.string.width_or_height_fullsize'))
  .justifyContent(FlexAlign.SpaceBetween)
}
```

**The two calls differ only in intent, and that is the point.** `NaviParams`
and `RoutePlanParams` take the same destination shape; picking `openMapNavi`
starts guidance immediately, `openMapRoutePlan` opens the route preview with
the destination filled in. Neither needs an origin - Petal Maps resolves the
user's current position itself, which is why this whole feature needs no
location permission in the calling app.

`vehicleType` is left commented out in the sample. Setting it (`0` for
driving) skips the mode picker, which is worth doing when the app already
knows - a takeaway courier flow should not ask.

Note that both handlers use `this.context`, a `private context: UIContext =
this.getUIContext()` captured once in the page. On the page that is fine -
it is a plain property, evaluated in the component's own context. The defect
is only in passing that value down into `StaticMap` as `@Prop`.

**Full-screen padding — same file** (as shipped)

```typescript
@StorageProp('topRectHeight')
topRectHeight: number = 0;
@StorageProp('bottomRectHeight')
bottomRectHeight: number = 0;
private context: UIContext = this.getUIContext()

// ...
.padding({
  top: this.context.px2vp(this.topRectHeight),
  bottom: this.context.px2vp(this.bottomRectHeight)
})
```

`EntryAbility` calls `setWindowLayoutFullScreen(true)` and subscribes to
`avoidAreaChange`, writing `topRect.height` and `bottomRect.height` into
`AppStorage` as **px**; the page converts with `px2vp` at the point of use.
Typed `@StorageProp` defaults of `0` mean a frame rendered before the first
`avoidAreaChange` pads by nothing rather than by `undefined`. This is the same
boilerplate as `FOOD-05`, which uses a one-shot `getWindowAvoidArea` read
instead of a subscription - the subscription is the better half of the two,
because it survives a rotation or a window resize.

## Permissions & config

**The sample declares none.** `entry/src/main/module.json5` has no
`requestPermissions` array at all, while the document's 权限说明 section
requires `ohos.permission.INTERNET` (`HW-17-0022`). Add it:

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

`INTERNET` is `system_grant`, so no `reason` or `usedScene` is needed and no
runtime prompt appears - which is exactly why the omission is easy to miss
until the map image never arrives.

The heavier configuration is outside the project: the map service switch in
AppGallery Connect plus manual signing. An unsigned or unswitched build
compiles and runs, and fails only at the API call.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `deviceTypes` is `phone`, `tablet`, `2in1`.
- **`zoom: 20` is outside the documented `StaticMapOptions` range of 2-17.**
  The 3-20 range belongs to `PoiDetailParams.zoom`, a different API on the
  same page; the two are easy to confuse and the sample appears to have.
- The static image is a snapshot: no panning, no zooming, no live traffic. If
  the user needs to explore, the handoff to Petal Maps is the answer, not a
  bigger thumbnail.
- Everything after the tap happens in Petal Maps. If it is not installed the
  `petalMaps` calls reject - so the `catch` from `HW-17-0024` is also the
  no-map-app path, not only the misconfiguration path.
- The store name, address, hours, reviews and rating are all static string
  resources. `rating` is a `@State` literal `3.0` with `indicator: true`, so
  the stars are display-only.
- `Color.Pink` is the `StaticMap` container background - visible while the
  image is loading and, without a fetch error handler, forever after a
  failure.

## Pitfalls

- **`HW-17-0022`** (D/high, confirmed): the sample declares no permissions at
  all although the document's 权限说明 requires `ohos.permission.INTERNET` and
  `staticMap.getMapImage` is a network call. Fix: add a `requestPermissions`
  array with `ohos.permission.INTERNET` to `entry/src/main/module.json5`.
- **`HW-17-0023`** (B/low, confirmed): the reusable `StaticMap` hardcodes
  `destinationPoiId: '563233191438217472'` while taking the coordinates as
  props, and the POI id takes precedence over coordinates - so every store's
  map tap opens the same place. Fix: accept `poiId` as a `@Prop` and pass it
  from `StoreDetailPage`, or drop it.
- **`HW-17-0024`** (B/low, confirmed): `getMapImage` has no `.catch` and the
  three `petalMaps` calls have no `try/catch`, although the reference
  documents `1002600004` "The Map permission is not enabled" - the exact
  failure the document's own setup checklist is about. Fix: catch, log the
  code, and show a toast.
- **`HW-17-0025`** (C/low, probable): `@Prop context: UIContext` deep-copies a
  framework handle on every sync, and the clone's `getHostContext()` need not
  match the live window context. Fix: drop the decorator, or call
  `this.getUIContext()` inside `StaticMap`.

## References

- `documentation/harmonyos-references/03_application-services/map-staticmap.md` - `getMapImage`, `StaticMapOptions`, `StaticMapMarker`, the error table
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-staticmap
- `documentation/harmonyos-references/03_application-services/map-petal-maps.md` - `openMapPoiDetail`, `openMapNavi`, `openMapRoutePlan` and their params
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-petal-maps
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - switching on the map service in AppGallery Connect
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/04_application-services/map-petalmaps.md` - the handoff guide
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-petalmaps
- `documentation/harmonyos-guides/08_coding-and-debugging/ide-signing.md` - manual signing, required before Map Kit works
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-signing
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `FOOD-01` - the full food app template, which also uses Map Kit and location
- `FOOD-05` - the same avoid-area boilerplate in its one-shot form
