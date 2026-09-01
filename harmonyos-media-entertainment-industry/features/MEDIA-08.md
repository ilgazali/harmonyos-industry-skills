---
id: MEDIA-08
title: Landscape/portrait video page - sensor rotation plus one orientation number driving the layout
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/08_orientation_switching.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/orientation_switching-0000002255691184
sample: huawei_industry_tree/13_media_entertainment/downloads/OrientationSwitch.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["window.getLastWindow", setPreferredOrientation, "window.Orientation.AUTO_ROTATION_UNSPECIFIED", "on('windowSizeChange')", "off('windowSizeChange')", "on('avoidAreaChange')", getWindowAvoidArea, setWindowLayoutFullScreen, "display.getDefaultDisplaySync", "display.Orientation", Video, VideoController, Slider, SliderChangeMode, Flex, FlexDirection, RelativeContainer, "@Prop", "@Link", "@StorageProp", "UIContext.px2vp", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0026, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a page has **two genuinely different layouts for the two
screen orientations** and the switch is driven by the sensor rather than by a
fullscreen button. A video detail page is the canonical case: portrait shows a
16:9 player above a scrolling description, landscape shows the player filling
80% of the width with the description squeezed into a 20% rail.

Two APIs do the work and they are easy to confuse. `setPreferredOrientation`
tells the **window** which orientations it is allowed to rotate to;
`display.getDefaultDisplaySync().orientation` tells the **page** which one it
is currently in. The first is permission to rotate, the second is the state
you lay out against. Between them sits a single `@State orientation: number`,
refreshed from a `windowSizeChange` subscription, and every responsive
attribute on the page is a ternary on it.

The technique generalises well past video: map navigation, e-readers, games,
any 2in1 window that can be dragged from tall to wide. The part specific to
media is which orientation mode you ask for - a player that must be landscape
uses `AUTO_ROTATION_LANDSCAPE` (see `MEDIA-05`), a player that supports both
uses `AUTO_ROTATION_UNSPECIFIED`, as here.

## Feature checklist

- The video page opens in portrait: a 212 vp player, then tabs, a title, an
  episode list and two image grids below it.
- Rotating the device turns the page landscape without recreating it: the
  player takes 80% of the width, the description takes the remaining 20% and
  scrolls independently.
- The rotation follows the sensor and obeys the system rotation lock.
- The player's control bar is taller in landscape and its horizontal padding
  grows, so the controls do not sit under the rounded corners.
- In landscape the control bar's bottom padding grows by the navigation
  indicator height.
- One tab of the description strip is dropped in landscape, where there is no
  room for it.
- Play/pause, a scrub bar with elapsed and total time, and a fullscreen icon.
- Scrubbing seeks only when the drag ends, not on every intermediate value.
- Leaving the page unsubscribes from the size-change listener.

## Architecture

One `entry` module, two components, one page, one util. No player service: the
`Video` component owns the playback.

```
entry/src/main/ets
├── components/IntroductionComponent.ets  the description rail: tabs, episodes, two image grids
├── components/VideoComponent.ets         the Video + the control bar + timeFormat
├── constants/CommonConstants.ets         every layout number, plus `export const PORTRAIT = 0`
├── entryability/EntryAbility.ets         full screen; windowClass + avoid areas (in vp) -> AppStorage
├── model/VideoInfoModel.ets              TABS/ICONS/LIST_NUM/LABELS/BC_IMG resource arrays
├── pages/VideoPlayback.ets               @Entry: owns `orientation`, switches the Flex direction
└── utils/OrientationUtil.ets             setOrientation(): getLastWindow + setPreferredOrientation
```

The documented tree matches the zip, and the document's constraints (API 20,
HarmonyOS 6.0.0, DevEco 6.0.0) match `compatibleSdkVersion: "6.0.0(20)"` -
neither `HW-13-0003` nor `HW-13-0004` applies here. Note the struct inside
`VideoPlayback.ets` is named `Index`, not `VideoPlayback`; harmless, since the
file path is what `loadContent` resolves, but it is the kind of drift that
makes a tree listing misleading.

**The design decision worth copying** is that orientation is read **once**, in
one place, into one number, and passed down as a `@Prop`:

```typescript
@State orientation: number = 0;
// ...
VideoComponent({ orientation: this.orientation, isPlaying: this.isPlaying });
IntroductionComponent({ orientation: this.orientation });
```

Neither child subscribes to anything. They receive a number and every
responsive attribute is `this.orientation !== PORTRAIT ? land : portrait`.
That means there is exactly one `windowSizeChange` subscription in the whole
app, one place to unsubscribe, and no chance of two components disagreeing
about the current orientation mid-frame.

**The part worth tightening** is the comparison itself. `PORTRAIT` is `0` and
the test everywhere is `!== PORTRAIT`, but `display.Orientation` has four
values: `PORTRAIT` 0, `LANDSCAPE` 1, `PORTRAIT_INVERTED` 2,
`LANDSCAPE_INVERTED` 3. Reverse portrait therefore renders the **landscape**
layout on any device that allows it. Test for the landscape values, or fold
the enum into a boolean `isLandscape` once.

## Implementation steps

1. **Declare the orientation policy in `module.json5`**:
   `"orientation": "auto_rotation_unspecified"` on the ability, so the page is
   rotatable from launch and not only after the first `aboutToAppear`.
2. **Publish the main window and the avoid areas** from `onWindowStageCreate`.
   This sample converts to vp **in the ability** (`context.px2vp(...)` before
   `AppStorage.setOrCreate`), so pages use the values directly - the opposite
   convention to `MEDIA-06`. Pick one per project and stay with it.
3. **Set the preferred orientation from the page** via
   `window.getLastWindow(context.getHostContext(), cb)` and
   `setPreferredOrientation(window.Orientation.AUTO_ROTATION_UNSPECIFIED)`.
   Mode 12 rotates with the sensor **under the control of the Control Panel
   rotation switch**, and lets the system decide which orientations a given
   device allows - typically excluding reverse portrait on phones.
4. **Subscribe to `windowSizeChange` in `aboutToAppear`** and re-read
   `display.getDefaultDisplaySync().orientation` inside the callback. The
   window is the thing that resizes; the display is the thing that knows which
   way round it is.
5. **Unsubscribe in `aboutToDisappear`.** The sample does, which is the right
   habit and not the norm across this industry's samples - but pass the handler
   reference rather than calling `off` bare (see Constraints).
6. **Switch the container axis, not the children**: one `Flex` whose
   `direction` is `FlexDirection.Row` in landscape and `FlexDirection.Column`
   in portrait, with percentage widths on the two columns.
7. **Add the navigation-indicator height to the control bar's bottom padding in
   landscape only**, where the bar sits against the system gesture area.
8. **Format times with hours *subtracted* from minutes** (`HW-13-0026`).
9. **Seek on `SliderChangeMode.End`**, tracking the value locally in between,
   so a drag does not fire a seek per frame.

## Verified snippets

All snippets are from `OrientationSwitch.zip`. Corrected forms are marked.

**Asking for rotation — `entry/src/main/ets/utils/OrientationUtil.ets`** (as shipped)

```typescript
import { window } from '@kit.ArkUI';

export function setOrientation(context: UIContext) {
  try {
    window.getLastWindow(context.getHostContext(), (err, data) => {   // 获取window实例
      if (err.code) {
        hilog.error(0xFF00, TAG, `Failed to obtain the top window. Cause: ${JSON.stringify(err)}`);
        return;
      }
      let windowClass = data;
      let orientation = window.Orientation.AUTO_ROTATION_UNSPECIFIED; // 设置窗口方向旋转模式
      try {
        windowClass.setPreferredOrientation(orientation, (err) => {
          if (err.code) {
            hilog.error(0xFF00, TAG, `Failed to set window orientation. Cause: ${JSON.stringify(err)}`);
            return;
          }
        });
      } catch (exception) {
        hilog.error(0xFF00, TAG, `Failed to set window orientation. Cause: ${JSON.stringify(exception)}`);
      }
    });
  } catch (exception) {
    hilog.error(0xFF00, TAG, `Failed to obtain the top window. Cause: ${JSON.stringify(exception)}`);
  }
}
```

**The choice of enum value is the whole decision.** `AUTO_ROTATION` (5) rotates
to all four orientations and **ignores** the user's rotation lock;
`AUTO_ROTATION_RESTRICTED` (8) rotates to all four but obeys it;
`AUTO_ROTATION_UNSPECIFIED` (12), used here, obeys the lock *and* lets the
system pick the permitted set - on a typical phone that means portrait,
landscape and reverse landscape but not reverse portrait. For a video page
that is the right default: a user who has locked rotation does not want a
player overriding it.

The doubled try/catch is not defensive noise. `getLastWindow` and
`setPreferredOrientation` both report failures two ways - a synchronous throw
for a bad parameter, and `err.code` in the callback for a runtime failure such
as `1300002` (abnormal window state) - so each needs both. Also note what the
reference says about when this call does nothing: in freeform window mode it
neither takes effect nor returns an error, and it applies only to the **main**
window.

The function takes a `UIContext` and immediately calls `getHostContext()`,
which is why it is callable from a page with no ability reference.

**One number drives the layout — `entry/src/main/ets/pages/VideoPlayback.ets`** (as shipped)

```typescript
@State orientation: number = 0;
private context: UIContext = this.getUIContext();
private windowClass: window.Window | undefined = undefined;

aboutToAppear() {
  setOrientation(this.context);
  // 监听横竖屏变化
  this.windowClass = AppStorage.get('windowClass');
  this.windowClass?.on('windowSizeChange', () => {
    const DEFAULT_DISPLAY = display.getDefaultDisplaySync();
    this.orientation = DEFAULT_DISPLAY.orientation;
  });
}

aboutToDisappear() {
  this.windowClass?.off('windowSizeChange');
}

build() {
  Flex({
    direction: this.orientation !== PORTRAIT ? FlexDirection.Row : FlexDirection.Column
  }) {
    Column() {
      VideoComponent({ orientation: this.orientation, isPlaying: this.isPlaying });
    }
    .width(this.orientation !== PORTRAIT ? CommonConstants.VIDEO_WIDTH_LAND : CommonConstants.FULL_SIZE)
    .height(this.orientation !== PORTRAIT ? CommonConstants.FULL_SIZE : CommonConstants.VIDEO_HEIGHT);

    Column() {
      IntroductionComponent({ orientation: this.orientation });
    }
    .width(this.orientation !== PORTRAIT ? CommonConstants.INTRODUCTION_WIDTH_LANDSCAPE : CommonConstants.FULL_SIZE)
    .clip(true);
  }
  .height(this.orientation !== PORTRAIT ? CommonConstants.FULL_SIZE : 'auto')
  .padding({
    top: this.orientation !== PORTRAIT ? 0 : this.topRectHeight,
  });
}
```

**The window signals, the display answers.** `windowSizeChange` is the event
that actually fires on rotation (and on a 2in1 window resize, and on entering
split-screen), but its payload is a size, not an orientation - so the handler
ignores it and asks `display.getDefaultDisplaySync()` instead. That also makes
the callback idempotent: any size change re-reads the truth.

`.height('auto')` in portrait against `FULL_SIZE` in landscape is the detail
that makes the same `Flex` work in both: in portrait the page is a column that
grows with its content and the description scrolls the page, in landscape it is
a fixed-height row whose second column scrolls internally. And the top padding
is applied **only in portrait** - in landscape the status bar avoid area is on
the side, so padding the top would leave a black band.

`topRectHeight` is used raw, without `px2vp`, because `EntryAbility` already
converted it:

```typescript
let topRectHeight = context.px2vp(cutoutArea.topRect.height);
AppStorage.setOrCreate<number>('topRectHeight', topRectHeight);
```

That is the opposite of `MEDIA-06`, which stores px and converts in the page.
Both work; mixing them within one project does not.

One caveat on the teardown: `off('windowSizeChange')` with no handler argument
removes **every** subscriber for that event on that window. It is correct here
because there is exactly one, but in a multi-page app pass the handler
reference.

**The time formatter — `entry/src/main/ets/components/VideoComponent.ets`** (corrected, see `HW-13-0026`)

```typescript
const CONVERSION = 60;

// 视频时间格式化
timeFormat(time: number) {
  let hoursStr = Math.floor(time / (CONVERSION * CONVERSION)).toString();
  let minutesStr = Math.floor((time % (CONVERSION * CONVERSION)) / CONVERSION).toString();  // FIX
  let secondsStr = Math.floor(time % CONVERSION).toString();

  if (minutesStr.length === 1) {
    minutesStr = '0' + minutesStr;
  }
  if (secondsStr.length === 1) {
    secondsStr = '0' + secondsStr;
  }
  return `${hoursStr}:${minutesStr}:${secondsStr}`;
}
```

The shipped line is `Math.floor(time / CONVERSION)` with the hours never
subtracted, so the minutes field counts total minutes: 3661 seconds renders as
`1:61:01`. The seconds field is correct because it already uses `%`, which is
what makes the omission easy to miss in review - two of the three components
are right.

This helper is presented as reusable and is the general form `MEDIA-05` lacks
(that sample concatenates `'00:' + seconds`, correct only below one minute).
Fix it here and it is worth lifting into a shared util. Note it always renders
an hours field, so a three-minute clip shows `0:03:12`; pad or drop the hours
depending on the design.

**The responsive control bar — same file** (as shipped)

```typescript
Slider({
  value: this.curTime,
  step: 1,
  min: 0,
  max: this.videoDuration,
  style: SliderStyle.OutSet
})
  .layoutWeight(1)
  .trackThickness(5)
  .blockSize({ width: 12, height: 12 })
  .onChange((value, mode) => {
    this.curTime = value;
    if (mode === SliderChangeMode.End) {
      this.videoController.setCurrentTime(value);      // seek once, on release
    }
  });
// ...
.height(this.orientation !== PORTRAIT ? CommonConstants.VIDEO_CONTROL_LAND_HEIGHT :
  CommonConstants.VIDEO_CONTROL_HEIGHT)
.padding({
  left: this.orientation !== PORTRAIT ? CommonConstants.PADDING_CONTROLS_LAND_HORIZONTAL :
    CommonConstants.PADDING_CONTROLS_HORIZONTAL,
  right: this.orientation !== PORTRAIT ? CommonConstants.PADDING_CONTROLS_LAND_HORIZONTAL :
    CommonConstants.PADDING_CONTROLS_HORIZONTAL,
  top: CommonConstants.PADDING_CONTROLS_VERTICAL,
  bottom: this.orientation !== PORTRAIT ?
    CommonConstants.PADDING_CONTROLS_VERTICAL + this.bottomRectHeight :   // avoid the indicator
    CommonConstants.PADDING_CONTROLS_VERTICAL
});
```

**`SliderChangeMode.End` is what separates a scrub bar from a stutter.**
`onChange` fires continuously while the thumb moves; assigning `curTime` keeps
the bar and the elapsed caption following the finger, and the seek is issued
once, when the drag ends. Compare `MEDIA-05`, which seeks on every value with
`SeekMode.Accurate` - fine for a 15-second demo clip, expensive for a real one.

`max: this.videoDuration` comes from `Video.onPrepared`, so the bar is bound to
the actual media rather than a constant. `layoutWeight(1)` lets the bar absorb
whatever the icons and the two time captions leave, which is what keeps the row
correct at both control-bar widths.

The bottom padding is the one place the avoid area reaches this component: in
landscape the bar sits at the bottom of a full-screen surface where the
navigation indicator lives, so `bottomRectHeight` (already vp) is added; in
portrait the bar is inside the 212 vp player box and needs nothing.

Note also `IntroductionComponent`, which drops a tab entirely in landscape:

```typescript
ForEach(TABS, (item: string, index) => {
  if (!(this.orientation !== PORTRAIT && index === 1)) {
    Text(item) /* ... */
  }
});
```

Hiding rather than shrinking is the right call for a 20%-wide rail - but the
`curTab` index is not remapped, so a user who selects tab 2 in portrait keeps a
selection that has no visible chip in landscape.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the video is a bundled
rawfile (`video3.mp4`) and orientation is a window property.

```json5
{
  "name": "EntryAbility",
  "srcEntry": "./ets/entryability/EntryAbility.ets",
  "orientation": "auto_rotation_unspecified"
}
```

The declared orientation and the `setPreferredOrientation` call ask for the
same thing, which is deliberate rather than redundant: the manifest value
applies from the splash screen, the API call applies from the page. If a
project only sets it in code, the launch is locked to the default until the
first page runs.

`build-profile.json5` enables `strictMode` with `caseSensitiveCheck: true` and
`useNormalizedOHMUrl: true` - which is exactly why the systematic tree-naming
defect `HW-13-0003` matters in this industry: under this flag a file referenced
with the wrong case does not resolve.

There is no `AVPlayer` here at all - the `Video` component owns its player -
so the systematic `HW-13-0005` (five media samples that never call `release()`)
has nothing to bite on. That is itself a reason to prefer `Video` when the
requirement is "play this file in this box".

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are `6.0.0(20)`, and the document agrees.
- `setPreferredOrientation` takes effect only on the **main** window and only
  outside freeform window mode; elsewhere it silently does nothing.
- `AUTO_ROTATION_UNSPECIFIED` obeys the Control Panel rotation switch, so on a
  device with rotation locked the page will not turn. That is intended, but it
  means the feature cannot be demonstrated with the lock on.
- The layout tests `orientation !== PORTRAIT`, so `PORTRAIT_INVERTED` (2)
  renders the landscape layout on devices that permit reverse portrait.
- `off('windowSizeChange')` is called without a handler and removes all
  subscribers for the event on that window.
- `windowClass` is read from `AppStorage` with optional chaining; if
  `EntryAbility` has not stored it the subscription is silently skipped and the
  page never becomes responsive.
- The description content is static resource arrays in `VideoInfoModel.ets`;
  the tabs, episode list and grids are not wired to any data.

## Pitfalls

- **`HW-13-0026`** (B/low, confirmed): `timeFormat` computes minutes as
  `Math.floor(time / 60)` without subtracting the hours, so anything from one
  hour renders as `1:61:01`. The helper is presented as reusable, which makes
  the defect portable. Fix: `Math.floor((time % 3600) / 60)`.
- **`HW-13-0003`** (E/low, confirmed, systematic - **does not apply here**):
  this document's tree matches the zip. Cited as the contrast case: sibling
  media docs list `Entryability.ets`, `logger.ets` and `index.d.ets` for files
  that are actually `EntryAbility.ets`, `Logger.ets` and `Index.d.ts`, and this
  project's `caseSensitiveCheck: true` is what turns that into a build failure.
- **`HW-13-0004`** (E/low, confirmed, systematic - **does not apply here**):
  this document's API-20 constraint matches the sample's `6.0.0(20)`. Three
  other docs in this industry claim API 20 over samples built for 5.0.4(16) and
  5.0.5(17); check the build profile before trusting a 约束与限制 block.
- **Unfiled, but worth knowing**: reverse portrait falls into the landscape
  branch; `off('windowSizeChange')` drops all handlers; and
  `IntroductionComponent` hides the tab at index 1 in landscape without
  remapping `curTab`, so a selection made in portrait can leave no chip
  highlighted.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setPreferredOrientation`, `on('windowSizeChange')`, `getWindowAvoidArea`, `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` - the `Orientation` enum, including `AUTO_ROTATION_UNSPECIFIED` (12) and how it differs from `AUTO_ROTATION` and `AUTO_ROTATION_RESTRICTED`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getDefaultDisplaySync` and `display.Orientation` (PORTRAIT 0, LANDSCAPE 1, PORTRAIT_INVERTED 2, LANDSCAPE_INVERTED 3)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `onPrepared`/`onUpdate`, `VideoController.setCurrentTime`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderChangeMode`, `SliderStyle.OutSet`, `blockSize`/`trackThickness` proportions
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-guides/03_application-framework/multi-window-layout-adapt.md` - laying out against a changing window size
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/multi-window-layout-adapt
- `MEDIA-05` - the landscape-only player (`auto_rotation_landscape`) and its weaker time formatting
- `MEDIA-06` - the other avoid-area convention: px in `AppStorage`, converted at the point of use
