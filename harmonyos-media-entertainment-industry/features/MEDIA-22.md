---
id: MEDIA-22
title: Video like burst - a transparent Canvas over the player that spawns, floats and fades emoji particles per tap
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/22_video_like.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_like-0000002332695009
sample: huawei_industry_tree/13_media_entertainment/downloads/thumbup.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, clearRect, globalAlpha, fillText, save, restore, setInterval, clearInterval, RelativeContainer, alignRules, PanGesture, PanGestureOptions, VideoController, setCurrentTime, SeekMode, "display.getDefaultDisplaySync", "cryptoFramework.createRandom", "@Link", "@StorageLink", "@Watch"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0055, HW-13-0056, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when you need **a particle burst on top of playing video** -
the flying hearts of a short-video feed, the emoji rain of a livestream, a
confetti pop on a completed action. The pattern is a full-size transparent
`Canvas` laid over the player inside a `RelativeContainer`, an array of
particle records, and one `setInterval` that advances and redraws all of them.

Reach for a `Canvas` rather than for N animated `Image` components as soon as
the particle count is unbounded. Declarative components carry a layout and a
diff cost each; a hundred emoji drawn with `fillText` cost one `clearRect` plus
a hundred `save/translate/scale/fillText/restore` sequences on the same
context, and disappear by being dropped from an array rather than by being
unmounted.

The rest of the card is about the two things this sample gets *wrong*, because
both are easy to repeat: the animation loop runs at 60 fps forever whether or
not anything is on screen (see "worth avoiding" below), and the seek bar
measures itself with two different widths (`HW-13-0055`). The particle maths
itself - a four-phase sway, an ease-in scale, a tail fade - is worth copying
verbatim.

## Feature checklist

- A four-tab shell (首页 / 短视频 / 影院 / 我的) with only the short-video tab
  populated; the tab strip does not swipe.
- The short-video tab plays a looping rawfile clip full-bleed with no native
  controls.
- The right-hand rail carries an avatar, a 关注 (follow) chip, and a
  like button whose icon toggles between the outline and filled variants.
- Tapping anywhere on the video spawns one random emoji at the tap point.
- Each emoji rises about 120 vp, sways left and right through four phases,
  scales up over the first fifth of its life, then fades out over the last
  40 %, and is destroyed after 1000 ms.
- Tapping the video also switches the rail's like icon to the filled state.
- A thin progress line under the video tracks playback.
- Dragging horizontally on the strip below the video thickens the line, hides
  the description and the rail, and seeks the video.
- Releasing the drag restores the overlay and keeps the new position.
- Returning from the background restarts playback.

## Architecture

One `entry` module. The feature splits cleanly into a layout host, a
particle layer and two dumb views.

```
entry/src/main/ets
├── constants/TabContentConstants.ets   ~45 id strings + the layout numbers
├── entryability/EntryAbility.ets       loads the page, writes isForeGround
├── entrybackupability/EntryBackupAbility.ets
├── pages/TabContentOverFlow.ets        @Entry, the four bottom tabs
├── view
│   ├── CanvasLike.ets                  the particle system: 187 lines, the whole animation
│   ├── VideoDes.ets                    author name + caption, 44 lines, no state
│   └── VideoTabContent.ets             Video + Canvas + rail + seek strip, 244 lines
└── viewmodel/MainPageData.ets          the four tab-bar entries
```

The documented tree matches the zip.

**The design decision worth copying** is the `RelativeContainer` layering in
`VideoTabContent`. The `Video`, the `CanvasLike` overlay, the description block
and the right-hand rail are all anchored to `__container__` (exposed as
`TabContentConstants.TAB_CONTAINER`) rather than nested inside each other, and
each declares an `.id()` that the next one anchors against. That is what lets
the description sit against the *canvas*'s bottom edge while the canvas sits
against the container's top edge - a nesting-based layout would have forced the
description inside the canvas, where it would swallow the taps the particle
system needs.

`CanvasLike` receives exactly two `@Link`s - `thumbupImage` and `flag` - and
writes both on tap, so a tap on the video updates the rail's icon without the
canvas knowing what a rail is.

**The design decision worth avoiding** is the unconditional 16 ms interval.
`startAnimation()` runs in `aboutToAppear` and never stops until
`aboutToDisappear`; every tick calls `updateIcons()` (which allocates a fresh
array) and `drawAllIcons()` (which issues a `clearRect` on a full-screen
canvas) even when `likeIcons` is empty - which is the state the page is in
almost all the time. Worse, `updateIcons` ends with `this.likeIcons = newIcons`,
a `@State` assignment, so ArkUI is asked to diff a component 60 times a second
for a value nothing renders from: the drawing is imperative, the array never
reaches `build()`. The fix is two lines - make `likeIcons` a plain field, and
start the interval on the first `addLikeIcon` / clear it when the array empties.

## Implementation steps

1. **Anchor the layers, do not nest them.** Give the canvas an `.id()` and
   anchor the caption and the rail against that id; anchor the video and the
   canvas against `__container__`.
2. **Size the canvas to the video** (`height($r('app.string.tab_video_height'))`,
   `width('100%')`) so tap coordinates and draw coordinates share an origin.
3. **Build the context once** as a component field:
   `new CanvasRenderingContext2D(new RenderingContextSettings(true))`. The
   `true` enables antialiasing; emoji glyphs scaled up look ragged without it.
4. **Capture the tap point from `ClickEvent`** - `event.x` / `event.y` are
   already relative to the canvas, so they can be stored as the particle's
   `initialX` / `initialY` with no conversion.
5. **Store a full record per particle**, including `createTime` and `lifespan`,
   and derive every animated property from `progress = (now - createTime) / lifespan`.
   Nothing is integrated frame by frame, so a dropped frame cannot accumulate
   drift.
6. **Rebuild the array each tick** by copying only the particles still inside
   their lifespan - that is the destruction mechanism.
7. **Draw with `save()` / `restore()` around each particle** and do the scale
   as translate-to-origin, scale, translate-back, so the glyph grows about its
   own centre rather than about the canvas origin.
8. **Divide the random byte by 256, not 255** (`HW-13-0056`).
9. **Use one width for both directions of the progress bar** (`HW-13-0055`):
   the same `screenW - TAB_INTERVAL_NUMBER` that clamps and inverts the drag
   must also map playback time to pixels.
10. **Clear the interval in `aboutToDisappear`** - the sample does this
    correctly, and it is the one lifecycle call that must not be missed.

## Verified snippets

All snippets are from `thumbup.zip`. Corrected forms are marked.

**The overlay — `entry/src/main/ets/view/VideoTabContent.ets`** (as shipped)

```typescript
RelativeContainer() {
  Video({
    src: $rawfile('tab_play_video.MP4'),
    currentProgressRate: PlaybackSpeed.Speed_Forward_1_00_X,
    previewUri: $r('app.media.preview'),
    controller: this.videoController
  })
    .autoPlay(true)
    .loop(true)
    .controls(false)
    .onPrepared((event) => {
      if (event !== undefined) {
        this.totalTime = event.duration;
      }
    })
    .alignRules({
      top: { anchor: TabContentConstants.TAB_CONTAINER, align: VerticalAlign.Top },
      left: { anchor: TabContentConstants.TAB_CONTAINER, align: HorizontalAlign.Start }
    });

  CanvasLike({ thumbupImage: this.thumbupImage, flag: this.flag })
    .id(TabContentConstants.TAB_VIDEO)          // the id everything else anchors to
    .height($r('app.string.tab_video_height'))
    .width('100%');

  VideoDes()
    .alignRules({
      bottom: { anchor: TabContentConstants.TAB_VIDEO, align: VerticalAlign.Bottom },
      left: { anchor: TabContentConstants.TAB_CONTAINER, align: HorizontalAlign.Start }
    })
    .visibility(this.isTouch ? Visibility.Hidden : Visibility.Visible);
}
```

**The canvas is the anchor, not the video.** `CanvasLike` carries
`.id(TAB_VIDEO)` and everything positioned "over the video" is really
positioned over the canvas - which is exactly the same rectangle, and is the
layer that survives if the video is swapped for another source. Note that
`CanvasLike` declares no `alignRules` of its own: with none given a
`RelativeContainer` child sits at the container origin, which is where the
video is.

`controls(false)` is what makes the overlay possible at all - the native
transport bar would eat taps in the lower third of the video.

**The particle record and the update pass — `entry/src/main/ets/view/CanvasLike.ets`** (as shipped)

```typescript
updateIcons() {
  const currentTime = Date.now();
  const newIcons: LikeIcon[] = [];
  for (let icon of this.likeIcons) {
    const existTime = currentTime - icon.createTime;
    if (existTime < icon.lifespan) {              // expiry = simply not copied over
      const progress = existTime / icon.lifespan;
      const verticalDistance = 120 * Math.pow(progress, 0.7);
      icon.y = icon.initialY - verticalDistance;  // fast start, slow finish
      let horizontalOffset = 0;
      if (progress < 0.25) {
        horizontalOffset = 0;                     // phase 1: straight up
      } else if (progress < 0.5) {
        const phaseProgress = (progress - 0.25) / 0.25;
        horizontalOffset = -icon.maxOffset * phaseProgress * icon.direction;
      } else if (progress < 0.75) {
        const phaseProgress = (progress - 0.5) / 0.25;
        horizontalOffset = icon.maxOffset * (2 * phaseProgress - 1) * icon.direction;
      } else {
        const phaseProgress = (progress - 0.75) / 0.25;
        horizontalOffset = icon.maxOffset * (1 - 2 * phaseProgress) * icon.direction;
      }
      icon.x = icon.initialX + horizontalOffset;
      if (progress < 0.2) {
        icon.scale = icon.initialScale + (icon.maxScale - icon.initialScale) * (progress / 0.2);
      } else {
        icon.scale = icon.maxScale;               // grow fast, then hold
      }
      if (progress > 0.6) {
        icon.opacity = 1.0 - ((progress - 0.6) / 0.4);
      } else {
        icon.opacity = 1.0;                       // fully opaque for the first 60 %
      }
      newIcons.push(icon);
    }
  }
  this.likeIcons = newIcons;
}
```

**Everything is a pure function of `progress`.** Position, sway, scale and
opacity are each recomputed from `(now - createTime) / lifespan` rather than
incremented, so a stalled frame produces a jump rather than permanent drift,
and a particle's trajectory is identical whether the device renders at 60 or at
30 fps.

`Math.pow(progress, 0.7)` is the whole feel of the rise: an exponent below 1
means most of the 120 vp is covered early, which reads as "launched" instead
of "floated". `direction` is randomised per particle to ±1 so a burst of taps
does not sway in unison, and `maxOffset` is 8-15 vp - small enough to read as a
wobble rather than a zigzag. Note the deliberate asymmetry between fade and
scale: the glyph reaches full size at 20 % of its life and only starts fading
at 60 %, so it is fully formed and fully opaque through the middle of the
flight, where the eye is.

**The random source — same file** (corrected, see `HW-13-0056`)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

doRandBySync(): number {
  // FIX: the sample divides by 255, so a byte of 255 yields exactly 1.0
  return cryptoFramework.createRandom().generateRandomSync(1).data[0] / 256;
}

getRandomEmoji(): string {
  return this.emojis[Math.floor(this.doRandBySync() * this.emojis.length)];
}

addLikeIcon(x: number, y: number) {
  const radius = 80 + this.doRandBySync() * 20;
  const emoji = this.getRandomEmoji();
  this.likeIcons.push(this.createIcon(x, y, radius, emoji));
}
```

**A random byte has 256 values, so the divisor is 256.** Dividing by 255 makes
the range `[0, 1]` *inclusive*, and `Math.floor(1.0 * 18)` is `18` on an
18-element array - `undefined`, which `fillText` renders as the literal text
`undefined`. It happens on one tap in 256, which is often enough to be seen and
rare enough to survive review.

The larger point is that `cryptoFramework.createRandom()` is the wrong tool
here: it constructs a CSPRNG object and pulls one byte from it, five separate
times per particle, for a decorative wobble. `Math.random()` is both cheaper
and free of the modulo-bias question. Keep the crypto RNG for tokens and nonces.

**Drawing — same file** (as shipped)

```typescript
drawAllIcons() {
  this.context.clearRect(0, 0, this.context.width, this.context.height);
  for (let icon of this.likeIcons) {
    this.context.save();
    this.context.globalAlpha = icon.opacity;
    this.context.translate(icon.x, icon.y);
    this.context.scale(icon.scale, icon.scale);
    this.context.translate(-icon.x, -icon.y);     // scale about the glyph, not the origin
    this.context.font = `${icon.fontSize}px`;
    this.context.textAlign = 'center';
    this.context.textBaseline = 'middle';
    this.context.fillText(icon.emoji, icon.x, icon.y);
    this.context.restore();
  }
}
```

**The translate-scale-untranslate sandwich is the load-bearing line.**
`context.scale` multiplies the whole coordinate system, so scaling before
drawing at `(x, y)` would also move the glyph away from the tap point by a
factor of `scale`. Translating to the point, scaling, and translating back
leaves `(x, y)` fixed and grows the glyph about itself.

`textAlign = 'center'` with `textBaseline = 'middle'` completes the effect:
without them the emoji hangs to the lower right of the tap and the growth is
visibly off-centre. `globalAlpha` is set inside the `save()` so the next
particle starts from a clean state - each particle's fade is independent.

**The progress bar — `entry/src/main/ets/view/VideoTabContent.ets`** (corrected, see `HW-13-0055`)

```typescript
private screenW: number = this.getUIContext().px2vp(display.getDefaultDisplaySync().width);

// playback -> pixels
.onUpdate((event) => {
  if (event !== undefined) {
    if (!this.isTouch) {                          // do not fight the finger
      this.offsetX =
        event.time / this.totalTime * (this.screenW - TabContentConstants.TAB_INTERVAL_NUMBER);
      //                                ^ FIX: the sample hardcodes 430 here
      this.positionX = this.offsetX;
    }
  }
})

// pixels -> playback
PanGesture(this.panOption)
  .onActionUpdate((event: GestureEvent) => {
    this.offsetX = this.positionX + event.offsetX;
    if (this.offsetX <= 0) {
      this.offsetX = 0;
    }
    if (this.offsetX >= this.screenW - TabContentConstants.TAB_INTERVAL_NUMBER) {
      this.offsetX = this.screenW - TabContentConstants.TAB_INTERVAL_NUMBER;
    }
    this.videoController.setCurrentTime(this.offsetX /
      (this.screenW - TabContentConstants.TAB_INTERVAL_NUMBER) * this.totalTime, SeekMode.Accurate);
  })
```

**Two directions, one width.** The bar's width is a rendered `Text` of width
`this.offsetX`, so `offsetX` is simultaneously "how far along the video is" and
"where the finger is". Any disagreement between the time→pixel and pixel→time
mappings makes the bar and the seek target diverge, and on a 430 vp reference
device the sample looks correct while on anything else the bar under- or
overfills and a release at mid-screen jumps to the wrong timestamp.

`SeekMode.Accurate` (rather than the faster keyframe modes) is the right call
for a drag: the user is watching the frame, not the number.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The clip is a
rawfile inside the HAP and nothing touches the gallery or the network.

`deviceTypes` is `phone`, `tablet`, `2in1`. Note that the ability entry has no
`label` - the launcher falls back to the app-level label.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The animation loop never idles.** 60 ticks a second of array rebuild,
  `@State` assignment and full-canvas `clearRect` run from `aboutToAppear` to
  `aboutToDisappear` regardless of whether any particle exists. Gate the
  interval on a non-empty array before shipping.
- `screenW` is read once in a field initializer from
  `display.getDefaultDisplaySync()`. It does not follow a fold, a rotation or a
  2in1 window resize, so the seek geometry is stale after any of those even
  once `HW-13-0055` is fixed. Read the container's own width instead.
- The tab strip sets `.scrollable(false)` and three of the four tabs render a
  single placeholder `Text`; only the short-video tab is implemented.
- `TAB_INTERVAL_NUMBER` (30 vp) is the total horizontal inset the bar reserves.
  It is subtracted in the mapping but the bar itself has no matching padding,
  so the geometry is approximate at both ends.
- Emoji rendering depends on the system font's colour-emoji coverage; the
  three animal and three gift glyphs in `emojis` are not guaranteed on every
  device profile.

## Pitfalls

- **`HW-13-0055`** (B/medium, confirmed): the playback→pixel mapping in
  `onUpdate` uses a hardcoded `430 - 30` while the drag clamp and the
  pixel→time mapping use `this.screenW - 30`. On any device that is not 430 vp
  wide the bar overflows or underfills and a drag released mid-screen seeks to
  a different timestamp than the bar shows. Fix: use `screenW` in both
  directions.
- **`HW-13-0056`** (B/low, confirmed): `doRandBySync` divides the random byte
  by 255, making the range inclusive of 1.0, so `Math.floor(1.0 * 18)` indexes
  past the end of the 18-element `emojis` array and `fillText` draws the string
  `undefined`. One tap in 256. Fix: divide by 256.
- **Unconditional 16 ms interval** (observation, no HW id): see Constraints.
  The `this.likeIcons = newIcons` assignment additionally dirties a `@State`
  variable that no `build()` reads.
- **Crypto RNG for decoration** (observation, no HW id): five
  `cryptoFramework.createRandom().generateRandomSync(1)` calls per particle
  where `Math.random()` would do.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component and `RenderingContextSettings`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `clearRect`, `globalAlpha`, `save`/`restore`, `translate`, `scale`, `fillText`, `textAlign`, `textBaseline`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `VideoController.setCurrentTime`, `SeekMode`, `onUpdate`, `onPrepared`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `scrollable`, `tabBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createRandom`, `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `huawei_industry_tree/13_media_entertainment/docs/22_video_like.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_like-0000002332695009
