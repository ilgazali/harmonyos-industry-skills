---
id: MEDIA-18
title: Auto-playing video feed - one AVPlayer re-targeted at the surface of the first fully visible list item
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/18_video_playlist.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_playlist-0000002286827110
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoPlayList.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, emitter, hilog, media, resourceManager, util, window]
min_api: 16
modules: [entry (entry)]
findings: [HW-13-0004, HW-13-0005, HW-13-0047, HW-13-0012, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a **scrolling feed has to play video inline** - the item
the user stopped on starts playing by itself, everything else shows a still.
Short-video feeds, news feeds with video cards, favourites lists, product
galleries: all the same problem.

The load-bearing idea is that there is exactly **one** `AVPlayer` for the whole
list, and it is moved between surfaces rather than duplicated. Each list item
owns an `XComponent` and therefore a surface id; when scrolling stops, the page
computes which index is fully visible, resets the shared player, points
`surfaceId` at that item's surface and starts. Ten visible cards cost one
decoder, not ten.

That generalises to any "only one of N children may be live" resource: a single
camera preview moved between cards, one audio renderer across a playlist, one
WebView instance recycled down a feed. The two mechanics to copy are the
**deferred play** (an item scrolled into view may not have a surface yet, so
remember its id and play from the surface's `onLoad`) and the **settle delay**
(don't start playing until the user has actually stopped, and re-check the
index when the timer fires).

**This sample never releases its player** (`HW-13-0005`) and never closes its
rawfile descriptors (`HW-13-0012`); both matter more here than in a
single-video page because the feed re-opens a descriptor on every swipe.

## Feature checklist

- A vertical `List` of 16:9 video cards, each with a title, a tag chip, an
  uploader row and a preview image.
- The list is fed by a `LazyForEach` over an `IDataSource`, nine items per page.
- When scrolling stops, the first fully visible card starts playing after a
  600 ms settle delay; scrolling on again before it elapses cancels it.
- The playing card swaps its preview image for a live `XComponent` only once
  the first frame has actually rendered.
- The `XComponent` is resized to the video's real aspect ratio, letterboxed
  inside the fixed card frame.
- A thin progress `Slider` sits along the bottom edge of the playing card, and
  a mute button toggles audio.
- Leaving a card remembers its playback position; coming back resumes from it.
- Pull to refresh rebuilds the list; reaching the end appends another page
  behind a spinner.
- Leaving the page pauses; returning resumes.

## Architecture

One `entry` module. The split is genuinely two-layer: a page that knows about
layout and scrolling, and a manager that knows about the player.

```
entry/src/main/ets
├── common
│   ├── Constants.ets              feed geometry and timings (item padding, divider, delays)
│   └── Object.ets                 VideoInfo + VideoDataSource (IDataSource, refresh/load)
├── entryability/EntryAbility.ets  full-screen layout, status-bar height -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                @Entry: list, scroll maths, XComponent lifecycle, controls
└── viewmodel/AvplayerManager.ets  the single AVPlayer, its callbacks, its state machine
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that `AVPlayerManager` never touches
the UI and `Index` never touches the player's state machine. The manager
publishes everything the page needs through `AppStorage`
(`CurrentTime`, `DurationTime`, `StartRender`) and one `emitter` event
(`'prepared'`, carrying the surface id plus the video's real width and height).
The page consumes those with `@StorageLink` and one `emitter.on`.

The `'prepared'` payload carrying the **surface id** is the detail that makes
this safe. By the time the player reports prepared, the user may already have
scrolled to a different card; the page checks the id against the currently
intended item before resizing:

```typescript
if (playSurfaceID === surfaceID) {
  this.setXComponentWH(vWidth, vHeight);
}
```

Without that guard, a late `prepared` from an abandoned start would resize the
wrong card. The same identity check appears in the `unloadID` path and in the
settle timer - three places, one principle: **every asynchronous continuation
re-validates that the target it was launched for is still the target.**

## Implementation steps

1. **Fix the item height up front** from the display width:
   `frameWidth = winWidth - 2 * padding`, `frameHeight = frameWidth / (16/9)`,
   `listItemHeight = frameHeight + padding + infoAreaHeight`. All the scroll
   maths depends on every item being exactly this tall.
2. **Tell the `List` about it** with `.childrenMainSize(new ChildrenMainSize(this.listItemHeight))`
   so `LazyForEach` can compute offsets without measuring.
3. **Give every `VideoInfo` its own `XComponentController`** at data-creation
   time, and record the surface id in the component's `onLoad`, clearing it in
   `onDestroy`.
4. **On `onScrollStop`, derive the index from `currentOffset().yOffset`,**
   rounding up when the item is more than the padding scrolled past.
5. **Delay the actual play by 600 ms and re-check** that the index is still
   the intended one when the timer fires.
6. **If the target item has no surface yet, store its id in `unloadID`** and
   start from that `XComponent`'s `onLoad` instead.
7. **Reset the shared player before re-targeting it,** saving the outgoing
   item's `currentTime` into its `VideoInfo.stopTime` so returning resumes.
8. **Guard `onReachEnd` against re-entry** - the flag exists but is never read
   (`HW-13-0047`).
9. **Release the player in `aboutToDisappear`** and close each rawfile
   descriptor after it (`HW-13-0005`, `HW-13-0012`).

## Verified snippets

All snippets are from `VideoPlayList.zip`. Corrected forms are marked.

**The scroll-stop index computation — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
.onScrollStop(() => {
  let yOffset = this.listScroller.currentOffset().yOffset;
  let curIndex = Math.floor(yOffset / (this.listItemHeight + Constants.LIST_DIVIDER_WIDTH));
  let offsetInItem = yOffset - curIndex * (this.listItemHeight + Constants.LIST_DIVIDER_WIDTH);
  if (offsetInItem > Constants.LIST_ITEM_PADDING) {
    curIndex += 1;                       // top item is more than its padding cut off
  }
  this.curIndex = curIndex;
  if (curIndex !== this.playIdx && curIndex < this.dataSource.totalCount()) {
    setTimeout(() => {
      if (this.curIndex === curIndex && this.curIndex !== this.playIdx) {
        this.play(curIndex);             // re-validate: the user may have scrolled again
      }
    }, Constants.DELAY_MS);              // 600 ms settle
  }
})
```

**Uniform item height is what buys the arithmetic.** Every `ListItem` is
`listItemHeight` tall and every divider `LIST_DIVIDER_WIDTH`, so the index of
the item under the top edge is a division - no `onVisibleAreaChange` per item,
no `IntersectionObserver`, no per-item state. `offsetInItem` is how far into
that item the top edge has travelled; once it exceeds the card's own padding
the card is no longer *fully* visible, so the feed advances to the next one.
This is the "first fully visible item" rule the doc promises, expressed in two
lines.

The 600 ms `setTimeout` is not cosmetic. `onScrollStop` fires at the end of
every fling, including the momentary stops of a flick-flick-flick gesture;
starting a decoder on each would thrash. The re-check inside the closure
(`this.curIndex === curIndex`) is what turns the delay into a cancellation:
`curIndex` is captured, `this.curIndex` is live, and a further scroll makes
them disagree so the stale timer does nothing. Note `curIndex` is a plain
private field, not `@State` - it drives no rendering, only this comparison.

**Moving the one player between surfaces — same file** (as shipped)

```typescript
play(index: number) {
  this.startRender = false;                       // hide the surface, show the preview
  this.avPlayerMgr.reset().then(() => {
    if (index !== this.playIdx) {
      this.dataSource.getData(this.playIdx).stopTime = this.currentTime;   // remember where we left
      this.playIdx = index;
    }
    if (this.dataSource.getData(index).surfaceID !== undefined) {
      this.unloadID = undefined;
      let src = this.dataSource.getData(index).src!;
      let stopTime = this.dataSource.getData(index).stopTime!;
      let surfaceID = this.dataSource.getData(index).surfaceID!;
      this.avPlayerMgr.start(src, stopTime, surfaceID);
    } else {
      this.unloadID = this.dataSource.getData(index).id;   // no surface yet: defer
    }
  });
}

// inside videoItemBuilder, on the XComponent:
.onLoad(() => {
  let surfaceID = info.xController!.getXComponentSurfaceId();
  info.surfaceID = surfaceID;
  if (info.id === this.unloadID) {
    this.play(index);                             // the deferred start, now that a surface exists
  }
})
.onDestroy(() => {
  info.surfaceID = undefined;                     // recycled out of the cache window
})
```

**`unloadID` is the whole trick.** `LazyForEach` with `cachedCount(8)` only
materialises nearby items, so an index scrolled to quickly can be the play
target before its `XComponent` exists. Rather than polling or forcing the item
to build, the page records *which item it wanted* - by `id`, not by index,
because a refresh renumbers indices - and the surface announces itself when it
arrives. `onDestroy` clearing `surfaceID` closes the loop: an item that leaves
the cache window is correctly treated as surfaceless again.

`startRender = false` before every switch, flipped back by the manager's
`startRenderFrame` callback, is what prevents the one-frame flash of the
previous video on the new card: the `XComponent` stays `Visibility.Hidden` and
the still preview stays up until a real frame has been drawn.

**The player wrapper, with teardown — `entry/src/main/ets/viewmodel/AvplayerManager.ets`** (corrected, see `HW-13-0005`, `HW-13-0012`)

```typescript
export class AVPlayerManager {
  private context: Context;
  private seekTime?: number;
  private surfaceID?: string;
  private isLoop: boolean = true;
  private avPlayer?: media.AVPlayer;
  private openedRawPath?: string;                  // FIX: nothing tracks the descriptor

  private async setMediaAsset(src: string) {
    if (this.openedRawPath !== undefined) {
      this.context.resourceManager.closeRawFdSync(this.openedRawPath);   // FIX: per-swipe leak
    }
    let rawFD: resourceManager.RawFileDescriptor = await this.context.resourceManager.getRawFd(src);
    this.openedRawPath = src;                      // FIX
    let avFD: media.AVFileDescriptor = { fd: rawFD.fd, offset: rawFD.offset, length: rawFD.length };
    this.avPlayer!.fdSrc = avFD;
  }

  async release() {                                // FIX: no release() exists in the project
    if (this.avPlayer === undefined) {
      return;
    }
    this.avPlayer.off('error');
    this.avPlayer.off('startRenderFrame');
    this.avPlayer.off('durationUpdate');
    this.avPlayer.off('timeUpdate');
    this.avPlayer.off('stateChange');
    await this.avPlayer.release();
    this.avPlayer = undefined;
    if (this.openedRawPath !== undefined) {
      this.context.resourceManager.closeRawFdSync(this.openedRawPath);
      this.openedRawPath = undefined;
    }
  }
}
```

and the `prepared` branch of `stateChange` that the page depends on, as shipped:

```typescript
case 'prepared':
  this.avPlayer!.loop = this.isLoop;
  this.avPlayer!.setMediaMuted(media.MediaType.MEDIA_TYPE_AUD, this.isMuted);
  this.avPlayer!.seek(this.seekTime!, media.SeekMode.SEEK_CLOSEST);
  this.avPlayer!.play();
  let eventData: emitter.EventData = {
    data: {
      surfaceID: this.surfaceID,
      width: this.avPlayer!.width,
      height: this.avPlayer!.height,
    }
  };
  emitter.emit('prepared', eventData);
  break;
```

**`prepared` is the only place the video's real dimensions exist**, which is
why the resize has to be event-driven rather than read from the data source -
the feed's `VideoInfo` records no resolution. Emitting them alongside the
surface id lets `setXComponentWH` letterbox correctly for both landscape and
portrait clips inside one fixed 16:9 frame.

The corrections are two omissions of the same kind. `setMediaAsset` calls
`getRawFd` on **every** start - every swipe, every refresh, every resume - and
the project contains no `closeRawFd` at all, so descriptors accumulate for as
long as the user scrolls. And there is no `release()` anywhere: the sample's
`aboutToDisappear` only calls `emitter.off('prepared')`. The player guide ends
its lifecycle with `release()` for a reason - `AVPlayer` holds a native decoder
and an audio stream. Note the ordering above: unregister callbacks, release the
player, and only then close the descriptor it was reading from.

**Paging the feed — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-13-0047`)

```typescript
.onReachEnd(() => {
  if (this.isLoading) {
    return;                                        // FIX: absent - the flag only drove the spinner
  }
  this.isLoading = true;
  setTimeout(() => {
    this.dataSource.load();                        // appends LOAD_COUNT (9) more items
    this.isLoading = false;
  }, Constants.LOADING_TIME_MS);
})
```

`onReachEnd` fires whenever the list is at its end boundary, and with
`EdgeEffect.Fade` and a friction of 2.0 an over-scroll produces several such
events inside the 1000 ms window. The shipped code sets `isLoading` purely to
drive the footer's `LoadingProgress` visibility and never reads it as a guard,
so one enthusiastic flick at the bottom appends 18 or 27 items instead of 9.
The fix is the one line above; the flag already exists and already means the
right thing.

## Permissions & config

**None declared.** All nine feed entries cycle two rawfile videos
(`video1.mp4`, `video2.mp4`) and three bundled preview images, so no media
library, no network and no user-grant permission is involved.

`EntryAbility` writes the status-bar height into `AppStorage` under
`TopRectHeight` and the page reads it with `@StorageLink`, applying it as
`padding({ top: this.uiContext!.px2vp(this.topRectHeight) })`.

## Constraints

- **The document's 约束与限制 claims API Version 20 Release or later, but the
  zip declares `compatibleSdkVersion: "5.0.4(16)"`** (`HW-13-0004`). The sample
  builds and runs four API versions below what the page states.
- The uniform-height assumption is absolute. Any variable-height card breaks
  `onScrollStop`'s arithmetic silently - it will pick a plausible but wrong
  index. If your cards vary, replace the maths with `onVisibleAreaChange` or
  `List.getVisibleListContentInfo`.
- `LazyForEach` keys on `info.id`, a fresh `generateRandomUUID()` per item, so
  a `refresh()` invalidates every key and rebuilds the whole list - which is
  what makes the deferred-play path routine rather than exceptional.
- `dataSource.load()` never stops: the feed is infinite by construction and
  there is no "no more items" terminal state.
- `stopTime` is written only when switching away *from* the playing item, and
  `refresh()` discards it along with the item, so resume is per-session and
  per-item only.
- Mute is a page-level boolean applied to the shared player, not a per-item
  property; it persists across card switches because `prepared` re-applies
  `isMuted`.

## Pitfalls

- **`HW-13-0004`** (E/low, confirmed): the constraints section claims
  API Version 20 while `build-profile.json5` targets `5.0.4(16)`. Two sibling
  media docs (`40_audio-v1_2-ts_64`, `41_video_subtitle`) overstate the same
  way against `5.0.5(17)`. Fix: align the 约束与限制 text with the samples'
  actual `compatibleSdkVersion`, or bump the samples.
- **`HW-13-0005`** (B/medium, confirmed): `AVPlayerManager` creates an
  `AVPlayer` and the project contains no `release()` call anywhere;
  `aboutToDisappear` only unregisters the emitter. Four other media samples in
  this industry share the defect - including `VideoPlayerResumeDemo`, whose
  whole subject is player lifecycle. Fix: add a `release()` on the manager and
  call it from `aboutToDisappear`.
- **`HW-13-0047`** (B/medium, probable): `onReachEnd` sets `isLoading` but
  never checks it, so a single over-scroll schedules several 1000 ms page
  loads and appends duplicate pages. Fix: `if (this.isLoading) return;`.
- **`HW-13-0012`** (B/low, confirmed): `setMediaAsset` calls
  `resourceManager.getRawFd` on every start and no `closeRawFd` exists in the
  project. Unbounded here - one descriptor per swipe and per refresh, for the
  process lifetime. Part of a seven-sample pattern across this industry. Fix:
  close the previous descriptor before opening the next, and again after
  `release()`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `childrenMainSize`, `cachedCount`, `onScrollStop`, `onReachEnd`, `divider`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `ListScroller.currentOffset()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, `onDataReloaded`, `onDataAdd`, keying
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE`, `getXComponentSurfaceId`, `onLoad` / `onDestroy`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` - `Refresh`, `pullToRefresh`, `refreshOffset`, `pullDownRatio`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-guides/02_media/video-playback.md` - the AVPlayer video lifecycle, surface binding and `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `surfaceId`, `fdSrc`, `startRenderFrame`, `loop`, `setMediaMuted`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFd` and `closeRawFd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `MEDIA-17` - the same rawfile-descriptor omission, bounded there because the fds are opened once
