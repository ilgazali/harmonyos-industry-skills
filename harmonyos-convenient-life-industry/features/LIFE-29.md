---
id: LIFE-29
title: Draw a draggable coverage circle on the map - marker drag and long-press distance driving a circle and a point annotation
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
sample: huawei_industry_tree/02_convenient_life/downloads/SetCoverageArea.zip
kits: ["@kit.MapKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [MapComponent, "map.MapComponentController", "map.MapEventManager", "mapController.getEventManager", "mapController.addMarker", "mapController.addCircle", "mapController.addPointAnnotation", "map.Marker", "marker.getPosition", "map.MapCircle", "mapCircle.remove", "map.PointAnnotation", "pointAnnotation.remove", "map.calculateDistance", "mapCommon.MapOptions", "mapCommon.MarkerOptions", "mapCommon.MapCircleOptions", "mapCommon.PointAnnotationParams", "mapCommon.CollisionRule", "mapCommon.TextPosition", "mapCommon.FontStyle", "mapCommon.LatLng", "on('markerDragStart')", "on('markerDrag')", "on('mapLongClick')", "off"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-02-0247, HW-02-0248, HW-02-0249, HW-02-0250, HW-02-0251, HW-02-0252, HW-02-0253, HW-02-0269, HW-02-0270]
status: verified-with-fixes
---

## When to use

Load this card for **a radius on a map that the user can move and resize**:
delivery range, service coverage, a geofence preview, "how far is this from
here". Three overlays and two gestures:

```
addMarker(draggable: true)      the centre the user can pick up
addCircle({ center, radius })   the coverage area
addPointAnnotation({ titles })  the label beside the centre

on('markerDrag')   -> redraw circle + annotation at the marker's new position
on('mapLongClick') -> radius = calculateDistance(marker, pressed point), redraw
```

The part worth taking is that **there is no circle-resize API**. `MapCircle` has
no setter for its centre or radius in this sample's usage - every change is
`remove()` the old overlay and `addCircle()` a new one. Same for the point
annotation. That is why the redraw discipline in pitfall 2 matters more here
than it would elsewhere: on a drag, that remove-and-add pair runs on every
frame.

`map.calculateDistance(latLngA, latLngB)` returns metres and is what turns a
long press into a radius - a synchronous call with no service round trip.

Take `LIFE-14`, `LIFE-16` or `LIFE-20` for the other `MapComponent` patterns in
this industry; `LIFE-25` if you want a static picture rather than an
interactive map.

## Feature checklist

- [ ] `ohos.permission.INTERNET` declared; Map Service enabled in AGC.
- [ ] All listeners registered through **one** object - the `MapEventManager`
      (HW-02-0249).
- [ ] Every listener unregistered in `aboutToDisappear` (HW-02-0247).
- [ ] The redraw serialised so overlapping frames cannot orphan overlays
      (HW-02-0248).
- [ ] `remove()` inside the same guard as the `add` call (HW-02-0248).
- [ ] Log messages naming the operation they guard (HW-02-0250).

## Architecture

One page, one file. There is no service layer and no model.

| File | Role |
| --- | --- |
| `pages/Index.ets` | `@Entry`. Map options, the map callback that wires everything, and the two redraw methods. |
| `entryability/EntryAbility.ets` | Loads the page. Nothing map-related. |

All state is private and undecorated, because nothing in the ArkTS tree renders
from it - the map owns the pixels:

```ts
private mapOptions?: mapCommon.MapOptions;
private mapController?: map.MapComponentController;
private callback?: AsyncCallback<map.MapComponentController>;
private mapEventManager?: map.MapEventManager;
private pointAnnotation?: map.PointAnnotation;
private marker?: map.Marker;
private mapCircle?: map.MapCircle;
private radius: number = 500;
private markerLatLng: mapCommon.LatLng = { latitude: 0, longitude: 0 };
private longClickLatLng: mapCommon.LatLng = { latitude: 0, longitude: 0 };
```

The whole build method is a `MapComponent` fed the options and the callback:

```ts
Stack({ alignContent: Alignment.End }) {
  Column() {
    MapComponent({ mapOptions: this.mapOptions, mapCallback: this.callback })
      .width('100%').height('100%');
  }.width('100%')
}.height('100%')
```

Everything else happens inside `this.callback`, which the component invokes once
the map is ready. That callback is where the marker is added, the first circle
and annotation are drawn, and the three listeners are attached.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Declare the network permission.** `INTERNET` is `system_grant`, so no
   runtime request and - per the module configuration reference - no `reason`
   is required, since `reason` "must be specified for a user_grant or
   manual_settings permission":

   ```json5
   "requestPermissions": [
     { "name": "ohos.permission.INTERNET" }
   ]
   ```

   Map Service must also be enabled for the application in AppGallery Connect.

2. **Set the initial camera and hold the callback.**

   ```ts
   this.mapOptions = {
     position: {
       target: { latitude: 31.984410259206815, longitude: 118.76625379397866 },
       zoom: 15
     }
   };
   this.callback = async (err, mapController) => {
     if (err) {
       console.error(`地图初始化失败, code is：${err.code}, message is ${err.message}`);
       return;
     }
     this.mapController = mapController;
     this.mapEventManager = this.mapController.getEventManager();
     // steps 3 to 6
   };
   ```

3. **Add the draggable marker.** `draggable: true` is what makes the drag
   events fire at all:

   ```ts
   let markerOptions: mapCommon.MarkerOptions = {
     position: { latitude: 31.984410259206815, longitude: 118.76625379397866 },
     icon: $r('app.media.locate'),
     rotation: 0, visible: true, zIndex: 0, alpha: 1,
     anchorU: 0.5, anchorV: 0.5,      // centre the icon on the point
     clickable: true, draggable: true, flat: false
   };
   this.marker = await this.mapController.addMarker(markerOptions);
   this.markerLatLng.latitude = this.marker.getPosition().latitude;
   this.markerLatLng.longitude = this.marker.getPosition().longitude;
   ```

4. **Draw the circle, serialising the redraw** (HW-02-0248). The shipped
   version removes before it adds, with an await in between:

   ```ts
   private redrawing = false;

   async changeRadius(latitude: number, longitude: number) {
     if (this.redrawing) { return; }
     this.redrawing = true;
     const previous = this.mapCircle;
     try {
       this.mapCircle = await this.mapController?.addCircle({
         center: { latitude, longitude },
         radius: this.radius,
         clickable: true,
         fillColor: 0x260A59F7,      // ARGB: 0x26 alpha over 0A59F7
         strokeColor: 0xFF0A59F7,
         strokeWidth: 5,
         visible: true,
         zIndex: 15
       });
       previous?.remove();            // remove only after the replacement exists
     } catch (error) {
       console.error(`add circle error, code: ${error.code}, message: ${error.message}`);
     } finally {
       this.redrawing = false;
     }
   }
   ```

   Colours are ARGB integers, so `0x260A59F7` is the brand blue at about 15%
   alpha for the fill and `0xFF0A59F7` fully opaque for the stroke.

5. **Draw the label beside the centre.** `PointAnnotationParams` is the richest
   options object in this card:

   ```ts
   let pointAnnotationOptions: mapCommon.PointAnnotationParams = {
     position: { latitude, longitude },
     repeatable: true,
     collisionRule: mapCommon.CollisionRule.NAME,      // collide on the label text
     textPosition: mapCommon.TextPosition.RIGHT,
     titles: [{
       content: '半径' + radius.toString() + '米',
       color: 0xFF000000,
       fontSize: 15,
       strokeColor: 0xFFFFFFFF,     // white outline keeps it readable over the map
       strokeWidth: 2,
       fontStyle: mapCommon.FontStyle.ITALIC
     }],
     icon: $r('app.media.locate'),
     showIcon: true,
     anchorU: 0.5, anchorV: 0.5,
     forceVisible: false,           // let the collision rule hide it when crowded
     priority: 3,
     minZoom: 2, maxZoom: 20,       // only rendered inside this zoom range
     visible: true,
     zIndex: 10
   };
   ```

   Apply the same serialised remove-after-add treatment as the circle.

6. **Wire the three listeners through the event manager** (HW-02-0249,
   HW-02-0251). The marker guide documents all the drag events on
   `MapEventManager`, and one callback can serve both drag events:

   ```ts
   let onDrag = (marker: map.Marker) => {
     let p = marker.getPosition();
     this.markerLatLng = p;
     this.changeRadius(p.latitude, p.longitude);
     this.movePointAnnotation(p.latitude, p.longitude, this.radius);
   };
   this.mapEventManager.on('markerDragStart', onDrag);
   this.mapEventManager.on('markerDrag', onDrag);

   this.mapEventManager.on('mapLongClick', async (position: mapCommon.LatLng) => {
     this.longClickLatLng = position;
     this.radius = Math.floor(map.calculateDistance(this.markerLatLng, this.longClickLatLng));
     this.changeRadius(this.markerLatLng.latitude, this.markerLatLng.longitude);
     this.movePointAnnotation(this.markerLatLng.latitude, this.markerLatLng.longitude, this.radius);
   });
   ```

   The long press keeps the **centre** where it is and only changes the radius -
   the distance from the marker to the pressed point, floored to whole metres.

7. **Unregister on the way out** (HW-02-0247). The sample has no
   `aboutToDisappear` at all:

   ```ts
   aboutToDisappear(): void {
     this.mapEventManager?.off('markerDragStart');
     this.mapEventManager?.off('markerDrag');
     this.mapEventManager?.off('mapLongClick');
   }
   ```

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`SetCoverageArea.zip#entry/src/main/ets/pages/Index.ets:35-65` - the map
callback, from controller to first draw:

```ts
    this.callback = async (err, mapController) => {
      if (!err) {
        this.mapController = mapController;
        this.mapEventManager = this.mapController.getEventManager();

        // Marker初始化参数
        let markerOptions: mapCommon.MarkerOptions = {
          position: {
            latitude: 31.984410259206815,
            longitude: 118.76625379397866
          },
          icon: $r('app.media.locate'),
          rotation: 0,
          visible: true,
          zIndex: 0,
          alpha: 1,
          anchorU: 0.5,
          anchorV: 0.5,
          clickable: true,
          draggable: true,
          flat: false,
        };
        // 创建Marker
        this.marker = await this.mapController.addMarker(markerOptions);
        // 获取当前marker点的坐标
        this.markerLatLng.latitude = this.marker.getPosition().latitude;
        this.markerLatLng.longitude = this.marker.getPosition().longitude;

        // 首次渲染点注释和圆形
        this.changeRadius(this.marker.getPosition().latitude, this.marker.getPosition().longitude);
        this.movePointAnnotation(this.marker.getPosition().latitude, this.marker.getPosition().longitude, this.radius);
```

`SetCoverageArea.zip#entry/src/main/ets/pages/Index.ets:84-94` - the long press
turned into a radius, which is the cleanest idea in the sample:

```ts
        // 长按地图重新画圆形方法
        let mapLongClickcallback = async (position: mapCommon.LatLng) => {
          this.longClickLatLng.latitude = position.latitude;
          this.longClickLatLng.longitude = position.longitude;

          this.radius = Math.floor(map.calculateDistance(this.markerLatLng, this.longClickLatLng));
          this.changeRadius(this.markerLatLng.latitude, this.markerLatLng.longitude);
          this.movePointAnnotation(this.markerLatLng.latitude, this.markerLatLng.longitude, this.radius);
        };

        this.mapEventManager.on('mapLongClick', mapLongClickcallback);
```

`SetCoverageArea.zip#entry/src/main/ets/pages/Index.ets:103-124` - the circle
redraw as shipped:

```ts
  async changeRadius(latitude: number, longitude: number) {
    this.mapCircle?.remove();
    let mapCircleOptions: mapCommon.MapCircleOptions = {
      center: {
        latitude: latitude,
        longitude: longitude
      },
      radius: this.radius,
      clickable: true,
      fillColor: 0x260A59F7,
      strokeColor: 0xFF0A59F7,
      strokeWidth: 5,
      visible: true,
      zIndex: 15
    };

    try {
      this.mapCircle = await this.mapController?.addCircle(mapCircleOptions);
    } catch (error) {
      console.error(`Marker icon builder test error, code: ${error.code}, message: ${error.message}`);
    }
  }
```

The `remove()` on the second line sits outside the `try` and before the `await`
- that is HW-02-0248 - and the catch message is HW-02-0250.

`SetCoverageArea.zip#entry/src/main/ets/pages/Index.ets:128-154` - the full
point-annotation options, which is the reference-quality part of this sample:

```ts
    let pointAnnotationOptions: mapCommon.PointAnnotationParams = {
      position: {
        latitude: latitude,
        longitude: longitude
      },
      repeatable: true,
      collisionRule: mapCommon.CollisionRule.NAME,
      textPosition: mapCommon.TextPosition.RIGHT,
      titles: [{
        content: '半径' + radius.toString() + '米',
        color: 0xFF000000,
        fontSize: 15,
        strokeColor: 0xFFFFFFFF,
        strokeWidth: 2,
        fontStyle: mapCommon.FontStyle.ITALIC
      }],
      icon: $r('app.media.locate'),
      showIcon: true,
      anchorU: 0.5,
      anchorV: 0.5,
      forceVisible: false,
      priority: 3,
      minZoom: 2,
      maxZoom: 20,
      visible: true,
      zIndex: 10
    };
```

`SetCoverageArea.zip#entry/src/main/ets/pages/Index.ets:162-173` - the entire
build method:

```ts
  build() {
    Stack({ alignContent: Alignment.End }) {
      Column() {
        MapComponent({
          mapOptions: this.mapOptions,
          mapCallback: this.callback,
        })
          .width('100%')
          .height('100%');
      }.width('100%')
    }.height('100%')
  }
```

## Permissions & config

One system-grant permission:

```json5
// entry/src/main/module.json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
  }
]
```

`ohos.permission.INTERNET` is permission level normal, authorization mode
**system_grant** - so there is no runtime request anywhere in the sample, and
the missing `reason` and `usedScene` are correct: the module configuration
reference says `reason` "must be specified for a user_grant or manual_settings
permission".

Account-side setup, from the document: '使用地图服务，需要先开通地图服务'
("to use Map Service you must first enable Map Service") in AppGallery Connect.
There is no `client_id` metadata in `module.json5` - Map Kit has not needed one
since HarmonyOS 5.0.2(14).

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **Map Service must be enabled in AGC** for the application.
- **`draggable: true` on the marker** is the precondition for any drag event.
- **Marker drag events belong to `MapEventManager`.** The marker guide's steps
  and example both use `mapEventManager.on('markerDragStart' | 'markerDrag' |
  'markerDragEnd', ...)`.
- **Overlays are replaced, not edited.** Changing the circle's centre or radius
  means `remove()` plus a fresh `addCircle()`, and the same for the point
  annotation.
- **`markerDrag` fires per frame** while the marker is held, so anything it
  calls runs at frame rate.
- **`map.calculateDistance` returns metres** and is synchronous.
- **Colours are ARGB integers** - `0xFF0A59F7` is opaque, `0x260A59F7` about
  15% alpha.
- **`minZoom` and `maxZoom` on the annotation** bound the zoom range in which it
  is drawn; `collisionRule` and `forceVisible` decide what happens when labels
  overlap.
- **`INTERNET` is system_grant** - declared, never requested.

## Pitfalls

1. **HW-02-0247 - three listeners, no teardown of any kind.**
   `Index.ets:67`, `:76` and `:94` register `markerDragStart`, `markerDrag` and
   `mapLongClick`. The page has no `aboutToDisappear` - `aboutToAppear` at `:24`
   is its only lifecycle method - and `off(` appears nowhere in the ZIP. Each
   callback closes over the page and its overlays, so the whole graph stays
   reachable from the map's listener registry. The sibling map sample in this
   industry shows the fix:
   `ConfirmDirectionInMap.zip#entry/src/main/ets/utils/MapUtil.ets:151-157`
   calls `mapEventManager.off(...)` for every event it registered.

2. **HW-02-0248 - the redraw pair is unawaited and races itself.**
   `changeRadius` (`:103-124`) reads `this.mapCircle` for its `remove()` and
   writes the same field after an `await`; `movePointAnnotation` (`:126-160`)
   has the identical shape. Both are called without `await` from the per-frame
   drag handler (`:77`, `:81`). A second call entering mid-await removes the
   already-removed overlay and then overwrites the field, so whatever the first
   call finally assigns stays on the map, unreferenced and unremovable. The
   `remove()` calls at `:104` and `:127` also sit *outside* the `try` that wraps
   only the `add`, so the double removal throws where nothing catches it.

3. **HW-02-0249 - the drag events are registered on the wrong object.**
   `:38` obtains the event manager, `:67` and `:76` then use
   `this.mapController.on(...)` for the two drag events, and `:94` uses
   `this.mapEventManager.on(...)` for the map event. The marker guide documents
   all three drag events on `MapEventManager` and its example is explicit -
   `this.mapEventManager.on("markerDrag", markerDragCallback);`. Beyond the
   inconsistency, it makes the missing teardown ambiguous: which object would
   you call `off` on?

4. **HW-02-0251 - the two drag handlers are byte-identical.** `:67-73` and
   `:76-82` differ only in the event name, comment included, and each calls
   `marker.getPosition()` six times for one position - on the hottest path in
   the sample. Extract one callback, read the position once, register it twice.

5. **HW-02-0250 - the circle failure is logged as a marker icon builder test
   error.** `:122` logs
   `` `Marker icon builder test error, code: ${error.code} ...` `` from the catch
   around `addCircle`. There is no marker, no icon and no builder in that block,
   and the annotation handler four dozen lines below gets it right (`:158`,
   `'add point annotation error'`). The document reproduces the wrong message at
   `29_set_coverage_area.md:67`.

6. **HW-02-0252 - the radius label is a concatenated Chinese literal.**
   `:137` builds `'半径' + radius.toString() + '米'` and `:97` logs
   `地图初始化失败`. Grepping the ZIP for `app.string` returns nothing, while
   `$r('app.media.locate')` is used for the icon at `:46` and `:144` - so the
   sample uses the resource system for media and not for the one string the
   feature renders.

7. **HW-02-0253 - the document's drag snippet drops the receiver.**
   `29_set_coverage_area.md:114` calls `changeRadius(...)` with no `this.`,
   two lines above a correctly written `this.movePointAnnotation(...)`; the
   shipped code has `this.changeRadius(...)` at `Index.ets:77`. The step 2
   snippet also shows `fillColor: 0x4D0A59F7` where the sample uses
   `0x260A59F7` (`Index.ets:112`), a visibly more opaque overlay than the
   screenshot above it.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Drawing markers on a map (drag events on `MapEventManager`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-marker
- Drawing circles on a map:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-circle
- Drawing point annotations on a map:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-annotation
- MapComponentController (`addCircle`, `addMarker`, `addPointAnnotation`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/map-map-mapcomponentcontroller
- MapEventManager (`on` / `off`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-mapeventmanager
- Map Kit setup in AppGallery Connect:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-config-agc
- ohos.permission.INTERNET:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all#ohospermissioninternet
- Module configuration file (when `reason` is mandatory):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
