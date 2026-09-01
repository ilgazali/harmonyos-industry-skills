---
id: TOUR-07
title: Named map markers - annotation labels, custom icons and a custom info window
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/07_map_marker.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
sample: huawei_industry_tree/09_tourism/downloads/MapMarker.zip
kits: ["@kit.MapKit", "@kit.BasicServicesKit", "@kit.ArkUI"]
apis: [MapComponent, customInfoWindow, "map.MarkerDelegate", "map.MapComponentController", addMarker, "mapCommon.MarkerOptions", "mapCommon.Text", "mapCommon.TextPosition", "mapCommon.FontStyle", setInfoWindowAnchor, setInfoWindowVisible, setTitle, setClickable, "on('markerClick')", "@BuilderParam", "@Prop", "@Watch", "@StorageProp"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-09-0041, HW-09-0042, HW-09-0043, HW-09-0044, HW-09-0045, HW-09-0046]
status: verified-with-fixes
---

## When to use

Load this card when pins on a map must **carry their name on the map itself**,
not only in a popup - shops in a mall, attractions in a park, stations on a
line - and tapping one should both enlarge the pin and drive a detail panel
elsewhere on the screen.

Two Map Kit features do the work, and neither is obvious:

- **`annotations`** on `MarkerOptions` draws a styled text label attached to
  the pin, positioned above or below the icon. No separate overlay, no manual
  layout.
- **`customInfoWindow`** on `MapComponent` replaces the default bubble with
  your own `@Builder`, which receives a `MarkerDelegate` so one builder serves
  every marker.

Compare with `TOUR-06`, where the pins are generated from a POI search and the
selection drives a list; here the pins are authored, labelled, and drive a
card.

## Feature checklist

- A map centred on a district, filling the screen.
- Three shop pins at fixed coordinates, each with a custom icon from
  `rawfile`.
- Each pin carries the shop name as a label under its icon.
- Tapping a pin opens a custom info window that swaps in an enlarged version
  of that shop's icon.
- Tapping a pin also updates a shop detail card pinned to the bottom of the
  screen: photo, name, review count, average spend, rating.
- One shop's info window is open when the map first loads.
- A search bar and back button float over the top of the map.

## Architecture

One `entry` module, one page, two presentation components and a static model.

```
entry/src/main/ets
├── components
│   ├── ShopDetail.ets      the bottom card; @Prop @Watch on the selected id
│   └── TopContent.ets      the floating search bar and back button
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/DataModel.ets     ShopModel + SHOPS (3 static entries, index 0..2)
└── pages/Map.ets           @Entry: MapComponent, the markers, the info window
entry/src/main/resources/rawfile/
├── humer.png               marker icons - rawfile, not media
├── sushis.png
└── fishes.png
```

The documented tree matches the zip.

**Marker icons come from `rawfile`, not from `media`.** `MarkerOptions.icon`
takes a string path relative to `resources/rawfile` - `icon: 'humer.png'` -
whereas everything else in the sample uses `$r('app.media....')`. The two
resolution mechanisms sit side by side in this project: the pin icons in
`rawfile`, the enlarged info-window images and the card photos in `media`.

**Selection flows one way through a number:**

```
markerClick ──> this.clickedShop ──@Prop @Watch──> ShopDetail.onValueChange
     │                                                 └─> SHOPS.filter(index === clickedShop)
     └──> marker.setInfoWindowVisible(true) ──> customInfoWindowBuilder($$)
```

The `@BuilderParam` info window and the `@Prop @Watch` card are two
independent consumers of the same tap - the info window gets the marker
through `MarkerDelegate`, the card gets a number through state. That split is
the reusable idea; what identifies the marker should not be its `zIndex`
(`HW-09-0042`).

## Implementation steps

1. **Declare `INTERNET` and `GET_NETWORK_INFO`** and add the `client_id`
   metadata (`HW-09-0045`).
2. **Put the marker icons in `resources/rawfile`** and reference them by bare
   filename.
3. **Build the map** with `mapOptions`, `mapCallback` and `customInfoWindow`
   on one `MapComponent`.
4. **Add each marker with an `annotations` array** carrying the label, and set
   `annotationPosition` to `TextPosition.BOTTOM` so the name sits under the
   icon. Use the enums, not integers (`HW-09-0043`).
5. **Set `setInfoWindowAnchor(0.5, 1)`** so the custom window is anchored at
   the pin's tip rather than its centre.
6. **Carry a real identifier on the marker** - `setTitle` with the shop id -
   and read that back on tap.
7. **Open the info window from the tap handler** with
   `setInfoWindowVisible(true)`.
8. **Drive the detail card by `@Prop @Watch`**, resolving the id against the
   model.
9. **Release the listener** in `aboutToDisappear` (`HW-09-0041`).

## Verified snippets

All snippets are from `MapMarker.zip`. Corrected forms are marked.

**The component wiring — `entry/src/main/ets/pages/Map.ets`** (as shipped)

```typescript
import { map, mapCommon, MapComponent } from '@kit.MapKit';

// the custom info window is a BuilderParam, so it can be overridden by a parent
@BuilderParam customInfoWindow: ($$: map.MarkerDelegate) => void = this.customInfoWindowBuilder;

build() {
  Stack() {
    Stack() {
      MapComponent({
        mapOptions: this.mapOptions,
        mapCallback: this.callback,
        customInfoWindow: this.customInfoWindow    // third parameter: replaces the default bubble
      })
        .width('100%')
        .height('100%')
      ShopDetail({ clickedShop: this.clickedShop })
    }
    .width('100%')
    .align(Alignment.Bottom)                       // card pinned to the bottom of the map

    TopContent()                                   // search bar floating over everything
  }
  .height('100%')
  .align(Alignment.Top)
}
```

**Two nested `Stack`s with opposite alignments** is the layout idiom for a map
screen: the inner one bottom-aligns the detail card over the map, the outer
one top-aligns the search bar over both.

**A marker with a name label — same file** (corrected, see `HW-09-0043`, `HW-09-0044`)

```typescript
const markerOptions: mapCommon.MarkerOptions = {
  position: { latitude: 31.891836, longitude: 118.876121 },
  rotation: 0,
  visible: true,
  zIndex: 0,
  alpha: 1,
  anchorU: 0.5,
  anchorV: 1,                       // anchor the icon by its bottom centre: the pin tip
  clickable: true,
  draggable: true,
  flat: false,
  annotations: [{                   // the name label - max 3 entries, min 1
    content: shopName,              // FIX: the sample hardcodes '元师傅汉堡' here
    fontStyle: mapCommon.FontStyle.REGULAR,   // FIX: the sample passes 1, which is BOLD
    strokeWidth: 3,                 // outline around the glyphs, so it reads over any tile
    fontSize: 12
  }],
  annotationPosition: mapCommon.TextPosition.BOTTOM,   // FIX: the sample passes the literal 2
  icon: 'humer.png'                 // path relative to resources/rawfile
};

const marker = await this.mapController?.addMarker(markerOptions);
marker?.setClickable(true);
marker?.setTitle('0');              // the shop id - read this back, not getZIndex()
marker?.setInfoWindowAnchor(0.5, 1);
```

**`annotations` is the feature this card exists for.** It attaches styled text
to the pin without a second overlay, and `strokeWidth` is what makes it
legible: a white outline around the glyphs keeps the name readable over dark
satellite tiles and busy street maps alike. `content` is capped at 136 vp and
ellipsised beyond that, so long names need shortening rather than wrapping.

`anchorV: 1` plus `setInfoWindowAnchor(0.5, 1)` put both the icon and its
popup on the coordinate rather than above it - forget either and the pin
floats off its position as you zoom.

**The custom info window — same file** (corrected, see `HW-09-0042`)

```typescript
@Builder
customInfoWindowBuilder($$: map.MarkerDelegate) {
  if ($$.marker) {
    Column() {
      // FIX: the sample switches on $$.marker.getZIndex()
      Image(this.iconForShop(Number($$.marker.getTitle())))
        .width(50)
    }
  }
}
```

The shipped version stacks all three images and toggles `Visibility.None` on
two of them, which works but builds three components per window. The
`MarkerDelegate` gives the builder the tapped marker, so **one builder serves
every marker** - there is no per-marker window to register.

The `if ($$.marker)` guard is required: the delegate is invoked before a
marker is bound.

**Reacting to a tap — same file** (corrected, see `HW-09-0041`, `HW-09-0042`)

```typescript
private mapEventManager?: map.MapEventManager;

this.callback = async (err, mapController) => {
  if (err) {
    hilog.error(0x0000, 'Map', 'map init failed: %{public}s', JSON.stringify(err));
    return;                                            // FIX: the sample has no error branch
  }
  this.mapController = mapController;
  await this.addMarkers();
  // FIX: the sample calls this.mapController.on(...) directly - see HW-09-0046
  this.mapEventManager = this.mapController.getEventManager();
  this.mapEventManager.on('markerClick', this.onMarkerClick);
};

private onMarkerClick = (marker: map.Marker) => {
  marker.setInfoWindowVisible(true);                   // open the custom window
  this.clickedShop = Number(marker.getTitle());        // FIX: the sample reads getZIndex()
};

aboutToDisappear(): void {                             // FIX: absent in the sample
  this.mapEventManager?.off('markerClick');
  this.mapController?.clear();
}
```

**The detail card — `entry/src/main/ets/components/ShopDetail.ets`** (as shipped)

```typescript
@Prop @Watch('onValueChange') clickedShop: number;
@State currentShop: ShopModel = new ShopModel();

onValueChange() {
  this.currentShop = SHOPS.filter((item: ShopModel) => item.index === this.clickedShop)[0];
}
```

`@Prop @Watch` is the right pairing here: the card owns no selection, it
recomputes from one number the page pushes down. Note the missing fallback -
`filter(...)[0]` is `undefined` for any id not in `SHOPS`, which is reachable
today because the id is a `zIndex`.

## Permissions & config

The sample declares **none**. It needs:

```json5
// entry/src/main/module.json5
"metadata": [
  { "name": "client_id", "value": "<your AGC Client ID>" }
],
"requestPermissions": [
  { "name": "ohos.permission.INTERNET",         "usedScene": { "when": "always" } },
  { "name": "ohos.permission.GET_NETWORK_INFO", "usedScene": { "when": "always" } }
]
```

Both are `system_grant`. No location permission is required - the map is
centred on fixed coordinates and my-location is not used.

Marker icons must live in `entry/src/main/resources/rawfile/`; `MarkerOptions.icon`
takes the filename relative to that directory. The sample ships `humer.png`,
`sushis.png` and `fishes.png` there.

Resource directories: `base`, `en_US`, `zh_CN`, `rawfile`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **AGC setup is mandatory** - Map Kit enabled under Manage open capabilities,
  manual signing completed, Client ID in `module.json5`.
- `annotations` requires **API 5.0.3(15)** or later; `iconBuilder`, the
  alternative to a `rawfile` icon, requires 6.0.0(20); `offsetX` / `offsetY`
  require 6.0.2(22); `forceVisible` requires 6.1.0(23).
- An `annotations` array holds **at most 3** entries and at least 1.
- Label `content` is capped at **136 vp** and ellipsised past it; `fontSize`
  is clamped to 0..100 and `strokeWidth` to 0..10.
- Markers collide: when two pins overlap, the map hides one unless
  `forceVisible` is set. That is what `zIndex` is for - which is why using it
  as an identity is a trap.
- The three shops and their coordinates are static; there is no search, and
  the search bar in `TopContent` is decorative.

## Pitfalls

- **`HW-09-0042` — `getZIndex()` is used as the shop id.** The paint order of
  the pins decides which card and which enlarged icon appear, so raising a pin
  above a neighbour silently changes the content. Each marker already carries
  the right id in `setTitle`, and nothing reads it.
- **`HW-09-0041` — `on('markerClick')` is never released** and the page has no
  `aboutToDisappear`. The closure writes `this.clickedShop`, so it keeps the
  destroyed page alive.
- **`HW-09-0045` — no network permission, no `client_id`.** The map cannot
  load from a clean checkout, and the document's five setup steps never
  mention `module.json5`.
- **`HW-09-0043` — `annotationPosition: 2` and `fontStyle: 1` are raw
  integers** for enum-typed fields. `1` is `BOLD`, not `REGULAR`, so every
  label renders bold by accident.
- **`HW-09-0044` — the label text is hardcoded Chinese** although the same
  three names exist as string resources that the detail card uses. The one
  string that is the subject of the document is the one that cannot be
  translated.
- **`HW-09-0046` — this document subscribes on the controller** while
  `TOUR-06` subscribes on the `MapEventManager`, which is what the official
  guide teaches. Two neighbouring documents, two surfaces for the same event.

## References

- `documentation/harmonyos-guides/04_application-services/map-marker.md` - markers, `rawfile` icons, annotations
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-references/03_application-services/map-common.md` - `MarkerOptions`, `Text`, `TextPosition`, `FontStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-references/03_application-services/map-map-marker.md` - `setTitle`, `setInfoWindowAnchor`, `setInfoWindowVisible`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-marker
- `documentation/harmonyos-references/06_application-services/map-map-markerdelegate.md` - the `customInfoWindow` parameter object
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-markerdelegate
- `documentation/harmonyos-references/06_application-services/map-mapcomponent.md` - `MapComponent` and `customInfoWindow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-mapcomponent
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `getEventManager()` and `markerClick`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the two network permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/04_system/configuration_client_id.md` - the `client_id` metadata
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/configuration_client_id
- `TOUR-06` - POI-driven markers and the list binding; `TOUR-12` - adding markers by tap and saving them
