---
id: SOCIAL-02
title: Full-screen image preview - a Swiper of photos with pinch, pan and double-tap zoom under one parallel gesture group
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/02_image_preview.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_preview-0000002266277321
sample: huawei_industry_tree/14_social_communication/downloads/imagePreview.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Swiper, disableSwipe, onAnimationEnd, DotIndicator, parallelGesture, GestureGroup, "GestureMode.Exclusive", PinchGesture, PanGesture, TapGesture, "Image.onComplete", objectFit, draggable, Grid, GridItem, Navigation, NavDestination, NavPathStack, pushPath, routerMap, "UIContext.px2vp", "@StorageProp", "window.getWindowAvoidArea", hilog]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0004, HW-14-0086, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a grid of thumbnails has to open into a full-screen
photo viewer** - a moments/feed post, a chat album, a product gallery. The
pattern is a `Swiper` of `Image`s pushed onto a `NavPathStack` as its own
`NavDestination`, with all four zoom and pan gestures hung off a single
`parallelGesture` group on the Swiper itself.

The transferable idea is that **the viewer owns one piece of transform state,
not one per image.** `activeImage` holds scale, offset, and the drag anchor for
whichever page is currently on screen; every other `Image` in the Swiper is
rendered with `scale(null)` and `offset(null)` and therefore costs nothing. On
page change the state is reset and re-seeded from the new image's measured
size. That keeps a nine-image gallery as cheap as a one-image one, and it is
what lets `disableSwipe` be a single boolean.

It generalises to any "flat list, one live item" viewer: a video pager, a
document reader, a card deck. The cost is that the live item and the measured
sizes are kept in two different places, which is exactly where this sample's
one defect lives (`HW-14-0004`).

## Feature checklist

- A 3x3 `Grid` of thumbnails under a user header, inside a feed post.
- Tapping any thumbnail opens a full-screen preview starting on that image.
- The preview pages left/right between all nine images with a dot indicator.
- Pinch scales the active image between 1x and 4x, keeping the visual centre
  stable.
- While zoomed in, dragging pans the image, clamped so it can never be dragged
  past its own edges.
- Double-tap toggles between 1x and 2x; at 1x it also clears any pan offset.
- Single-tap closes the preview and returns to the feed.
- Paging is disabled while the image is zoomed, and re-enabled once the pan
  reaches the horizontal edge or the image returns to 1x.

## Architecture

One `entry` module. The feed page and the preview page are separate routes,
joined by a `routerMap` profile rather than by an import.

```
entry/src/main/ets
├── components
│  ├── ImageInfo.ets           the 3x3 thumbnail Grid; pushes the preview route
│  ├── InteractiveInfo.ets     like / comment strip under the post
│  └── UserInfo.ets            avatar, nickname, post text
├── constants/StyleConstants.ets   numeric literals (ZERO, ONE, gesture finger counts)
├── entryability/EntryAbility.ets  full-screen layout, avoid areas -> AppStorage
├── pages
│  ├── ImagePreview.ets        the whole viewer: ImageViewerComponent + its @Builder route entry
│  └── MainPage.ets            @Entry, the Navigation host and the feed post
└── viewmodel/ImageData.ets    TopItem / BottomItem classes and the nine-image list
```

The documented tree matches the zip. (Four other trees in this industry do
not - see `HW-14-0001`.)

**The design decision worth copying** is that the preview is a *route*, not a
dialog or an overlay. `MainPage` publishes its `NavPathStack` into `AppStorage`
under `'pathStack'` in `aboutToAppear`; `ImageInfo` reads it back in its own
`aboutToAppear` and calls `pushPath({ name: 'imagePreviewPage', param: ... })`.
`route_map.json` maps that name to `ImagePreviewBuilder`, the `@Builder`
function exported at the bottom of `ImagePreview.ets`:

```typescript
@Builder
export function ImagePreviewBuilder(_: string, params: ImagePreviewParam) {
  ImageViewerComponent({ imageList: params.imageList, active: params.active });
}
```

The payoff is that the viewer is loaded lazily by name and gets the system
back gesture for free - `pathStack.pop()` and the hardware back do the same
thing. The cost is the `AppStorage` handoff, which is untyped at the boundary
(`AppStorage.get('pathStack') as NavPathStack`) and will silently produce an
empty stack if the two components ever mount in the other order.

## Implementation steps

1. **Host the feed in a `Navigation(this.pathStack)`** and publish the stack:
   `AppStorage.setOrCreate('pathStack', this.pathStack)`.
2. **Declare the preview in `resources/base/profile/route_map.json`** with
   `pageSourceFile` and `buildFunction`, and point `module.json5` at it with
   `"routerMap": "$profile:route_map"`.
3. **Push the whole list plus the tapped index**, not just the one image - the
   Swiper needs the neighbours to page to.
4. **Measure each image in `onComplete`** and store `px2vp`-converted width and
   height into `imageListSize[i]`; only copy into `activeImage` when `i` is the
   active index.
5. **Bind the gestures with `parallelGesture`, not `gesture`**, so the Swiper's
   own built-in swipe still runs alongside them.
6. **Group them `GestureMode.Exclusive`** so a pinch and a pan can never both
   win on the same touch sequence.
7. **Drive `disableSwipe` from the scale**, and clear it again when a pan
   reaches the horizontal limit so the user can page on from a zoomed image.
8. **Reset the transform on page change in `onAnimationEnd`, and guard the size
   lookup** - `imageListSize[index]` is only populated once that image has
   finished loading (`HW-14-0004`).

## Verified snippets

All snippets are from `imagePreview.zip`. Corrected forms are marked.

**The thumbnail grid and the route push - `entry/src/main/ets/components/ImageInfo.ets`** (as shipped)

```typescript
@Component
export struct ImageInfo {
  pathStack: NavPathStack = new NavPathStack();
  @Prop imageList: (string | Resource)[];

  aboutToAppear(): void {
    this.pathStack = AppStorage.get('pathStack') as NavPathStack;
  }

  build() {
    Grid() {
      ForEach(this.imageList, (item: string | Resource, index: number) => {
        GridItem() {
          Image(item)
            .width($r('app.float.index_img_width'))
            .aspectRatio(StyleConstants.INDEX_IMG_ASPECTRATIO)
            .borderRadius($r('app.float.borderRadius_sixteen'))
            .onClick(() => {
              this.pathStack.pushPath({
                name: 'imagePreviewPage',
                param: { imageList: this.imageList, active: index } as ImagePreviewParam
              });
            });
        }
        .width($r('app.float.index_img_width'))
        .aspectRatio(StyleConstants.INDEX_IMG_ASPECTRATIO);
      });
    }
    .columnsTemplate('1fr 1fr 1fr')
    .rowsTemplate('1fr 1fr 1fr')
  }
}
```

**Both `columnsTemplate` and `rowsTemplate` are set**, which pins the Grid to
exactly nine cells and turns off scrolling - the correct choice for a fixed
九宫格 (nine-square grid) post thumbnail block. `aspectRatio(1)` on both the
`GridItem` and the `Image` is what keeps the cells square when the row height
is derived from the container rather than the content.

The route parameter is an interface, not a class:
`ImagePreviewParam { imageList; active }`. Passing the whole list is
deliberate - the preview must be able to page to images the user has not
tapped.

**Measuring each image as it loads - `entry/src/main/ets/pages/ImagePreview.ets`** (as shipped)

```typescript
Swiper() {
  ForEach(this.imageList, (item: string | Resource, i: number) => {
    Image(item)
      .objectFit(ImageFit.Contain)
      .draggable(false)
      .scale(this.active === i ? { x: this.activeImage.scale, y: this.activeImage.scale } : null)
      .offset(this.active === i ? { x: this.activeImage.offsetX, y: this.activeImage.offsetY } : null)
      .onComplete(event => {
        if (event?.loadingStatus) {
          this.imageListSize[i] = {
            width: this.getUIContext().px2vp(Number(event.contentWidth)),
            height: this.getUIContext().px2vp(Number(event.contentHeight))
          };
          if (this.active === i) {
            this.activeImage.width = this.imageListSize[i].width;
            this.activeImage.height = this.imageListSize[i].height;
          }
        }
      });
  });
}
.disableSwipe(this.disabledSwipe)
.index(this.active)
.itemSpace(StyleConstants.SWIPER_ITEM_SPACE)
.onAreaChange((_, n) => {
  this.containerWidth = Number(n.width);
  this.containerHeight = Number(n.height);
});
```

**Three lines carry the design.** `draggable(false)` is mandatory: without it a
long-press-and-move on an `Image` starts a system drag and steals the pan
gesture. The ternaries on `scale` and `offset` mean only the active page is
transformed, so paging away from a zoomed image cannot leave a stale transform
behind on a neighbour. And `onComplete` is the *only* source of real image
dimensions - `contentWidth`/`contentHeight` arrive in px and must be converted
with `px2vp` before they can be compared against `containerWidth`, which
`onAreaChange` reports in vp.

`imageListSize` is a plain private array, not `@State`. That is correct: it is
read inside gesture callbacks, never rendered, so making it observable would
only cost re-renders.

**The four gestures - same file** (as shipped, trimmed)

```typescript
.parallelGesture(
  GestureGroup(GestureMode.Exclusive,
    PinchGesture({ fingers: StyleConstants.PINCHGESTURE_FINGERS_COUNT })
      .onActionStart(() => {
        this.defaultScale = this.activeImage.scale;
      })
      .onActionUpdate((event) => {
        let scale = event.scale * this.defaultScale;
        if (scale <= 4 && scale >= 1) {
          this.activeImage.offsetX = this.activeImage.offsetX / (this.activeImage.scale - 1) * (scale - 1) || 0;
          this.activeImage.offsetY = this.activeImage.offsetY / (this.activeImage.scale - 1) * (scale - 1) || 0;
          this.activeImage.scale = scale;
        }
        this.disabledSwipe = this.activeImage.scale > 1;
      }),
    PanGesture()
      .onActionStart(event => {
        if (!event.fingerList?.[0]) {
          return;
        }
        this.activeImage.dragOffsetX = event.fingerList[0].globalX;
        this.activeImage.dragOffsetY = event.fingerList[0].globalY;
      })
      .onActionUpdate((event) => {
        if (this.activeImage.scale === 1 || !event.fingerList?.[0]) {
          return;
        }
        let offsetX = event.fingerList[0].globalX - this.activeImage.dragOffsetX +
        this.activeImage.offsetStartX;
        if (this.activeImage.width * this.activeImage.scale > this.containerWidth &&
          (this.activeImage.width * this.activeImage.scale - this.containerWidth) / 2 >=
          Math.abs(offsetX)) {
          this.activeImage.offsetX = offsetX;
        }
        if ((this.activeImage.width * this.activeImage.scale - this.containerWidth) / 2 < Math.abs(offsetX)) {
          this.disabledSwipe = false;
        }
      })
      .onActionEnd(() => {
        this.activeImage.offsetStartX = this.activeImage.offsetX;
        this.activeImage.offsetStartY = this.activeImage.offsetY;
      }),
    //双击手势
    TapGesture({ count: StyleConstants.TAPGESTURE_FINGERS_COUNT })
      .onAction(() => {
        if (this.activeImage.scale > 1) {
          this.activeImage.scale = 1;
          this.activeImage.offsetX = 0;
          this.activeImage.offsetY = 0;
          this.activeImage.offsetStartX = 0;
          this.activeImage.offsetStartY = 0;
          this.disabledSwipe = false;
        } else {
          this.activeImage.scale = 2;
          this.disabledSwipe = true;
        }
      }),
    //单击手势
    TapGesture({ count: StyleConstants.ONE })
      .onAction(() => {
        this.pathStack?.pop();
      }),
  )
)
```

**`parallelGesture` rather than `gesture` is the load-bearing choice.** The
Swiper already consumes horizontal drags for paging; a plain `.gesture()` would
compete with that and one of the two would be lost. `parallelGesture` lets the
group run alongside the component's built-in recogniser, and `disableSwipe`
then arbitrates between them declaratively instead of by gesture priority.

Inside the group, `GestureMode.Exclusive` means the first gesture to be
recognised wins the whole sequence - two fingers can never simultaneously pinch
and pan. Note that the two `TapGesture`s are declared **double-tap first**:
within an exclusive group the single-tap would otherwise fire before the system
could tell the two apart.

The offset rescale in `onActionUpdate` is the "keep the visual centre stable"
trick: the current offset is expressed as a fraction of the current overscale
(`scale - 1`) and re-applied at the new one. The `|| 0` tail catches the
`0/0 = NaN` at exactly `scale === 1`. The pan clamp is the same expression in
reverse - half the overflow `(w * scale - containerWidth) / 2` is the furthest
the image may travel before its edge would enter the viewport.

**Resetting on page change - same file** (corrected, see `HW-14-0004`)

```typescript
.onAnimationEnd((index) => {
  if (index !== this.active) {
    this.activeImage = {
      width: 0,
      height: 0,
      scale: 1,
      offsetX: 0,
      offsetY: 0,
      offsetStartX: 0,
      offsetStartY: 0,
      dragOffsetX: 0,
      dragOffsetY: 0,
    };
  }
  this.active = index;
  this.disabledSwipe = this.activeImage.scale > 1;
  if (!this.imageListSize[index]) {   // FIX: absent in the sample
    return;                           // onComplete will seed it when the load finishes
  }
  this.activeImage.width = this.imageListSize[index].width;
  this.activeImage.height = this.imageListSize[index].height;
})
```

`onAnimationEnd` fires when the paging animation settles, which is not the same
moment as the new image finishing its decode. `imageListSize` is a **sparse**
array filled one entry at a time by nine independent `onComplete` callbacks, so
`imageListSize[index]` is `undefined` for any image whose load has not
completed - swiping quickly, or over a large remote image, dereferences it and
throws a `TypeError`. The guard is enough because `onComplete` already copies
into `activeImage` when `this.active === i`; skipping the assignment here
simply defers the seeding to the load callback.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - all nine images are
`$r('app.media.imgN')` resources. That also keeps this sample clear of
`HW-14-0003`, the copy-pasted permission block that leaves unused declarations
in four of this industry's other samples.

`deviceTypes` is `["phone"]` only. `EntryAbility` sets
`ColorMode.COLOR_MODE_LIGHT` unconditionally at `onCreate`, then goes
full-screen and publishes `topRectHeight` / `bottomRectHeight` into
`AppStorage`; `MainPage` reads them back with `@StorageProp` and converts with
`px2vp` at the point of use.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The scale range is hardcoded `>= 1 && <= 4` in the pinch handler. There is no
  spring-back: a pinch beyond either bound is simply ignored mid-gesture rather
  than rubber-banding.
- The pan clamp uses the *measured* image size, so a preview opened on an image
  that has not loaded yet pans against `width = 0` and refuses to move until
  `onComplete` lands.
- `avoidAreaChange` is registered in `onWindowStageCreate` and never released;
  `setWindowLayoutFullScreen` returns a promise that is logged but whose
  rejection does not change behaviour. Standard boilerplate across this corpus.
- The nine images are bundled resources. There is no loading placeholder, no
  error image and no download path - a real feed needs `alt` on the `Image` at
  minimum.

## Pitfalls

- **`HW-14-0004`** (B/low, probable): `onAnimationEnd` reads
  `this.imageListSize[index].width` for the page just swiped to, but that entry
  is only written by that image's own `onComplete`; a fast swipe onto a
  not-yet-decoded image throws a `TypeError`. The same code appears verbatim in
  the `SendOriginalImage` sample's `ImageViewerComponent.ets`, so it is a
  template defect rather than a one-off. Fix: guard with
  `if (!this.imageListSize[index]) return;` and let `onComplete` seed the size.
- **`HW-14-0001`** (E/low, confirmed): the industry-wide 工程目录 mismatch. This
  sample's tree is accurate, but four sibling docs list files their zips do not
  contain - do not trust a documented tree in this industry without checking the
  zip.
- The `AppStorage.get('pathStack') as NavPathStack` handoff between `MainPage`
  and `ImageInfo` is an unchecked cast across a mount-order dependency. It works
  because `MainPage` is the `@Entry`, but a refactor that reuses `ImageInfo`
  elsewhere gets a silently empty stack and dead thumbnails.
- `@Prop active` is written back from `onAnimationEnd` (`this.active = index`).
  `@Prop` is a one-way copy, so this mutates the local copy only - fine here
  because the parent never re-renders, but it will not survive being lifted
  into a stateful host.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `onComplete`, `contentWidth`/`contentHeight`, `objectFit`, `draggable`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureGroup` and `GestureMode.Exclusive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `disableSwipe`, `onAnimationEnd`, `DotIndicator`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `columnsTemplate` / `rowsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - the preview route
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `routerMap` and `pushPath`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `SOCIAL-01` - the industry index this card hangs off
