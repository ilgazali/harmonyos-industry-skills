---
id: LIFE-25
title: Show the commute from a listing to work - navi route distance and duration, a static map preview, and a handoff to Petal Maps
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
sample: huawei_industry_tree/02_convenient_life/downloads/CommutingCalculation.zip
kits: ["@kit.MapKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.AbilityKit"]
apis: ["navi.getDrivingRoutes", "navi.getWalkingRoutes", "navi.DrivingRouteParams", "navi.RouteParams", "navi.RouteResult", "navi.Route", "navi.RouteStep", "navi.RouteCoordinate", "staticMap.getMapImage", "staticMap.StaticMapOptions", "staticMap.StaticMapMarker", "staticMap.IconSize", "petalMaps.openMapRoutePlan", "petalMaps.RoutePlanParams", Tabs, TabContent, SubTabBarStyle, onContentWillChange, Navigation, NavPathStack, Scroll, ForEach, Prop, State, StorageProp, "AppStorage.setOrCreate", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "windowStage.getMainWindow", "display.getDefaultDisplaySync", "UIContext.px2vp", "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0198, HW-02-0199, HW-02-0200, HW-02-0201, HW-02-0202, HW-02-0203, HW-02-0204, HW-02-0205, HW-02-0206, HW-02-0207, HW-02-0208, HW-02-0209, HW-02-0210, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **how far and how long from here to there**, rendered inline:
a property listing, a store page, a venue page - anywhere you want to state a
distance and a travel time between two known coordinates and offer a real map
as the next step.

Three Map Kit surfaces, each doing one job, and none of them needing a
permission:

```
navi.getDrivingRoutes(params)   -> RouteResult.routes[0].steps[0].distance (m)
                                                          .durationInTraffic (s)
navi.getWalkingRoutes(params)   -> the same shape, walking
staticMap.getMapImage(options)  -> a PixelMap you put in an Image
petalMaps.openMapRoutePlan(ctx, params) -> hands off to the Petal Maps app
```

**No permission is declared in this project at all** - `module.json5` has no
`requestPermissions` block. Both endpoints are fixed coordinates, so there is no
positioning involved; that is what keeps the feature permission-free. The moment
you replace one endpoint with the user's current position you inherit the
location permissions, and this card stops covering the whole story.

**What it does need is AppGallery Connect setup.** The document is explicit: Map
Service has to be switched on for the application in AGC and the build has to be
manually signed. Without that, every one of the four calls above fails - which
is why the error paths in the pitfalls below matter more here than they would
elsewhere.

Take `LIFE-14`, `LIFE-16` or `LIFE-20` instead if you need an interactive
`MapComponent` rather than a picture.

## Feature checklist

- [ ] Map Service enabled in AGC and the build manually signed.
- [ ] `routes` and `steps` length-checked before indexing (HW-02-0201).
- [ ] A visible failure state when route planning does not return
      (HW-02-0202).
- [ ] `getMapImage` given a `.catch` and a placeholder (HW-02-0203).
- [ ] `openMapRoutePlan` wrapped in try/catch (HW-02-0204).
- [ ] The control the user reads as "route" actually opening Petal Maps
      (HW-02-0206).
- [ ] Avoid areas read inside the `setWindowLayoutFullScreen` promise
      (HW-02-0199), and `off('avoidAreaChange')` in `onWindowStageDestroy`
      (HW-02-0198).
- [ ] The `getMainWindow` callback checking its error parameter
      (HW-02-0200).

## Architecture

Four files, no service layer.

| File | Role |
| --- | --- |
| `pages/HomePage.ets` | `@Entry`. Header image, a five-entry `Tabs` bar whose first tab holds the detail page, and a bottom contact bar. |
| `pages/HousingResourcesPage.ets` | The listing detail. Runs both route lookups in `aboutToAppear`, formats the numbers, and embeds `StaticMap`. |
| `pages/StaticMap.ets` | Fetches the static map image and owns the tap that opens Petal Maps. |
| `constants/StyleConstants.ets` | The two coordinate pairs and the listing attribute list. |

The two endpoints are constants, and this is the piece worth copying:

```ts
export class StyleConstants {
  static readonly LATITUDE: number = 32.02065982629459;        // the workplace
  static readonly LONGITUDE: number = 118.788899213002;
  static readonly HOUSE_LATITUDE: number = 31.974111162916152; // the listing
  static readonly HOUSE_LONGITUDE: number = 118.7731414756938;
}
```

`HousingResourcesPage` holds them as private fields and passes them down to
`StaticMap` through four `@Prop`s, so the static image, the route lookups and
the Petal Maps handoff all describe the same pair of points.

Routing: `main_pages.json` lists only `pages/HomePage`; there is no `routerMap`
and nothing is ever pushed - see pitfall 9.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Enable Map Service before anything else.** From the document: sign in to
   AppGallery Connect, open the project, select the application, go to API
   management, switch on 地图服务 ("Map Service"), then complete manual signing.
   Nothing in this feature works until that is done, and no permission or
   `client_id` metadata is needed on top of it - this project's `module.json5`
   has neither.

2. **Plan the driving route.** Both endpoints go in as `RouteCoordinate`s;
   `origins` is an array, `destination` is a single point:

   ```ts
   let params: navi.DrivingRouteParams = {
     origins: [{ 'latitude': this.houseLatitude, 'longitude': this.houseLongitude }],
     destination: { 'latitude': this.latitude, 'longitude': this.longitude },
     language: 'zh_CN'
   };
   let result = await navi.getDrivingRoutes(params);
   ```

   `language` accepts only `zh_CN` and `en`; omitting it uses the device
   language. There is a second overload, `getDrivingRoutes(context, params)`,
   added in 5.0.0(12) - prefer it when you have a context to hand.

3. **Read the result defensively** (HW-02-0201). `RouteResult.routes` is
   documented to come back empty when no route exists:

   ```ts
   if (!result.routes.length || !result.routes[0].steps.length) {
     this.routeFailed = true;                 // see step 5
     return;
   }
   this.distance = result.routes[0].steps[0].distance as number;         // metres
   this.distanceString = (this.distance / 1000).toFixed(1);              // kilometres
   this.drivingTime = result.routes[0].steps[0].durationInTraffic as number;  // seconds
   this.drivingTimeString = this.secondToTime(this.drivingTime);
   ```

   `Route` itself carries no distance or duration - those live on `RouteStep`,
   in metres and seconds. Every field on `RouteStep` is optional, which is why
   the `as number` casts are there.

4. **Plan the walking route with the base parameter type.** Walking takes
   `navi.RouteParams`, not `DrivingRouteParams`:

   ```ts
   let params: navi.RouteParams = {
     origins: [{ 'latitude': this.houseLatitude, 'longitude': this.houseLongitude }],
     destination: { 'latitude': this.latitude, 'longitude': this.longitude },
     language: 'zh_CN'
   };
   let result = await navi.getWalkingRoutes(params);
   this.walkingTime = result.routes[0].steps[0].durationInTraffic as number;
   ```

   Walking planning only works between points within 150 km of each other in a
   straight line.

5. **Give failure a face** (HW-02-0202). The shipped page catches, logs, and
   interpolates the untouched empty strings into a fixed sentence:

   ```ts
   @State routeFailed: boolean = false;
   ...
   if (this.routeFailed) {
     Text($r('app.string.commute_unavailable'));
   } else {
     Text(`通勤到夫子庙约${this.distanceString}公里`);
     Text(`驾车约${this.drivingTimeString}，步行约${this.walkingTimeString}`);
   }
   ```

   Both sentences and the destination name should come from `string.json` with
   placeholders rather than being baked into the template (HW-02-0207).

6. **Format seconds for display.**

   ```ts
   secondToTime(seconds: number): string {
     let hourUnit = 60 * 60;
     let hour: number = Math.floor(seconds / hourUnit);
     let minute: number = Math.floor((seconds - hour * hourUnit) / 60);
     if (hour > 0) { return `${hour}小时${minute}分钟`; }
     if (minute > 0) { return `${minute}分钟`; }
     return '1分钟';
   }
   ```

   The final branch rounds anything under a minute up to "1 minute" rather than
   showing zero.

7. **Fetch the static map, and handle the rejection** (HW-02-0203):

   ```ts
   let markers: Array<staticMap.StaticMapMarker> = [{
     location: { latitude: this.houseLatitude, longitude: this.houseLongitude },
     defaultIconSize: staticMap.IconSize.NORMAL
   }];
   let option: staticMap.StaticMapOptions = {
     location: { latitude: this.houseLatitude, longitude: this.houseLongitude },
     zoom: 17,          // maximum; the valid range is 2 to 17
     imageWidth: 512,   // with scale 2 the range is (0, 512]
     imageHeight: 512,
     scale: 2,          // the returned image is scale x width by scale x height
     markers: markers
   };
   staticMap.getMapImage(option)
     .then((value) => { this.image = value; })
     .catch((err: BusinessError) => { hilog.error(0x0000, 'StaticMap', `getMapImage failed. Code: ${err.code}`); });
   ```

   The returned `PixelMap` goes straight into an `Image`, so it does **not**
   need a manual `release()` - the image decoding guide states the `Image`
   component manages a PixelMap passed to it.

8. **Hand off to Petal Maps from the control that says so** (HW-02-0206,
   HW-02-0204):

   ```ts
   .onClick(async () => {
     let params: petalMaps.RoutePlanParams = {
       originPosition: { latitude: this.houseLatitude, longitude: this.houseLongitude },
       destinationPosition: { latitude: this.latitude, longitude: this.longitude }
     };
     try {
       await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params);
     } catch (error) {
       let err = error as BusinessError;
       hilog.error(0x0000, 'StaticMap', `openMapRoutePlan failed. Code: ${err.code}`);
     }
   })
   ```

   Put this on the labelled route button, not only on the map picture. The
   shipped sample has it on the picture and leaves the button showing a
   placeholder toast.

9. **Set up the immersive layout in the right order** (HW-02-0199,
   HW-02-0200, HW-02-0198):

   ```ts
   windowStage.getMainWindow((err: BusinessError, data) => {
     if (err.code) {
       hilog.error(DOMAIN, 'testTag', `getMainWindow failed. Cause: ${JSON.stringify(err)}`);
       return;                                   // the shipped sample has no guard
     }
     this.windowClass = data;
     let uiContext = this.windowClass.getUIContext();
     this.windowClass.setWindowLayoutFullScreen(true).then(() => {
       let navArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
       AppStorage.setOrCreate('bottomRectHeight', uiContext.px2vp(navArea.bottomRect.height));
       let systemArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
       AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(systemArea.topRect.height));
       this.windowClass.on('avoidAreaChange', (data) => { /* later changes */ });
     }).catch((err: BusinessError) => { hilog.error(...); });
   });
   ```

   ```ts
   onWindowStageDestroy(): void {
     this.windowClass?.off('avoidAreaChange');
   }
   ```

   The conversion to vp happens at the point of storage here, so the pages read
   plain vp numbers with `@StorageProp`.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`CommutingCalculation.zip#entry/src/main/ets/pages/HousingResourcesPage.ets:43-69` -
the driving lookup, its parameter shape, and the two fields it extracts:

```ts
  async calculateDrivingTime() {
    let params: navi.DrivingRouteParams = {
      // 起点的经纬度
      origins: [{
        'latitude': this.houseLatitude,
        'longitude': this.houseLongitude
      }],
      // 终点的经纬度
      destination: {
        'latitude': this.latitude,
        'longitude': this.longitude
      },
      language: 'zh_CN'
    };

    try {
      let result = await navi.getDrivingRoutes(params);
      this.distance = result.routes[0].steps[0].distance as number;
      this.distanceString = (this.distance / 1000).toFixed(1);
      this.drivingTime = result.routes[0].steps[0].durationInTraffic as number;
      this.drivingTimeString = this.secondToTime(this.drivingTime);
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      hilog.error(0x0000, 'test',
        `error in getting driving routes. Code is ${err.code}, message is ${err.message}`);
    }
  }
```

The two `routes[0].steps[0]` reads are HW-02-0201 - length-check first.

`CommutingCalculation.zip#entry/src/main/ets/pages/HousingResourcesPage.ets:97-111` -
seconds turned into a Chinese duration phrase:

```ts
  secondToTime(seconds: number): string {
    let hourUnit = 60 * 60;
    let hour: number = Math.floor(seconds / hourUnit);
    let minute: number = Math.floor((seconds - hour * hourUnit) / 60);
    let hourStr: string = `${hour.toString()}`;
    let minuteStr: string = `${minute.toString()}`;
    if (hour > 0) {
      return `${hourStr}小时${minuteStr}分钟`;
    }
    if (minute > 0) {
      return `${minuteStr}分钟`;
    } else {
      return '1分钟';
    }
  }
```

`CommutingCalculation.zip#entry/src/main/ets/pages/StaticMap.ets:64-89` - the
static map request, with the marker and the size rules the reference imposes:

```ts
  init() {
    // Set static image marker parameters
    let markers: Array<staticMap.StaticMapMarker> = [{
      location: {
        latitude: this.houseLatitude,
        longitude: this.houseLongitude
      },
      defaultIconSize: staticMap.IconSize.NORMAL
    }];
    // Static image assembly parameters
    let option: staticMap.StaticMapOptions = {
      location: {
        latitude: this.houseLatitude,
        longitude: this.houseLongitude
      },
      zoom: 17,
      imageWidth: 512,
      imageHeight: 512,
      scale: 2,
      markers: markers,
    };
    // Get static image
    staticMap.getMapImage(option).then((value) => {
      this.image = value;
    });
  }
```

The missing `.catch` on the last call is HW-02-0203.

`CommutingCalculation.zip#entry/src/main/ets/pages/StaticMap.ets:41-62` - the
Petal Maps handoff, and the four `@Prop`s that feed it:

```ts
  @Builder
  mapShow() {
    // Display the static image obtained
    Image(this.image)
      .width('100%')
      .height(167)
      .fitOriginalSize(false)
      .objectFit(ImageFit.Cover)
      .onClick(async () => {
        let params: petalMaps.RoutePlanParams = {
          originPosition: {
            latitude: this.houseLatitude,
            longitude: this.houseLongitude
          },
          destinationPosition: {
            latitude: this.latitude,
            longitude: this.longitude
          }
        };
        await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params);
      });
  }
```

`CommutingCalculation.zip#entry/src/main/ets/pages/HousingResourcesPage.ets:215-220` -
the coordinates handed down to the map component as plain props:

```ts
            StaticMap({
              latitude: this.latitude,
              longitude: this.longitude,
              houseLatitude: this.houseLatitude,
              houseLongitude: this.houseLongitude
            });
```

`CommutingCalculation.zip#entry/src/main/ets/entryability/EntryAbility.ets:44-67` -
the immersive layout and the avoid-area reads, in the order the sample ships
them:

```ts
    let windowClass: window.Window | undefined = undefined;
    windowStage.getMainWindow((err: BusinessError, data) => {
      windowClass = data;
      let uiContext: UIContext | null = null;
      uiContext = windowClass.getUIContext();

      // 设置窗口全屏
      let isLayoutFullScreen = true;
      windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(() => {
        hilog.info(0x0000, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
      }).catch((err: BusinessError) => {
        hilog.error(0x0000, 'testTag',
          'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
      });

      let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR; // 以导航条避让为例
      let avoidArea = windowClass.getWindowAvoidArea(type);
      let bottomRectHeight = avoidArea.bottomRect.height; // 获取到导航条区域的高度
      AppStorage.setOrCreate('bottomRectHeight', uiContext.px2vp(bottomRectHeight));

      type = window.AvoidAreaType.TYPE_SYSTEM; // 以状态栏避让为例
      avoidArea = windowClass.getWindowAvoidArea(type);
      let topRectHeight = avoidArea.topRect.height; // 获取状态栏区域高度
      AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(topRectHeight));
```

Three findings live in that block: the unchecked `err` (HW-02-0200), the reads
outside the promise chain (HW-02-0199), and the listener registered on a local
handle just below it (HW-02-0198).

## Permissions & config

**No permissions.**
`CommutingCalculation.zip#entry/src/main/module.json5` has no
`requestPermissions` block, and there is no `client_id` metadata either - Map
Kit has not needed a client ID since HarmonyOS 5.0.2(14). Both endpoints are
constants, so no positioning permission is involved.

What is required instead is account-side configuration, quoted from the
document: '本示例需要开通地图服务，并进行相应配置' ("This sample requires Map
Service to be enabled, with the corresponding configuration") - switch on
地图服务 under API management in AppGallery Connect for the application, confirm
the capability is open, and complete manual signing.

Module configuration:

```json5
"deviceTypes": ["phone", "tablet", "2in1"],
"pages": "$profile:main_pages",
```

No `routerMap` - the project has no navigation routes.

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
"strictMode": { "caseSensitiveCheck": true, "useNormalizedOHMUrl": true }
```

`oh-package.json5` has no runtime dependencies - Map Kit is an SDK kit.

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **Map Service must be enabled in AGC and the build manually signed.** Every
  call in this card fails otherwise, which is why the error paths matter.
- **`RouteResult.routes` can be empty.** The reference states: "Planned routes
  from the departure place to the destination. If no routes are available, an
  empty array will be returned."
- **`Route` carries no distance or duration.** Those are on `RouteStep`, both
  optional: `distance` in metres, `duration` and `durationInTraffic` in seconds.
- **Driving planning returns at most three routes** and accepts at most five
  waypoints.
- **Walking planning is limited to 150 km straight-line** between the two
  points.
- **`origins` holds at most 31 coordinates**, and adjacent points must be more
  than one metre apart. With no waypoints set, the departure and destination
  coordinates must not be identical.
- **`language` accepts only `zh_CN` and `en`.** Omit it to use the device
  language.
- **Static map limits:** `zoom` is an integer from 2 to 17; with `scale: 2` both
  `imageWidth` and `imageHeight` must be in (0, 512], and with `scale: 1` in
  (0, 1024]. The sample sits exactly on the upper bound of both.
- **The static map PixelMap does not need releasing** because it is handed to an
  `Image`, which manages it.
- **`getWindowAvoidArea` returns px.** This sample converts with `px2vp` before
  storing.

## Pitfalls

1. **HW-02-0201 - the route result is indexed without a length check.**
   `HousingResourcesPage.ets:60`, `:62` and `:88` all read
   `result.routes[0].steps[0]`, and neither `routes.length` nor `steps.length`
   is tested anywhere in the ZIP. The reference documents the empty array as the
   normal no-route outcome, so the one case the API tells you to handle is
   handled by accident: the TypeError falls into the surrounding `catch` and is
   logged as a route-planning failure. The document reproduces the same
   unchecked shape in its snippet (`25_commuting_calculation.md:29-31`, `:36`).

2. **HW-02-0202 - a failed lookup renders a sentence with holes.** Both catch
   blocks (`:64-68`, `:90-94`) log and stop, leaving `distanceString`,
   `drivingTimeString` and `walkingTimeString` at their initial `''`. Those go
   straight into the two sentences the page exists to show (`:166`, `:177`), so
   the user reads "About kilometres to Confucius Temple" and "About by car,
   about on foot". Given that the feature needs AGC setup to work at all, this
   is the first-run experience for anyone who has not finished it.

3. **HW-02-0203 - `getMapImage` has no `.catch`.** `StaticMap.ets:86-88` is
   `staticMap.getMapImage(option).then((value) => { this.image = value; });` and
   nothing more. A failure is an unhandled rejection and a blank 167vp strip
   that still takes taps - and that strip is the only control that opens Petal
   Maps.

4. **HW-02-0204 - `openMapRoutePlan` is awaited with nothing catching it.**
   `StaticMap.ets:60` awaits it as the last statement of an async `onClick` with
   no `try`. The document presents the same bare line as its entire second
   implementation step. The route-planning calls in the sibling file both wrap
   their awaits in `try/catch`, so the sample contradicts itself.

5. **HW-02-0206 - the route button and the route action are on different
   controls.** `HousingResourcesPage.ets:185-206` renders a button with a path
   icon and the label `$r('app.string.route')` whose handler shows
   `'此功能仅供展示'` ("This feature is for display only"). The real
   `openMapRoutePlan` call is bound to the static map picture in another file
   (`StaticMap.ets:49-61`), with no affordance marking it as tappable. A reader
   who came for the document's second step presses the only control that says
   route and is told it is a mock.

6. **HW-02-0198 - `on('avoidAreaChange')` is registered on a callback-local
   handle and never unsubscribed.** `EntryAbility.ets:69` subscribes;
   `windowClass` is a `let` declared at `:44` inside `onWindowStageCreate`, so
   even adding `off` to `onWindowStageDestroy` (`:82-85`, which only logs) would
   have nothing to call it on. Store the window on the ability.

7. **HW-02-0199 - the avoid areas are read outside the promise that makes them
   correct.** `EntryAbility.ets:52-57` chains `setWindowLayoutFullScreen`
   properly but its `.then()` only logs; the two `getWindowAvoidArea` calls sit
   after the chain at statement level (`:60`, `:65`). The values are consumed as
   layout offsets by `HomePage.ets:42` and `:143`.

8. **HW-02-0200 - the `getMainWindow` callback ignores its error.**
   `EntryAbility.ets:45-48` assigns `windowClass = data` and calls
   `windowClass.getUIContext()` on the next line without checking `err`. Nine
   lines earlier the same method guards its `loadContent` callback correctly
   with `if (err.code) { ...; return; }`. On the failure path everything the
   callback sets up is lost to an uncaught throw.

9. **HW-02-0205 - the home page's five tabs can never be switched.**
   `HomePage.ets:88-90` attaches `.onContentWillChange(() => { return false; })`
   to the `Tabs`. The reference states plainly that `false` "means that the tab
   cannot switch to the new page and will remain on the current page", and lists
   a tap on the tab bar among the triggers. Four of the five `TabContent` bodies
   are empty anyway (`:60-61`, `:67-68`, `:74-75`, `:81-82`). Nothing in the
   file says the veto is deliberate.

10. **HW-02-0207 - the feature's output sentences are hardcoded literals with
    the landmark baked in.** `HousingResourcesPage.ets:166` hardcodes
    `通勤到夫子庙约${...}公里`, `:177` hardcodes the driving and walking phrase,
    and `secondToTime` hardcodes the unit words (`:104`, `:107`). The name
    夫子庙 appears only in that template - the coordinates it refers to are
    unnamed constants in another file - so changing the destination silently
    leaves the wrong landmark on screen. The same two rows already use
    `$r('app.string.title')` and `$r('app.string.route')` for their labels.

11. **HW-02-0208 - dead scaffolding on the home page.** `HomePage.ets:25`
    creates a `NavPathStack` and `:36` wraps everything in `Navigation`, but
    nothing is ever pushed, there is no `routerMap`, and `main_pages.json` lists
    one page. `:28` declares `windowHeight`, `:32` computes it, and nothing
    reads it. The `Navigation` is the misleading half: the document's project
    tree presents three pages, and only one of them is a page.

12. **HW-02-0209 - the reference-documents section links `navi` to the Petal
    Maps page.** `25_commuting_calculation.md:80` labels the entry
    `navi（路径规划）` and points at `map-petal-maps`, the same URL as the
    petalMaps entry two lines below. The correct URL, `map-navi-api`, is already
    used earlier in the same document at `:15`.

13. **HW-02-0210 - the project tree misspells the detail page.** The tree
    (`:73`) says `housingResourcesPage.ets`; the shipped file and the import in
    `HomePage.ets:20` are `HousingResourcesPage`. The project builds with
    `"caseSensitiveCheck": true`, and the two sibling entries in the same tree
    match the ZIP exactly.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- navi (route planning - `getDrivingRoutes`, `getWalkingRoutes`, `RouteResult`,
  `Route`, `RouteStep`, `RouteParams`, `RouteCoordinate`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-navi-api
- petalMaps (launching Petal Maps):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/map-petal-maps
- staticMap (`getMapImage`, `StaticMapOptions` size and zoom limits):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/map-staticmap
- Map Kit setup in AppGallery Connect:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/map-config-agc
- Signing configuration:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ide-signing
- Tabs (`onContentWillChange` return semantics):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- Image decoding (when a PixelMap needs releasing):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
