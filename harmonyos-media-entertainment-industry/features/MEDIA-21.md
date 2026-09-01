---
id: MEDIA-21
title: Multi-camera video wall - a Grid of autoplaying Video components where tapping a secondary angle promotes it to the main one
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/21_multi_camera_video.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_camera_video-0000002297933590
sample: huawei_industry_tree/13_media_entertainment/downloads/MultiCameraVideo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Video, VideoController, Grid, GridLayoutOptions, GridItem, ForEach, "@StorageLink", "@Watch", "@StorageProp", aspectRatio, "window.getWindowAvoidArea", "AppStorage.setOrCreate", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0054, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a page must show **several video streams at once and let
the user pick which one is large** - the multi-angle view in a sports app, a
security-camera wall, a concert stream with a stage cam and three performer
cams, or a shopping livestream with a product close-up alongside the host.

The pattern here is deliberately small: there is no player object at all.
Every tile is an ArkUI `Video` component with `autoPlay(true)`, `loop(true)`
and `controls(false)`, and the "switch angle" gesture is a two-element swap
inside a `@State` array. The main tile is simply index 0.

It generalises to any grid where one cell is privileged and the rest are
interchangeable. What makes it worth reading is the pair of decisions around
the swap: the `ForEach` key is the **video filename**, not the index, so the
swap moves an already-playing component instead of rebuilding it; and the
visibility of each tile is keyed by that same filename in a separate `Map`, so
closing a tile survives every later reshuffle. `HW-13-0054` is a one-character
mistake in exactly that map - read it before copying the initializer.

## Feature checklist

- A dark page titled 主镜头 with a two-column grid of six video tiles.
- The tile at index 0 spans two rows and is labelled 主镜 (main camera); every
  other tile is labelled 副镜 (secondary camera).
- All visible tiles autoplay, loop and show no transport controls.
- Every tile keeps a 16:9 aspect ratio regardless of its grid span.
- Tapping a secondary tile swaps it with the main tile - both the video source
  and the caption move together.
- Tapping the main tile does nothing.
- Each secondary tile carries a close cross in its top-right corner; tapping it
  replaces the video with a grey placeholder.
- Tapping the placeholder brings the video back.
- Sending the app to the background stops every tile; returning to the
  foreground restarts them.

## Architecture

One `entry` module, three ArkUI files plus a constants class. No model layer,
no player manager, no media API imports at all.

```
entry/src/main/ets
├── constants/StyleConstants.ets     every numeric literal and the three booleans
├── entryability/EntryAbility.ets    full screen, avoid areas -> AppStorage, isForeGround flag
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets               @Entry, 33 lines: padding + MultiCameraVideo()
└── views
    ├── MultiCameraVideo.ets         the grid, the three parallel arrays, the swap
    └── VideoComponent.ets           one tile: Video + close cross + 主镜/副镜 badge
```

The documented tree matches the zip exactly, including file case - unusually
for this industry, where `HW-13-0003` catalogues trees that were hand-written
rather than regenerated.

**The design decision worth copying** is that the grid holds *names*, not
component state. `videoList` is a `string[]` of rawfile names, `titleList` a
parallel `Resource[]`, and `visibleMap` a `Map<string, boolean>` keyed by the
name. Because the `ForEach` key generator is `(item: string) => item`, ArkUI
identifies each tile by its filename: after `changeIndex(0, index)` swaps two
array slots, the framework recognises both keys, moves the existing subtrees,
and the `Video` components are never torn down. Playback continues across the
swap with no seek, no reload, no black frame. Key on the index instead and you
get the opposite behaviour - two tiles rebuilt from scratch and two restarted
videos.

The visibility map is keyed by name for the same reason and pays off in the
same place: `clearVideoByIndex` looks up `videoList[index]` to get a name, so a
tile closed before a swap stays closed after it. The parallel `titleList` is
the weak point of the arrangement - it must be swapped in lockstep with
`videoList` by hand, which `changeIndex` does, and nothing enforces.

**Worth knowing before you copy `layoutOptions`:** `irregularIndexes: [0]`
means `onGetIrregularSizeByIndex` is consulted *only* for index 0. The `return
[1, index % this.videoList.length + 1]` fallback in that callback is therefore
dead code - and a good thing too, since for index 2 it would ask for a
three-column span inside a two-column grid.

## Implementation steps

1. **Put the rawfiles in `resources/rawfile`** and list their names in a
   `@State string[]`. There is no path handling and no fd anywhere - `$rawfile`
   takes the bare filename.
2. **Build the parallel caption array** as `Resource[]` so the captions are
   translatable, and swap it in the same function that swaps the sources.
3. **Key the visibility map by filename, once per file** (`HW-13-0054`). The
   shipped initializer lists `rain.mp4` twice and never lists `heart.mp4`, so
   `visibleMap.get('heart.mp4')` is permanently `undefined`.
4. **Declare index 0 irregular** with `irregularIndexes: [0]` and return
   `[1, 2]` from `onGetIrregularSizeByIndex` so the main camera occupies two
   rows of the two-column grid.
5. **Key the `ForEach` by the item string**, not the index, or the swap
   restarts both videos.
6. **Give every tile `.aspectRatio(16 / 9)`** on the `VideoComponent`, not on
   the `Video` inside it, so the badge and the close cross are laid out inside
   the same box.
7. **Return early when index 0 is tapped** - swapping the main camera with
   itself would still dirty the arrays.
8. **Push foreground state through `AppStorage`** from the ability's
   `onForeground` / `onBackground`, and have each tile `@Watch` it to call
   `controller.start()` / `controller.stop()`.

## Verified snippets

All snippets are from `MultiCameraVideo.zip`. Corrected forms are marked.

**The grid and the swap — `entry/src/main/ets/views/MultiCameraVideo.ets`** (corrected, see `HW-13-0054`)

```typescript
@Component
export struct MultiCameraVideo {
  @State visibleMap: Map<string, boolean> = new Map([
    ['ranch.mp4', true], ['flower.mp4', true], ['product.mp4', true],
    ['rain.mp4', false], ['heart.mp4', false], ['music.mp4', false]
    //                     ^ FIX: the sample repeats 'rain.mp4' here and never keys heart.mp4
  ]);
  @State videoList: string[] = ['ranch.mp4', 'flower.mp4', 'product.mp4',
    'rain.mp4', 'heart.mp4', 'music.mp4'];
  @State titleList: Resource[] = [
    $r('app.string.ranch'), $r('app.string.flower'), $r('app.string.snow'),
    $r('app.string.rain'), $r('app.string.heart'), $r('app.string.music')
  ];
  layoutOptions: GridLayoutOptions = {
    regularSize: [1, 1],
    irregularIndexes: [0],                       // only index 0 is asked for a size
    onGetIrregularSizeByIndex: (index: number) => {
      if (index === 0) {
        return [1, 2];                           // main camera: one row, two columns
      }
      return [1, index % this.videoList.length + 1];
    }
  };

  changeIndex(i: number, j: number) {
    let temp: string = '';
    temp = this.videoList[i];
    this.videoList[i] = this.videoList[j];
    this.videoList[j] = temp;
    let tmp: Resource;
    tmp = this.titleList[i];                     // the caption must travel with the source
    this.titleList[i] = this.titleList[j];
    this.titleList[j] = tmp;
  }

  clearVideoByIndex(index: number) {
    this.visibleMap.set(this.videoList[index], false);
  }
}
```

**Three literals carry the design.** `irregularIndexes: [0]` is what makes the
first tile twice as wide without a second grid or a header slot - `Grid`
handles the reflow. `regularSize: [1, 1]` fixes every other tile at one cell,
so the six sources land in a stable 1-big-plus-5-small arrangement. And
`visibleMap` being keyed by *name* rather than position is what lets
`clearVideoByIndex` translate an index to a name at the moment of the tap and
never think about indices again.

The duplicate `'rain.mp4'` in the initializer is invisible in normal use only
because `undefined` and `false` are both falsy at the `if (this.visibleMap.get(item))`
call site: the heart tile renders as a placeholder, which happens to be the
intended initial state. It stops being invisible the moment any code needs to
distinguish "explicitly hidden" from "never seen this key" - a persisted
layout, a "restore all" button, an analytics count of hidden tiles.

**The grid body — same file** (as shipped)

```typescript
Grid(undefined, this.layoutOptions) {
  ForEach(this.videoList, (item: string, index: number) => {
    GridItem() {
      Column() {
        Row() {
          Text(this.titleList[index])
            .fontSize(StyleConstants.FONT_SIZE_14)
            .fontColor(Color.White)
            .textAlign(TextAlign.Start);
        }.width(StyleConstants.FULL).height(StyleConstants.HEIGHT_32);

        if (this.visibleMap.get(item)) {
          VideoComponent({
            videoName: item,
            index: index,
            clearVideoByIndex: (index1: number) => {
              this.clearVideoByIndex(index1);
            }
          }).aspectRatio(StyleConstants.ASPECT_RATIO)
            .onClick(() => {
              if (index === 0) {
                return;                          // tapping the main camera is a no-op
              }
              this.changeIndex(0, index);
            });
        } else {
          Column() {
            Image($r('app.media.path4')).width(StyleConstants.WIDTH_24);
          }
          .backgroundColor($r('app.color.icon_color'))
          .aspectRatio(StyleConstants.ASPECT_RATIO)
          .onClick(() => {
            this.visibleMap.set(item, true);     // the placeholder is the restore button
          });
        }
      }
    };
  }, (item: string) => item);                    // key = filename, NOT index
}
.columnsTemplate('1fr 1fr')
.height(StyleConstants.HEIGHT_645)
```

`aspectRatio` appears on both branches of the `if`, and it must: the
placeholder has to occupy exactly the space the video vacated or the grid
reflows every time a tile is closed. Applying it to `VideoComponent` (the
custom component) rather than to the `Video` inside it means the badge and the
close cross are positioned against the same 16:9 box the video fills.

The `else` branch doubling as the restore control is the cheapest possible
undo: no toast, no undo bar, no state beyond the map entry that was just
written.

**One tile — `entry/src/main/ets/views/VideoComponent.ets`** (as shipped)

```typescript
@Component
export struct VideoComponent {
  @Prop videoName: string = '';
  @Prop index: number = -1;
  @Watch('network') @StorageLink('isForeGround') isForeGround: boolean = false;
  @Require clearVideoByIndex: (index: number) => void;
  controller: VideoController = new VideoController();

  network() {
    if (this.isForeGround) {
      this.controller.start();                   // back in front: resume this tile
    } else {
      this.controller.stop();                    // backgrounded: stop, do not leave six decoders running
    }
  }

  build() {
    Stack() {
      Stack() {
        Video({
          src: $rawfile(this.videoName),
          controller: this.controller
        })
          .autoPlay(StyleConstants.TRUE)
          .controls(StyleConstants.FALSE)
          .loop(StyleConstants.TRUE);
        if (this.index !== 0) {
          Column() {
            Image($r('app.media.ic_public_close')).width(StyleConstants.WIDTH_12);
          }
          .position({ right: StyleConstants.POSITION_RIGHT_4, top: StyleConstants.POSITION_TOP_4 })
          .onClick(() => {
            this.clearVideoByIndex(this.index);
          });
        }
        Column() {
          Text(this.index === 0 ? '主镜' : '副镜')  // main camera / secondary camera
            .fontSize(StyleConstants.FONT_SIZE_10)
            .fontColor(Color.White);
        }
        .position({ left: StyleConstants.POSITION_LEFT_4, top: StyleConstants.POSITION_TOP_4 });
      };
    };
  }
}
```

**`@Watch` on a `@StorageLink` is the whole background story.** The ability
writes `AppStorage.setOrCreate('isForeGround', ...)` in `onForeground` and
`onBackground`; every tile links that key and reacts independently, so six
decoders stop and start without the page coordinating anything. This matters
more here than in a single-player app: leaving a six-tile wall decoding in the
background is six times the drain.

Note that each tile owns its `VideoController`. The controller is *not* part
of the swappable state - it stays with the component instance, which stays with
the filename key, which is why the swap does not disturb playback. Also note
`@Prop index`, not `@Link`: the tile is told where it currently sits and
re-renders its own badge when the parent's arrays change, but it cannot move
itself.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - every video ships
as a rawfile inside the HAP, and nothing touches the gallery, the network or
the filesystem.

`deviceTypes` is `phone`, `tablet`, `2in1`. The ability sets
`COLOR_MODE_LIGHT` in `onCreate`, then paints the whole grid black anyway.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The grid height is a hardcoded 645 vp** (`StyleConstants.HEIGHT_645`) and
  it is the only vertical sizing in the page. On a tablet or a resized 2in1
  window the wall neither grows nor scrolls, despite `2in1` being a declared
  device type.
- Six simultaneously decoding `Video` components is a real load. The sample
  gets away with it because the clips are short rawfiles; a production wall
  should decode only the visible tiles and drop the rest to a poster frame.
- The `ForEach` key is the filename, so **two tiles may never show the same
  file**. Duplicate names would collide into one component.
- `EntryAbility` registers `avoidAreaChange` and never releases it, and drops
  the promise from `setWindowLayoutFullScreen` - the same window boilerplate
  that recurs across this corpus. Harmless in a single-ability sample, wrong in
  an app with more than one page.

## Pitfalls

- **`HW-13-0054`** (B/low, confirmed): the `visibleMap` initializer lists
  `'rain.mp4'` twice and omits `'heart.mp4'`, so `visibleMap.get('heart.mp4')`
  is always `undefined`. It works today only because `undefined` is falsy in
  the same place the intended `false` would be; any logic that distinguishes
  "hidden" from "unknown key" breaks. Fix: key each of the six files once.
- **Dead irregular-size branch** (observation, no HW id): with
  `irregularIndexes: [0]`, `onGetIrregularSizeByIndex` is never called for a
  non-zero index, so the `[1, index % length + 1]` fallback is unreachable.
  Copy it into a grid with more irregular indexes and index 2 will request a
  three-column span from a two-column template.
- **Parallel arrays with no invariant** (observation, no HW id): `videoList`
  and `titleList` are held in step only by `changeIndex` swapping both. A
  single-array `{ src, title }` model would make the invariant structural.

## References

- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `VideoController`, `autoPlay`, `loop`, `controls`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `GridLayoutOptions`, `irregularIndexes`, `onGetIrregularSizeByIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `huawei_industry_tree/13_media_entertainment/docs/21_multi_camera_video.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_camera_video-0000002297933590
- `MEDIA-24` - the other end of the spectrum: one `AVPlayer` on an `XComponent` surface with explicit lifecycle
