---
id: MEDIA-39
title: Scrub preview - AVImageGenerator thumbnails in a popup that follows the slider
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/39_video_thumbnail.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_thumbnail-0000002565954707
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoThumbnail.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, fileIo, hilog, media, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0088, HW-13-0089, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for the **preview-while-scrubbing** control: drag the progress
bar and a small still of that moment floats above your thumb. It is the one
video-player affordance users notice when it is missing, and it is what turns
seeking from a guess into a decision.

The mechanism is a second media object. `AVPlayer` keeps playing (or stays
paused) on the `XComponent` surface while a separate `AVImageGenerator`, opened
on the **same file with its own file descriptor**, answers
`fetchFrameByTime(timeUs, AV_IMAGE_QUERY_CLOSEST, params)`. Never seek the
player to build a preview - seeking is stateful, slow, and visible.

The generalisation is "a cheap read-only view of a heavy media object,
alongside it". The same split gives you a filmstrip under a video editor
timeline, a chapter-marker grid, or a poster frame for a share sheet. What
transfers is the discipline around it: one in-flight request at a time, a
guard flag that is always released, and a preview whose x-offset is computed
from the slider value rather than from touch coordinates.

## Feature checklist

- Full-screen landscape video on an `XComponent` surface; `input.mp4` is copied
  from `rawfile` into the sandbox on first launch.
- Tapping the video toggles the whole control overlay (back button, transport,
  slider, time).
- The slider spans 70% of the width and reflects `currentTime` / `duration`
  live.
- Dragging (`SliderChangeMode.Moving`) shows a rounded popup above the slider
  holding the frame at the dragged time.
- The popup follows the thumb and stops at both ends instead of leaving the
  track.
- Releasing (`SliderChangeMode.End`) hides the popup and seeks the player to
  that time.
- The elapsed/total readout is formatted `mm:ss`, or `hh:mm:ss` past an hour.
- Both the player and the generator - and both file descriptors - are released
  in `aboutToDisappear`.

## Architecture

One `entry` module, five ArkTS files.

```
entry/src/main/ets
├── common/Constants.ets           POPUP_RADIUS 10, POPUP_WIDTH 200, RAW_FILE_NAME 'input.mp4'
├── common/utils.ets               time2str (mm:ss / hh:mm:ss), setScreenOrientation
├── entryability/EntryAbility.ets  immersive setup
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                @Entry - the whole UI, the slider handler, the popup builder
├── viewmodel/VideoPlayer.ets      AVPlayer wrapper: state machine, AppStorage publishing
└── viewmodel/VideoThumbnail.ets   AVImageGenerator wrapper: init / setMediaAsset / fetchFrame / release
```

The documented tree matches the zip.

**The design decision worth copying** is the two symmetrical wrappers. Both
`VideoPlayer` and `VideoThumbnail` expose exactly `init()`, `setMediaAsset(path)`,
`release()`, and both hold their own `fileIo.File`. Neither knows the UI exists;
they publish through `AppStorage`:

```typescript
AppStorage.setOrCreate('VideoDuration', timeMS);
AppStorage.setOrCreate('VideoCurrTime', this.avPlayer?.currentTime);
AppStorage.setOrCreate('VideoAspectRadio', this.avPlayer.width / this.avPlayer.height);
AppStorage.setOrCreate('IsPlaying', true);
```

and the page consumes them as `@StorageProp`. That is what keeps `Index.ets`
free of media state: it owns only `isShow`, `popupShow`, `popupOffsetX` and
`thumbnail` - four things about the UI and nothing about the video. It also
means the page never has to worry about which thread the player's callbacks
arrive on.

The cost is a global namespace. `VideoThumbnail.fetchFrame` reads
`AppStorage.get('VideoAspectRadio')` written by `VideoPlayer` - an implicit
dependency between two classes that otherwise do not know about each other,
and the place `HW-13-0088` hides.

## Implementation steps

1. **Copy the rawfile into the sandbox once.** `AVImageGenerator.fdSrc` needs a
   real descriptor with an offset and length, so a `filesDir` copy guarded by
   `fileIo.access(...)` is the simplest source both objects can share.
2. **Open the file twice**, once per object. Sharing one `fd` between a player
   and a generator means sharing a file position; they seek independently.
3. **Set the surface id in the `initialized` state**, then `prepare()`, then
   read `width`/`height` in `prepared` - `avPlayer.width` is meaningless before
   that.
4. **Publish `width / height` as the aspect ratio** and remember which way
   round it is. This is one number used by two consumers, and only one of them
   reads it correctly (`HW-13-0088`).
5. **Handle `SliderChangeMode.Moving` and `.End` separately**: `Moving` shows
   the popup and fetches; `End` hides it and seeks the player.
6. **Throttle with a single in-flight flag**, because `onChange` fires far
   faster than a frame can be decoded - and reset that flag in a `finally`
   (`HW-13-0089`), never only on the success path.
7. **Ask for the frame with `AV_IMAGE_QUERY_CLOSEST`** and a `PixelMapParams`
   sized to the popup. Compute the height by **dividing** the width by the
   aspect ratio (`HW-13-0088`).
8. **Offset the popup from the slider value**, clamped to `[0, sliderWidth - POPUP_WIDTH]`,
   and attach it with `bindPopup(..., { placement: Placement.LeftTop, offset })`.
9. **Release both objects and both descriptors** in `aboutToDisappear`, and
   `display.off('change')` with them.

## Verified snippets

All snippets are from `VideoThumbnail.zip`. Corrected forms are marked.

**Fetching the frame - `entry/src/main/ets/viewmodel/VideoThumbnail.ets`** (corrected, see `HW-13-0088`)

```typescript
async setMediaAsset(path: string) {
  if (this.avImageGenerator === undefined) {
    return;
  }
  try {
    this.file = await fileIo.open(path, fileIo.OpenMode.READ_WRITE);
    let fileSize = (await fileIo.stat(path)).size;
    let fdSrc: media.AVFileDescriptor = { fd: this.file.fd, offset: 0, length: fileSize };
    this.avImageGenerator.fdSrc = fdSrc;
  } catch (err) {
    console.error(`Set AVImageGenerator media asset failed, ${JSON.stringify(err)}`);
  }
}

async fetchFrame(timeMs: number): Promise<PixelMap | undefined> {
  if (this.avImageGenerator === undefined) {
    return undefined;
  }
  let timeUs = timeMs * 1000;                                   // the API takes microseconds
  let options = media.AVImageQueryOptions.AV_IMAGE_QUERY_CLOSEST;
  let videoAspectRadio = AppStorage.get('VideoAspectRadio') as number;   // width / height
  let param: media.PixelMapParams =
    { width: Constants.POPUP_WIDTH, height: Constants.POPUP_WIDTH / videoAspectRadio };  // FIX: was *
  let frame = await this.avImageGenerator.fetchFrameByTime(timeUs, options, param);
  return frame;
}
```

**Three details decide whether this works.** The slider carries milliseconds
and `fetchFrameByTime` takes microseconds, so the `* 1000` is not decoration.
`AV_IMAGE_QUERY_CLOSEST` returns the frame nearest the requested time rather
than the nearest sync frame, which is what makes a scrub feel continuous -
`AV_IMAGE_QUERY_PREVIOUS_SYNC` would jump between keyframes several seconds
apart. And `PixelMapParams` is a decode-time resize: asking for a 200 vp-wide
frame rather than a full-resolution one is the difference between a preview
that keeps up with a drag and one that does not.

The multiply-versus-divide is the shipped bug. With `VideoAspectRadio` defined
as `width / height` (1.78 for 16:9), the sample asks for
`200 x 200*1.78 = 200 x 356` - a portrait box for a landscape video - and the
document reproduces the same formula. It is invisible only because the popup
re-stretches the result with `.aspectRatio(this.videoAspectRadio)`; what you
pay is a decode into a wrongly shaped buffer on every scrub tick.

**The slider handler - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-13-0089`)

```typescript
Slider({ min: 0, max: this.videoDuration, value: this.videoCurrTime, style: SliderStyle.OutSet })
  .height(20)
  .sliderInteractionMode(SliderInteraction.SLIDE_ONLY)
  .onChange(async (value: number, mode: SliderChangeMode) => {
    if (mode === SliderChangeMode.End) {
      this.popupShow = false;
      this.videoPlayer.seek(value);
    } else if (mode === SliderChangeMode.Moving) {
      if (!this.popupShow) {
        this.popupShow = true;
      }
      if (!this.isFetching) {
        this.isFetching = true;
        try {                                        // FIX: absent in the sample
          this.calPopupOffsetX(value);
          this.thumbnail = await this.videoThumbnail.fetchFrame(value);
        } finally {
          this.isFetching = false;                   // FIX: sample resets only on success
        }
      }
    }
  });
```

**`isFetching` is a throttle, not a lock.** `onChange` fires per touch move -
tens of events per second - while a decode takes tens of milliseconds, so
without the flag the generator is asked for frames faster than it can answer
and the previews arrive out of order. Dropping the intermediate requests is
correct behaviour here: the user only cares about the latest position.

Which is exactly why the flag must be released unconditionally. In the shipped
code a single rejected `fetchFrameByTime` - a transient decoder error, a frame
past the end of the file - leaves `isFetching` at `true` forever, and the
preview is dead for the rest of the session with no error visible anywhere.
The flag is `private isFetching` rather than `@State` on purpose: it controls
work, not rendering, and making it observable would re-render the page twice
per fetch for nothing.

`sliderInteractionMode(SliderInteraction.SLIDE_ONLY)` is the companion setting:
it disables tap-to-jump on the track, so every position change comes through
the `Moving` -> `End` sequence the preview logic assumes.

**Placing the popup - same file** (as shipped)

```typescript
calPopupOffsetX(value: number) {
  let sliderWidth = this.windowWidth * 0.7;              // the Row that holds the slider is 70%
  let radio = value / this.videoDuration;
  let offsetX = sliderWidth * radio - Constants.POPUP_WIDTH / 2;
  if (offsetX < 0) {
    offsetX = 0;
  } else if (offsetX + Constants.POPUP_WIDTH > sliderWidth) {
    offsetX = sliderWidth - Constants.POPUP_WIDTH;
  }
  this.popupOffsetX = offsetX;
}

@Builder
popupBuilder() {
  Image(this.thumbnail)
    .width('100%')
    .aspectRatio(this.videoAspectRadio);
}

// on the Row that wraps the Slider:
.width('70%')
.bindPopup(this.popupShow, {
  builder: this.popupBuilder,
  enableArrow: false,
  radius: Constants.POPUP_RADIUS,
  width: `${Constants.POPUP_WIDTH}vp`,
  placement: Placement.LeftTop,
  offset: { x: this.popupOffsetX },
})
```

**`Placement.LeftTop` plus a computed x-offset is the trick.** A popup anchored
to the slider's container would sit centred above it and stay there; anchoring
it to the container's top-left corner turns `offset.x` into an absolute
position along the track, so the popup can be driven by the slider *value*
rather than by touch coordinates - which also means it lands correctly when the
value changes without a touch.

The clamp is what keeps the preview inside the track at both ends: without it
the popup would hang half off-screen at 0% and 100%. `enableArrow: false` is
the right choice here for the same reason - an arrow would have to move with
the thumb and would drift away from the popup body once the clamp engages.

**Player state machine - `entry/src/main/ets/viewmodel/VideoPlayer.ets`** (as shipped)

```typescript
this.avPlayer?.on('stateChange', async (state: media.AVPlayerState) => {
  if (this.avPlayer === undefined) {
    return;
  }
  AppStorage.setOrCreate('IsPlaying', state === 'playing');
  switch (state) {
    case 'initialized':
      this.avPlayer.surfaceId = this.surfaceID;      // surface is set here, not before
      this.avPlayer.prepare();
      break;
    case 'prepared':
      let videoAspectRadio = this.avPlayer.width / this.avPlayer.height;
      AppStorage.setOrCreate('VideoAspectRadio', videoAspectRadio);
      this.avPlayer.play();
      break;
    default:
      break;
  }
});
```

**The order is fixed by the state machine, not by taste.** `fdSrc` moves the
player from `idle` to `initialized`; `surfaceId` may only be assigned in
`initialized`; `prepare()` is what produces `prepared`, and only there do
`width` and `height` exist. Assigning the surface earlier throws, and reading
the dimensions earlier gives zeros - which would put `NaN` into the aspect
ratio every consumer downstream depends on.

## Permissions & config

**None.** No `requestPermissions` in `module.json5`: the video ships as a
rawfile and is copied into the app's own sandbox, so no media-library or
storage permission is involved. `deviceTypes` is `["phone"]`.

The orientation is imposed from code rather than config:

```typescript
await setScreenOrientation(context!, window.Orientation.LANDSCAPE);
```

called from `XComponent.onLoad`, via `window.getLastWindow(context)` and
`setPreferredOrientation`. It is never restored on exit, so the ability leaves
the window in landscape behind it.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `windowWidth` is assigned **only** inside the `display.on('change')` callback,
  so it is `0` until the first display change event arrives. Until then
  `sliderWidth` is `0` and the clamp pins the popup to the left edge. Seed it
  once in `aboutToAppear` from `display.getDefaultDisplaySync()`.
- The `Row` width (`'70%'`) and the multiplier in `calPopupOffsetX` (`* 0.7`)
  are the same number written twice; changing one silently breaks the tracking.
- `POPUP_WIDTH` is a raw `200` used both as a vp string for `bindPopup` and as
  a number in the offset arithmetic - correct only while the popup is sized in
  vp.
- The forward button is decorative: it carries no `onClick`.
- `rawfile2sandbox` writes the file with `getRawFileContentSync`, i.e. the
  whole video through memory. Fine for a bundled demo clip, wrong for anything
  large.
- Only `AVImageGenerator` supports this pattern; there is no batched
  "give me N frames" API, so a filmstrip means N sequential fetches.

## Pitfalls

- **`HW-13-0088`** (B/medium, confirmed): the thumbnail height is computed as
  `POPUP_WIDTH * (width / height)` instead of `POPUP_WIDTH / (width / height)`,
  so a 16:9 video is decoded into a 200x356 portrait buffer instead of 200x112.
  The document reproduces the same formula. Only the popup's `.aspectRatio`
  re-stretch hides it. Fix: divide.
- **`HW-13-0089`** (B/low, confirmed): a single rejected `fetchFrameByTime`
  leaves `isFetching` latched at `true`, because the reset sits after the
  `await` with no `try`/`finally` - thumbnails stop for the rest of the
  session, silently. Fix: reset the flag in a `finally`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avimagegenerator.md` - `createAVImageGenerator`, `fdSrc`, `fetchFrameByTime`, `AVImageQueryOptions`, `PixelMapParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avimagegenerator
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the state machine, `surfaceId`, `seek`, `SeekMode`, `durationUpdate` / `timeUpdate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `onChange`, `SliderChangeMode`, `sliderInteractionMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-popup.md` - `bindPopup`, `Placement`, `offset`, `enableArrow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-popup
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - obtaining the surface id for the player
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `MEDIA-18` - the playlist card that shares this player wrapper shape (and the release defect this sample avoids)
