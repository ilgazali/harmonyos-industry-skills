---
id: MEDIA-05
title: Player gesture sliders - thumbless vertical Sliders for brightness and volume over a video
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/05_custom_slider.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_slider-0000002284049981
sample: huawei_industry_tree/13_media_entertainment/downloads/CustomSlider.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@ohos.multimedia.avVolumePanel"]
apis: [Slider, SliderStyle, "Axis.Vertical", reverse, trackThickness, trackColor, selectedColor, selectedBorderRadius, blockSize, AVVolumePanel, Video, VideoController, "windowStage.getMainWindowSync", "window.setWindowBrightness", setWindowLayoutFullScreen, setWindowSystemBarProperties, setSpecificSystemBarEnabled, "@Provide", "@Consume", "UIContext.px2vp", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0017, HW-13-0003]
status: verified-with-fixes
---

## When to use

Load this card when you are building the **gesture layer of a video player** -
the two invisible vertical tracks that raise brightness on the left half of the
frame and volume on the right. The pattern is a `Slider` with
`style: SliderStyle.NONE`, `direction: Axis.Vertical` and `reverse: true`,
stretched over the video and styled until it looks like a bar rather than a
form control.

The transferable idea is that **a slider does not have to look like a slider**.
`SliderStyle.NONE` drops the thumb entirely, leaving a track whose selected
portion is the whole affordance; from there `trackThickness`,
`selectedBorderRadius` and a translucent `trackColor` turn a stock component
into the fat rounded bar every player uses. The same recipe covers subtitle
opacity, danmaku transparency, playback-speed strips and EQ bands - anything
where the value is continuous, the control is transient, and a visible thumb
would be noise.

The second half of the card is what each slider is wired to, and the two ends
are not symmetric. Brightness is a **window** property the app owns outright
(`setWindowBrightness`). Volume is a **system** property the app cannot write:
the sample publishes a number and lets `AVVolumePanel` do the actual change.
Read that distinction before assuming a volume slider can just set a value.

## Feature checklist

- A full-screen black page in forced landscape, with a `Video` filling most of
  the frame and no built-in controls.
- A brightness column on the left: an icon, a live numeric readout, a `+`, a
  vertical bar, a `-`.
- A volume column on the right with the same structure.
- Dragging the brightness bar upward brightens the screen immediately, and the
  readout follows.
- Dragging the volume bar raises the system volume through the system volume
  panel, which appears while the value changes.
- A bottom control row: play/pause toggle, elapsed and total time, a horizontal
  progress `Slider`, a fullscreen icon.
- Seeking with the progress bar jumps the video accurately.
- The video starts on page show and pauses on page hide.

## Architecture

One `entry` module, three ArkUI files and a constants class. There is no model
layer - the state is four numbers and a boolean.

```
entry/src/main/ets
├── common/Constants.ets              CommonConstants + TimeObject/DurationObject event shapes
├── entryability/EntryAbility.ets     full screen, black status bar, windowStage -> AppStorage
├── entrybackupability/
└── pages/HomePage.ets                @Entry HomePage + the reusable VerticalSliderItem struct
```

**The documented tree does not match the zip**: the document's 工程目录 lists
the page under `page/`, while the project ships `pages/`. It is the same class
of transcription defect as the systematic `HW-13-0003`, which catches
`Entryability.ets`, `logger.ets` and `index.d.ets` in sibling media docs. This
project's `build-profile.json5` does not enable `caseSensitiveCheck`, so the
mismatch is documentation-only here - but the sibling projects that do enable
it would fail to resolve.

The document's constraint block (API 20, HarmonyOS 6.0.0, DevEco 6.0.0) agrees
with `compatibleSdkVersion: "6.0.0(20)"` in the zip. That is worth stating
because three other docs in this industry do not (`HW-13-0004`).

**The design decision worth copying** is that `VerticalSliderItem` is one
component used twice, parametrised by a `mode` string, and the two instances
differ only in the four numbers that a brightness scale and a volume scale
disagree about: max (`1.0` vs `15`), step (`0.01` vs `1`), icon size and
horizontal offset. Everything visual - the track colour, the thickness, the
corner radius, the `+`/`-` captions - is written once:

```typescript
VerticalSliderItem({
  sliderValue: this.lightness,
  icon: $r('app.media.lightness'),
  imageSize: CommonConstants.VERTICALSLIDER_IMAGESIZE_BRIGHTNESS,
  distance: CommonConstants.VERTICALSLIDER_DISTANCE_BIGHTNESS,   // -700: pushed left
  mode: CommonConstants.VERTICALSLIDER_MODE_BIGHTNESS
});
```

The part **worth changing** is the discriminator. `mode: string` compared
against the literals `'bright'` and `'volume'` inside `onChange` means the
compiler cannot tell you when a third mode (saturation, subtitle opacity) is
added and a branch is missed. A union type, or a config object carrying
`max`/`step`/`apply`, removes the three ternaries and the string comparison at
once.

## Implementation steps

1. **Force the orientation in `module.json5`**, not in code:
   `"orientation": "auto_rotation_landscape"` on the ability makes the player
   landscape-only from the splash screen onward.
2. **Publish the `WindowStage` from the ability** into `AppStorage` in
   `onWindowStageCreate`, and set full-screen layout plus the black status bar
   there. The page then only reads.
3. **Build the slider column** as `Icon / value / + / Slider / -` in a
   `Column`, with `Blank().height(...)` as the spacer rather than margins, so
   the column keeps its rhythm at any track height.
4. **Style the `Slider` into a bar**: `style: SliderStyle.NONE` removes the
   thumb, `direction: Axis.Vertical` with `reverse: true` makes up mean more,
   and `trackThickness` plus `selectedBorderRadius` give the rounded pill.
   Under `SliderStyle.NONE` the default `selectedBorderRadius` is `0` and
   `blockSize` has no effect, so both must be set or skipped deliberately.
5. **Route `onChange` by mode** - brightness to `setWindowBrightness`, volume
   to the shared `@Provide` value - and always write `sliderValue` back so the
   readout tracks the drag.
6. **Let `AVVolumePanel` own the actual volume.** An application cannot set
   system volume directly; it hands the panel a target level and the panel
   shows the system UI while changing it.
7. **Never write `zIndex(Number($r('app.float.…')))`** (`HW-13-0017`). `$r()`
   returns a descriptor object, `Number()` of it is `NaN`, and `zIndex(NaN)`
   silently falls back to `0`. Use a numeric constant, as the sample already
   does for the control row.
8. **Derive the elapsed/total captions from the duration**, not from string
   concatenation with a fixed `'00:'` prefix.

## Verified snippets

All snippets are from `CustomSlider.zip`. Corrected forms are marked.

**The thumbless vertical slider — `entry/src/main/ets/pages/HomePage.ets`** (as shipped)

```typescript
Slider({
  value: this.sliderValue,
  min: CommonConstants.VERTICLASILDER_MIN,                     // 0.0 for both modes
  max: this.mode === 'bright' ? CommonConstants.VERTICLASILDER_MAX_BRIGHTNESS :   // 1.0
    CommonConstants.VERTICLASILDER_MAX_VOLUME,                                    // 15
  direction: Axis.Vertical,
  reverse: true,
  style: SliderStyle.NONE,
  step: this.mode === 'bright' ? CommonConstants.VERTICLASILDER_STEP_BRIGHTNESS : // 0.01
    CommonConstants.VERTICLASILDER_STEP_VOLUME                                    // 1
})
  .trackColor($r('app.string.trackcolor'))            // '#94fffefe' - translucent white
  .trackThickness($r('app.float.trackthickness'))     // 13
  .selectedColor($r('app.string.selectedcolor'))      // '#beffffff'
  .selectedBorderRadius($r('app.float.borderradius')) // 10
  .height($r('app.float.slider_height'))              // 120
  .onChange((value) => {
    if (this.mode === 'bright') {
      this.changeLightness(value);
    } else if (this.mode === 'volume') {
      this.volume = value;                            // @Consume - drives AVVolumePanel
    }
    this.sliderValue = value;
  });
```

**Four options carry the design.** `style: SliderStyle.NONE` removes the thumb,
which is what turns a form control into a player bar. `direction: Axis.Vertical`
plus `reverse: true` is a pair: a vertical slider defaults to *min at the top*,
so without `reverse` dragging up would dim the screen. `trackThickness: 13`
against a 120 vp track is the aspect ratio that reads as a bar rather than a
line - note that the reference's `trackThickness : blockSize` guidance does not
apply here because `NONE` ignores `blockSize` altogether. And
`selectedBorderRadius: 10` is not decoration: under `SliderStyle.NONE` its
default is `0`, so a square-ended fill is what you get if you forget it.

The two ranges are genuinely different scales - brightness is a float in
`[0.0, 1.0]` and volume is an integer step count up to 15 - which is why `max`
and `step` are the two values the mode ternary has to switch. The readout above
the bar therefore has to switch too: `Math.floor(this.sliderValue * 10)` for
brightness, plain `Math.floor` for volume.

**Brightness: the window is the sink — same file** (as shipped)

```typescript
@Component
struct VerticalSliderItem {
  @State sliderValue: number = 0;
  distance: number = 0;
  mode: string = '';
  @Consume volume: number;
  windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;
  mainWin: window.Window = this.windowStage.getMainWindowSync();

  changeLightness(value: number) {
    this.mainWin.setWindowBrightness(value);
  }
  // ...
}
```

`setWindowBrightness` takes a float in `[0.0, 1.0]`, which is exactly the
slider's range, so no conversion is needed - and `-1.0` is the documented way
to hand control back to the system. Two properties of the API shape this
design: the brightness applies **only while the window is foreground and
focused**, and it is reset when the window goes to background. That is why a
player sets it per session and never persists it.

The two field initialisers deserve a warning. `AppStorage.get('windowStage')`
is cast with `as` and then `getMainWindowSync()` is called on it at field-init
time, before `aboutToAppear`. It works here only because `EntryAbility` writes
the key inside `onWindowStageCreate` before `loadContent` resolves. Any code
path that renders this component without that write - a preview, a second
ability, a unit test - throws on construction. A nullable field plus a lazy
getter costs two lines and removes the ordering dependency.

**Volume: the system panel is the sink — same file** (corrected, see `HW-13-0017`)

```typescript
@Provide volume: number = 5;                       // HomePage owns it, the sliders @Consume it

AVVolumePanel({
  volumeLevel: this.volume
})
  .zIndex(CommonConstants.ROW_CONTROLLER_ZINDEX)   // FIX: was Number($r('app.float.avvolumepanel_zindex')) -> NaN
  .width($r('app.string.avvolumepanel_width'));    // '30%'
```

**An application cannot set the system volume.** The `avVolumePanel` reference
is explicit about it: the app invokes the system volume panel and the *user*
adjusts the volume, with a system prompt UI shown so the change is never
silent. So the volume slider's `onChange` does not call an audio API at all -
it writes `this.volume`, the `@Provide`/`@Consume` link pushes the new value
into `AVVolumePanel`'s `@Prop volumeLevel`, and the panel performs and
announces the change. Values outside the device's range are clamped by the
panel rather than rejected.

The `zIndex` is the corrected part. `$r('app.float.avvolumepanel_zindex')`
resolves to a `Resource` descriptor object, `Number(object)` is `NaN`, and
`zIndex(NaN)` degrades to `0`. `float.json` defines `video_zindex: 1`,
`avvolumepanel_zindex: 3` and `verticalslideritem_zindex: 3`, none of which
ever reach the layout - the stack currently works only because declaration
order inside the `Stack` happens to match the intended depth. Reorder the four
children and the video covers the sliders. Numeric constants (as
`ROW_CONTROLLER_ZINDEX` already is) or `resourceManager.getNumber` are the two
correct forms.

**The progress row — same file** (as shipped)

```typescript
Text(this.videoValue < 10 ? '00:0' + String(this.videoValue) : '00:' + String(this.videoValue))
  .fontSize($r('app.float.text_progress_fontsize'))
  .fontColor(Color.White);
Text('00:' + String(this.duration))
  .fontSize($r('app.float.text_progress_fontsize'))
  .fontColor(Color.White);

Slider({
  value: $$this.videoValue,
  min: CommonConstants.SLIDER_PROGRESS_MIN,     // 0
  max: CommonConstants.SLIDER_PROGRESS_MAX,     // 15 - the bundled clip's length
  direction: Axis.Horizontal
})
  .width(this.getUIContext().px2vp(CommonConstants.SLIDER_PROGRESS_WIDTH))
  .trackThickness($r('app.float.slider_trackthickness'))
  .blockSize({ width: $r('app.float.slider_blocksize'), height: $r('app.float.slider_blocksize') })
  .onChange((value: number) => {
    this.controller.setCurrentTime(value, SeekMode.Accurate);
  });
```

Two things here are worth copying and one is not. `$$this.videoValue` is the
two-way binding that lets `Video.onUpdate` push the playhead into the slider
while the user's drag pushes it back out - without `$$` the bar would fight the
playback. `SeekMode.Accurate` is the right mode for a scrub bar: the cheaper
key-frame modes land the user somewhere near, not where they let go.

What is not worth copying is the time formatting. `'00:' + String(seconds)`
is correct only below one minute, and `max: 15` is the bundled clip's length
hardcoded into a constant rather than taken from `onPrepared`'s `duration`.
Compare `MEDIA-08`, whose `timeFormat` helper is the general form - and which
has its own modulo defect (`HW-13-0026`).

## Permissions & config

**None.** The sample declares no `requestPermissions`, which is correct:
window brightness is a property of the app's own window, and system volume is
changed by the system panel on the user's behalf, not by the app.

The ability's orientation is pinned declaratively:

```json5
{
  "name": "EntryAbility",
  "srcEntry": "./ets/entryability/EntryAbility.ets",
  "orientation": "auto_rotation_landscape"   // 随传感器横屏旋转 - landscape both ways
}
```

`auto_rotation_landscape` rotates with the sensor between landscape and reverse
landscape only, and is not controlled by the Control Panel rotation switch.
For a player that can also run portrait, see `MEDIA-08`, which uses
`auto_rotation_unspecified` instead.

`EntryAbility` also hides the navigation indicator and keeps the status bar
with a black background and light icons - the standard immersive player setup:

```typescript
windowClass.setWindowLayoutFullScreen(true);
windowClass.setWindowSystemBarProperties({ statusBarColor: '#ff000000', isStatusBarLightIcon: true });
windowClass.setSpecificSystemBarEnabled('navigationIndicator', false);
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  and the document agrees.
- `AVVolumePanel` is available from API 12. It **displays no UI on wearables**
  (the volume still changes) and **is not supported on TVs** - a player
  targeting either needs its own volume affordance.
- `setWindowBrightness` has no effect on TVs, and on non-2in1 devices before
  HarmonyOS 6.1.0 it blocks Control Panel brightness changes while active.
- The progress bar's range is a hardcoded 15, matching `product.mp4`. Any other
  clip needs `max` bound to the `onPrepared` duration.
- The slider offsets (`-700` / `710` vp) and the control row's `-54` top margin
  are absolute numbers tuned for one screen size. On a tablet or a resized 2in1
  window the two columns will not sit over the video's edges.
- `en_US` and `zh_CN` resource directories exist but carry only the module
  boilerplate strings; every user-visible string lives in `base`.

## Pitfalls

- **`HW-13-0017`** (B/low, confirmed): `Number($r('app.float.*_zindex'))` is
  always `NaN`, so all three resource-driven `zIndex` values silently become
  `0` and the intended stacking survives only because declaration order happens
  to match. Fix: use numeric constants or `resourceManager.getNumber`.
- **`HW-13-0003`** (E/low, confirmed, systematic): the media docs' project
  trees misspell real files. Here the page directory is documented as `page/`
  and shipped as `pages/`; sibling docs carry `Entryability.ets`, `logger.ets`
  and `index.d.ets` for files that are actually `EntryAbility.ets`,
  `Logger.ets` and `Index.d.ts`. Fix: regenerate the trees from the zips.
- **Unfiled, but worth knowing**: `windowStage` is read from `AppStorage` and
  `getMainWindowSync()` is called in a field initialiser, so the component
  construction depends on `EntryAbility` having run first; and the elapsed/total
  captions are `'00:' + seconds`, which breaks above one minute.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderStyle`, `reverse`, `trackThickness`, `selectedBorderRadius` defaults per style
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/04_media/ohos-multimedia-avvolumepanel.md` - `AVVolumePanel`, `volumeLevel`, the wearable/TV restrictions
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-multimedia-avvolumepanel
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowBrightness`, `setWindowLayoutFullScreen`, `setSpecificSystemBarEnabled`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` - `getMainWindowSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `VideoController.setCurrentTime`, `SeekMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `MEDIA-08` - the general `timeFormat` helper this sample lacks, and the portrait/landscape variant of the same player
