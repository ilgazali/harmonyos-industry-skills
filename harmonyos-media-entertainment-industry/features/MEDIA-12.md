---
id: MEDIA-12
title: Video playlist linkage - one shared index drives the player and the track list
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/12_video_switching_association.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_switching_association-0000002296452929
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoSwitchingAssociation.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, Video, VideoController, Tabs, TabContent, TabsController, "@Link", "@Watch", "@StorageLink", IDataSource, ForEach, ListScroller, "Scroller.scrollTo", Slider, SliderInteraction, "UIContext.getPromptAction", showToast, "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0034, HW-13-0035, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for the **player-plus-playlist screen**: a video pinned at the
top, a list of episodes or clips beneath it, and one selection shared between
them. Tapping a row plays it; the previous/next buttons move the same
selection; the list highlights whatever is playing.

The transferable idea is the direction of the data. There is exactly one
piece of truth, `currentIndex` on the page, published to both children with
`@Link`. Neither child owns it, so neither can disagree with the other -
tapping a row and pressing "next" go through the same state, and the
highlight is a pure function of it. That shape holds for an audio playlist, a
course with chapters, a photo carousel with thumbnails, or any master-detail
pair where the detail is a player.

**The row-selection linkage works; the scroll-follow does not.** The list is
built with `Flex` and a `ForEach`, but a `ListScroller` is created and called
on every switch, which throws (`HW-13-0034`). The fix is to build the rows in
a real `List` - which is what a playlist of any useful length needs anyway.

## Feature checklist

- A title bar, a video card, and a five-row playlist inside a bottom-tab
  shell.
- The video plays the currently selected clip from `rawfile` and shows its
  own progress bar with elapsed and total time.
- The card opens on a first frame rather than a black rectangle, without
  autoplaying.
- Play/pause toggles the video; previous and next move through the list and
  swap the source.
- Previous on the first clip and next on the last raise a toast instead of
  wrapping.
- The row of the playing clip turns blue and its title marquees; the others
  are ellipsised.
- Tapping any row plays it.
- The list scrolls to keep the playing row visible once the selection passes
  the fourth item.
- Backgrounding the app pauses the video; returning to the foreground resumes
  it.

## Architecture

One `entry` module, split page / view / model the way a feature of this size
should be.

```
entry/src/main/ets
├── common/Constants.ets           layout numbers and the scroll thresholds
├── common/ListData.ets            VIDEO_LIST_DATA - five {videoName, videoSrc} entries
├── entryability/EntryAbility.ets  full-screen, avoid areas + isForeGround -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/ItemModel.ets            VideoItem { videoName: ResourceStr; videoSrc: Resource | string }
├── model/ListDataSource.ets       IDataSource implementation over VideoItem[]
├── pages/MainPage.ets             @Entry - owns currentIndex/currentPlayVideo, the Tabs shell
├── view/ListView.ets              the playlist rows and the (broken) scroll-follow
└── view/VideoView.ets             the Video, its progress bar, and the transport row
```

The documented tree matches the zip.

**The design decision worth copying** is the pair of `@Link`s. `MainPage`
holds both the index and the resolved item:

```typescript
@State currentPlayVideo: VideoItem = { videoName: '', videoSrc: '' };
@State currentIndex: number = 0;
```

and hands the same two to both children, which declare them as `@Link`. So
`VideoView`'s next button writes `currentIndex` and `currentPlayVideo`,
`ListView`'s row tap writes the same two, and each write re-renders both
sides. `ListView` additionally decorates its copy with
`@Watch('onCurrentIndexChange')`, which is the clean way to attach a side
effect - scrolling - to a state it does not own.

Carrying *both* the index and the item is mild redundancy (the item is
`videoList.getData(currentIndex)`), and it is the reason the two must be
written together everywhere; a stricter version would publish only the index
and derive the item. It is defensible here because `Video`'s `src` wants the
resource directly.

The seam that does not hold up is `ListDataSource`. It is a full `IDataSource`
- listeners, `notifyDataAdd`, `notifyDataReload`, the lot - and no
`LazyForEach` ever consumes it. `VideoView` uses it as a plain array
(`getData`, `totalCount`) while `ListView` ignores its own copy and iterates
the imported `VIDEO_LIST_DATA` constant instead. Either drive the list from
the data source through `LazyForEach`, or delete it.

## Implementation steps

1. **Keep the selection on the page**, not in either child, and publish it
   with `@Link` so both write to the same state.
2. **Load the data source once in `aboutToAppear`** and set
   `currentPlayVideo` from index 0 before the first frame.
3. **Bind `Video`'s `src` to `currentPlayVideo.videoSrc`.** Changing the
   source is the entire "switch video" operation - there is no controller call
   to make.
4. **Suppress the black first frame** with `autoPlay(true)` plus an
   `onStart` that pauses immediately when `isPlaying` is false. The video
   decodes and paints one frame, then stops.
5. **Render the rows inside a `List` bound to the `ListScroller`.** A bare
   `Flex` leaves the scroller unbound and every `scrollTo` throws `100004`
   (`HW-13-0034`).
6. **Scroll from the `@Watch` on the index**, offsetting only once the
   selection passes the visible threshold, and resetting to the top below it.
7. **Guard the ends of the list** in the previous/next handlers and toast
   rather than wrapping.
8. **Give the progress `Slider` an `onChange`** that calls
   `controller.setCurrentTime`, or drop `sliderInteractionMode` and make it a
   read-only `Progress` (`HW-13-0035`).
9. **Pause on background and resume on foreground** by watching `isForeGround`
   from `AppStorage` and calling the real controller - this sample does it
   correctly, unlike `MEDIA-11` (`HW-13-0033`).

## Verified snippets

All snippets are from `VideoSwitchingAssociation.zip`. Corrected forms are marked.

**The shared selection - `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct VideoLinkageListView {
  @State currentPlayVideo: VideoItem = { videoName: '', videoSrc: '' };   // 当前播放的视频
  @State currentIndex: number = 0;                                        // 当前播放下标
  @State selectTabIndex: number = 1;
  private videoList: ListDataSource = new ListDataSource();
  private controller: TabsController = new TabsController();

  async aboutToAppear(): Promise<void> {
    VIDEO_LIST_DATA.forEach((video: VideoItem) => {
      this.videoList.pushData(video);
    });
    this.currentPlayVideo = this.videoList.getData(this.currentIndex);
  }

  build() {
    Column() {
      // ... title bar ...
      Tabs({ barPosition: BarPosition.End, controller: this.controller, index: this.selectTabIndex }) {
        TabContent() {
          Column() {
            VideoView({
              currentPlayVideo: this.currentPlayVideo,
              currentIndex: this.currentIndex,
              videoList: this.videoList
            }).margin({ bottom: 12 });
            ListView({
              currentPlayVideo: this.currentPlayVideo,
              currentIndex: this.currentIndex,
              videoList: this.videoList
            });
          }
        }
      }
      .vertical(false)
      .onChange((index: number) => { this.selectTabIndex = index; })
    }
  }
}
```

**Two children, one pair of variables, and no callbacks between them.** Both
child structs declare `@Link currentPlayVideo` and `@Link currentIndex`, so
the parameters above are two-way bindings, not copies - `VideoView`'s
`this.currentIndex++` is a write to `MainPage`'s state, which re-renders
`ListView`'s highlight. No event bus, no `@Provide`/`@Consume`, no store.

`videoList` is passed by plain assignment because it is an ordinary class
instance, which is fine for a fixed list but means a mutation to it after
construction would not trigger any re-render.

The `Tabs` shell holds a single `TabContent` and exists for the bottom bar
alone; `selectTabIndex` defaults to `1` for the screenshot even though there
is only index `0` to switch to. Do not read the `Tabs` here as part of the
pattern.

**The video card - `entry/src/main/ets/view/VideoView.ets`** (as shipped, with the seek corrected)

```typescript
Video({
  src: this.currentPlayVideo.videoSrc,
  controller: this.controller
})
  .controls(false)
  .autoPlay(true)
  .height(178)
  .width('100%')
  .onPrepared((event) => { this.videoDuration = event.duration; })
  .onStart(() => {
    // 视频初始化播放时暂停，解决黑屏问题
    if (!this.isPlaying) {
      this.controller.pause();
    }
  })
  .onUpdate((event) => { this.currentTime = event.time; })
  .onFinish(() => { this.isPlaying = false; });

Slider({
  value: this.currentTime, step: 1, min: 0,
  max: this.videoDuration, style: SliderStyle.OutSet
})
  .width(250)
  .trackThickness(2)
  .sliderInteractionMode(SliderInteraction.SLIDE_ONLY)
  .onChange((value: number, mode: SliderChangeMode) => {     // FIX: absent in the sample
    if (mode === SliderChangeMode.End) {
      this.controller.setCurrentTime(value, SeekMode.Accurate);
      this.currentTime = value;
    }
  });
```

**`autoPlay(true)` with an immediate `pause()` in `onStart` is the trick
worth taking.** Its Chinese comment - 视频初始化播放时暂停，解决黑屏问题
(pause on initial playback to solve the black-screen problem) - names the
problem exactly: a `Video` that has never played shows a black rectangle,
because there is no poster and nothing has been decoded. Starting playback
and stopping it inside the first `onStart` leaves one decoded frame on
screen. It costs a few milliseconds of decode and replaces the need for a
separate thumbnail asset. The guard `if (!this.isPlaying)` is what keeps it a
first-frame trick rather than an unconditional stall: once the user is
playing, `onStart` after a source change does not pause.

The rest of the wiring is conventional and correct: `onPrepared` fills the
duration, `onUpdate` the position, `onFinish` clears `isPlaying`, and
`controls(false)` hides the system controls because the row below replaces
them.

The slider is the piece that was never finished. It declares
`SliderInteraction.SLIDE_ONLY` - explicitly enabling drag on the thumb - and
ships no `onChange`, so the thumb follows the finger and then snaps back at
the next `onUpdate` (`HW-13-0035`). `setCurrentTime(value, SeekMode.Accurate)`
is the `VideoController` call it needs; gating on `SliderChangeMode.End`
issues one seek per gesture instead of one per frame.

**Switching tracks - same file** (as shipped)

```typescript
// 下一条
Image(this.currentIndex < this.videoList.totalCount() - 1 ? $r('app.media.video_linkage_list_play_next') :
  $r('app.media.video_linkage_list_play_next_on'))
  .height(20)
  .onClick(() => {
    if (this.currentIndex < this.videoList.totalCount() - 1) {
      this.currentIndex++;
      this.currentPlayVideo = this.videoList.getData(this.currentIndex);
    } else {
      this.uiContext.getPromptAction().showToast({
        message: $r('app.string.video_linkage_list_last_data_toast')
      });
    }
  });

network(changedPropertyName: string) {
  // 切换到前台了，视频播放
  if (this.isForeGround) {
    this.controller.start();
  } else {
    // 切换到后台了，视频停止播放
    this.controller.pause();
  }
}
```

**The switch is two assignments and nothing else.** No `stop()`, no
`reset()`, no controller call at all: writing `currentPlayVideo` changes
`Video`'s `src`, and the component tears down the old pipeline and builds the
new one itself. That is the whole "linkage" - and it is why the same two
lines appear identically in `ListView`'s row tap.

The same boundary condition drives the icon and the handler: the `next` image
is the enabled or disabled variant of itself depending on the same comparison
that the click handler makes, so the affordance cannot get out of step with
the behaviour.

`network()` is the counter-example to `MEDIA-11`'s `HW-13-0033`: it watches
the same `isForeGround` key from `AppStorage` and calls the real controller
in both directions instead of only flipping a flag. Note it resumes
unconditionally on foreground, so a video the user had deliberately paused
starts playing again after a trip to the home screen - gate the `start()` on
`isPlaying` if that matters.

**The playlist - `entry/src/main/ets/view/ListView.ets`** (corrected, see `HW-13-0034`)

```typescript
@Component
export struct ListView {
  @Link currentPlayVideo: VideoItem;
  @Link @Watch('onCurrentIndexChange') currentIndex: number;
  scroller: ListScroller = new ListScroller();

  onCurrentIndexChange(): void {
    if (this.currentIndex > Constants.VIDEO_LIST_SCROLL_TO_INDEX) {
      this.scroller.scrollTo({
        yOffset: Constants.VIDEO_LIST_ITEM_HEIGHT * (this.currentIndex - Constants.VIDEO_LIST_SCROLL_TO_INDEX),
        xOffset: 0
      });
    } else {
      this.scroller.scrollTo({ yOffset: 0, xOffset: 0 });
    }
  }

  build() {
    Column() {
      List({ scroller: this.scroller }) {           // FIX: sample uses a bare Flex - scroller unbound
        ForEach(VIDEO_LIST_DATA, (item: VideoItem, index: number) => {
          ListItem() {
            this.FlexExample(index, item);
          }
        });
      }
      .divider({ strokeWidth: 5, color: '#22000000' })
    }
    .height(220)
    .borderRadius(16)
    .backgroundColor(Color.White)
    .margin({ left: 16, right: 16 })
    .padding({ left: 14, right: 14 });
  }
}
```

**A `Scroller` is bound by being passed to a scrollable container, and to
exactly one.** The shipped `build()` renders
`Flex({ direction: FlexDirection.Column, ... })` with the rows and interleaved
`Divider`s inside a fixed 220vp `Column`; `this.scroller` is constructed,
never handed to anything, and then called on *every* index change - including
the `else` branch, so even selecting row 0 throws "controller not bound to
component" (error 100004). The feature is not merely inert; the watcher
raises on every interaction.

Moving the rows into `List({ scroller: this.scroller })` fixes the binding
and gives the list the scrolling it needs regardless: 220vp of viewport
against five 44vp rows already overflows, and a real playlist has more than
five. The hand-rolled `Divider` after every row but the last becomes
`.divider({...})` on the `List`, and `ListItem` wrappers replace the bare
builder calls.

Note the row builder itself is worth keeping as written: the selected row is
identified by `this.currentIndex === index` and drives three attributes at
once - the play icon variant, the text colour, and
`TextOverflow.MARQUEE` versus `TextOverflow.Ellipsis`, so only the playing
title scrolls its text. That is one comparison expressing the whole selection
state.

## Permissions & config

**None.** The sample declares no `requestPermissions`; all four videos are
`rawfile` assets (`video1.mp4`, `video2.mp4`, `flower.mp4`, `videoTest.mp4` -
five entries, with `video1.mp4` used twice).

`EntryAbility` does the usual full-screen setup, publishes `topRectHeight`
and `bottomRectHeight` into `AppStorage`, keeps an `avoidAreaChange`
subscription, and publishes `isForeGround` from `onForeground` /
`onBackground`. `MainPage` consumes the avoid areas as
`padding({ top: this.uiContext.px2vp(this.topRectHeight) })` on the title bar
- `px2vp` at the point of use, which is the correct conversion for a value
the window reported in px.

Colours and strings are resources throughout
(`app.color.video_linkage_list_item_selected`, `app.string.video_name1`...),
with `sys.color.white` and `sys.color.comp_background_list_card` used where a
system token exists.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`
  with `strictMode.caseSensitiveCheck` on, so this constraint section is
  accurate - unlike the three docs in `HW-13-0004`.
- `Video` decodes through the system player; there is no `AVPlayer` here and
  therefore nothing to `release()` - the class of leak in `HW-13-0005` does
  not apply to this sample.
- The scroll geometry is hardcoded: `VIDEO_LIST_ITEM_HEIGHT` is 30 while the
  rows actually occupy 20vp plus 12vp margins top and bottom, so even bound to
  a `List` the offset arithmetic is approximate. Prefer
  `scrollToIndex(index, true, ScrollAlign.CENTER)` over computed offsets.
- The list is a fixed five entries in `ListData.ets`; there is no paging, and
  `LazyForEach` is never used despite `ListDataSource` implementing
  `IDataSource`.
- Speed control, the alarm button and the back arrow are toasts.
- Foreground resume is unconditional - a deliberately paused video restarts
  when the app returns.

## Pitfalls

- **`HW-13-0034`** (B/medium, confirmed): `ListView` creates a `ListScroller`
  and calls `scrollTo` from the `@Watch` on every video switch, but `build()`
  renders a `Flex` + `ForEach` and no `List` is bound to the scroller - every
  prev/next/row tap throws "controller not bound to component" (100004), and
  the scroll-follow can never work. Fix: render the rows in a `List` bound to
  the scroller.
- **`HW-13-0035`** (B/low, confirmed): the progress `Slider` enables
  `SliderInteraction.SLIDE_ONLY` but has no `onChange`, so the thumb drags
  and then snaps back on the next `onUpdate`; `setCurrentTime` is never
  called. Fix: add `onChange` gated on `SliderChangeMode.End` calling
  `controller.setCurrentTime`.
- **`ListDataSource` is dead weight**: a full `IDataSource` with change
  notifications that no `LazyForEach` consumes, used as a plain array by one
  child and ignored by the other, which iterates the raw
  `VIDEO_LIST_DATA` constant instead.
- **Index and item are written as a pair in three places**; miss one and the
  player and the highlight disagree. Publishing only the index and deriving
  the item removes the hazard.

## References

- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `VideoController.setCurrentTime`, `onPrepared`/`onStart`/`onUpdate`/`onFinish`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListScroller`, `scrollTo`, `scrollToIndex`, `divider`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/errorcode-scroll.md` - error 100004, "controller not bound to component"
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/errorcode-scroll
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `TabContent`, `TabsController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderInteraction`, `SliderChangeMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `getPromptAction().showToast`, `px2vp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `MEDIA-09` - the same `Video` + `VideoController` pair, interrupted by a network switch
- `MEDIA-11` - the same `isForeGround` lifecycle key, implemented wrongly there
