---
id: UTIL-26
title: LED banner - a full-screen landscape Marquee with a measured space-padding trick
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/26_led_scroll_text.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/led_scroll_text-0000002366978977
sample: huawei_industry_tree/15_utilities/downloads/SimpleLed.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Marquee, "window.setWindowLayoutFullScreen", "window.setPreferredOrientation", "window.setWindowSystemBarEnable", "window.getWindowAvoidArea", "display.getDefaultDisplaySync", "UIContext.getMeasureUtils", measureText, Canvas, CanvasRenderingContext2D, createLinearGradient, "@CustomDialog", CustomDialogController, expandSafeArea]
min_api: 20
modules: [entry]
findings: [HW-15-0060, HW-15-0061, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the app has a **"show this on the screen so other people can
read it" mode** - a concert banner, a birthday message held up across a room, a
queue number, a stage prompt. The screen stops being a UI and becomes a display
device: landscape, no system bars, one enormous line of text scrolling forever.

The pattern is a configuration page and a display page in one component,
switched by a single `isFullScreen` boolean, with the window itself
reconfigured on the transition: `setWindowLayoutFullScreen(true)`,
`setPreferredOrientation(LANDSCAPE)`, `setWindowSystemBarEnable([])`. Every one
of those has to be undone on the way back, which is the part most
implementations get wrong.

The piece worth stealing even if you never build an LED banner is the
**measured space padding** in `getMarqueeSrc()`. `Marquee` only scrolls when its
content is wider than its box, so short text sits motionless. The sample
measures the text and a single space with `MeasureUtils` and concatenates
exactly enough spaces to push the content past the screen width, then repeats
the text - which is the general recipe for turning any marquee into a seamless
loop regardless of content length.

## Feature checklist

- The app opens on a portrait configuration page: display content, font size,
  font colour, background colour, a scroll on/off switch and a scroll speed.
- Font size defaults to the screen width in vp, so one character fills the
  screen edge to edge when rotated.
- Tapping the font-colour or background-colour row opens a bottom sheet with an
  HSV saturation/value canvas, a hue bar, a live colour swatch and a hex field.
- Editing the hex field and blurring it moves both canvas trackers to that
  colour; dragging either canvas updates the hex field.
- 开始展示 (start display) switches the app to landscape, hides the status and
  navigation bars, and shows the text as a full-screen marquee in the chosen
  colours.
- If scrolling is off, the text is shown static.
- Tapping the marquee toggles a 返回 (back) button, positioned below the
  landscape avoid area; pressing it restores portrait, the system bars and the
  configuration page.

## Architecture

One `entry` module, one page and two views.

```
entry/src/main/ets
├── constant/Constants.ets        colours, percentage strings, tracker radii, INFINITE_LOOP = -1
├── entryability/EntryAbility.ets forces COLOR_MODE_LIGHT, loads pages/MainPage
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets            @Entry - the isFullScreen switch, the window calls, the Marquee
├── utils/ColorUtils.ets          hexToHsv / hsvToHex, exported as a singleton instance
└── view
    ├── ColorSelector.ets         @CustomDialog - two Canvases, the HSV picker
    └── ContentConfig.ets         the portrait settings form, six @Link fields
```

The documented tree matches the zip.

**The design decision worth copying** is that `ContentConfig` takes all six
settings as `@Link`, not `@Prop`, and owns nothing. `MainPage` holds the single
copy of `contentStr`, `contentFontSize`, `contentFontColor`,
`contentBackGroundColor`, `whetherScroll` and `scrollSpeed`; the form writes
through to them, and the same `MainPage` state is what the `Marquee` renders in
the other branch of the `if`. There is no save step and no synchronisation code,
because there is only ever one copy of each value.

`ColorSelector` extends that one level further: it is a `@CustomDialog` with a
single `@Link color: string`, so the picker writes straight into
`ContentConfig`'s link, which writes straight into `MainPage`'s state. The
dialog controllers are created eagerly as fields and nulled in
`aboutToDisappear()` - the sanctioned way to break the retain cycle between a
component and a `CustomDialogController` it owns.

## Implementation steps

1. **Grab the window once**, from
   `(this.getUIContext().getHostContext() as common.UIAbilityContext).windowStage.getMainWindowSync()`,
   and keep it as a field. Every mode change is a call on that object.
2. **Resolve the default content to a real string in `aboutToAppear`**, do not
   leave a `Resource` in a field that will be concatenated (`HW-15-0060`).
3. **Measure before you pad.** `this.getUIContext().getMeasureUtils()` gives
   `measureText({ textContent, fontSize })`; measure the content and a single
   space at the *display* font size.
4. **Compare against the screen height, not the width** - the marquee runs in
   landscape, so the long edge is `display.getDefaultDisplaySync().height`.
5. **Build the loop as `text + spaces + text`** so the tail of one pass and the
   head of the next are separated by exactly one screen of blank.
6. **Enter display mode in one handler**: compute the marquee source, set
   `isFullScreen`, `setWindowLayoutFullScreen(true)`,
   `setPreferredOrientation(window.Orientation.LANDSCAPE)`, read the avoid area
   for the back-button margin, then `setWindowSystemBarEnable([])`.
7. **Undo all four on the way back.** The back button restores the layout flag,
   re-enables `['status', 'navigation']` and returns to `PORTRAIT`.
8. **Set `loop: -1` (`Constants.INFINITE_LOOP`) and drive `start` from the
   scroll switch**, so turning scrolling off leaves the text rendered but still.
9. **In the colour picker, update the model before deriving the colour from it**
   (`HW-15-0061`).

## Verified snippets

All snippets are from `SimpleLed.zip`. Corrected forms are marked.

**The marquee source — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0060`)

```typescript
@State contentStr: string = '';                       // FIX: was ResourceStr = $r('app.string.display_content')
@State uiContextMeasure: MeasureUtils = this.getUIContext().getMeasureUtils();
@State contentStrWidth: number = 0;
@State spaceWidth: number = 0;

aboutToAppear(): void {                               // FIX: absent in the sample
  this.contentStr = this.context.resourceManager.getStringSync($r('app.string.display_content').id);
}

getMarqueeSrc(): ResourceStr {
  // Calculate the width of the displayed string
  this.contentStrWidth = this.uiContextMeasure.measureText({
    textContent: this.contentStr,
    fontSize: this.contentFontSize
  });
  // Calculate the width of a space
  this.spaceWidth = this.uiContextMeasure.measureText({
    textContent: ' ',
    fontSize: this.contentFontSize
  });
  if (this.contentStrWidth > this.screenHeight) {
    return this.contentStr
  }

  // When the width of the current displayed content is smaller than the screen width,
  // extend the space to make the character string width exceed the screen width to implement horse racing lamp
  if (this.whetherScroll) {
    return this.contentStr + ' '.repeat(this.screenHeight / this.spaceWidth) + this.contentStr;
  } else {
    return this.contentStr;
  }
}
```

**Three measurements decide the string.** `contentStrWidth` decides whether
padding is needed at all - text already wider than the long screen edge scrolls
on its own. `spaceWidth` converts "one screen of blank" into a space count, so
the gap is a real screen width rather than a guessed number of spaces, and it
stays correct when the user changes the font size. `screenHeight` is deliberate:
the marquee is displayed in landscape, so the display's *height* in the
portrait-reported geometry is the width the text has to beat.

The correction is the initial value. `contentStr` ships as
`$r('app.string.display_content')` typed `ResourceStr`, and both consumers treat
it as a string - `getMarqueeSrc()` concatenates it with `+`, and the `Marquee`
calls `.toString()` on the result. A `Resource` is an object, so before the user
edits anything the LED screen shows `[object Object]`. Resolving it once in
`aboutToAppear` fixes both consumers and costs nothing; the `TextInput` in
`ContentConfig` already writes a plain `string` back into the same field, so
after the first edit the sample works and the bug only ever shows on first run.

**Entering and leaving display mode — same file** (as shipped)

```typescript
Button($r('app.string.start_display'))
  .onClick(() => {
    this.contentStrNormalize = this.getMarqueeSrc();
    this.isFullScreen = true;
    this.windowClass.setWindowLayoutFullScreen(true);
    // Set the landscape mode
    this.windowClass.setPreferredOrientation(window.Orientation.LANDSCAPE);
    this.topMargin = this.getUIContext()
      .px2vp(this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height)
    this.windowClass.setWindowSystemBarEnable([]);
  })

// ... and the way back, on the button that the marquee's onClick reveals:

Button($r('app.string.back'))
  .margin({ top: this.topMargin })
  .onClick(() => {
    this.isFullScreen = false;
    this.isButtonVisible = false;
    this.windowClass.setWindowLayoutFullScreen(false);
    this.windowClass.setWindowSystemBarEnable(['status', 'navigation']);
    // Return to portrait mode
    this.windowClass.setPreferredOrientation(window.Orientation.PORTRAIT);
  })
```

**The order matters on the way in.** `getWindowAvoidArea` is read *after*
`setWindowLayoutFullScreen(true)` and *before* `setWindowSystemBarEnable([])`,
which is the only window in which the value describes the immersive landscape
layout the button will actually live in. Read it first and you get the portrait
status bar; read it last and the bars are already gone.

`contentStrNormalize` is computed once here rather than bound to the marquee,
which is the right call: `measureText` is not something to run on every render
pass of a component that redraws continuously. The cost is that changing the
text while displaying would not take effect - but the display mode has no
editing UI, so it cannot happen.

All four window calls return promises that the sample drops. On the back path
that is a real risk: if `setPreferredOrientation(PORTRAIT)` rejects, the app is
left sideways with the config page in it and nothing logged.

**The HSV panel touch handler — `entry/src/main/ets/view/ColorSelector.ets`** (corrected, see `HW-15-0061`)

```typescript
.onTouch((event) => {
  let x = event.touches[0].x
  let y = event.touches[0].y
  if (x >= this.colorPanelContext.width) {
    x = this.colorPanelContext.width
  }
  if (x < 0) {
    x = 0
  }
  if (y >= this.colorPanelContext.height) {
    y = this.colorPanelContext.height
  }
  if (y < 0) {
    y = 0
  }
  this.colorPanelTrackPoint =
    new Point(x - Constants.PANEL_TRACK_POINT_RADIUS, y - Constants.PANEL_TRACK_POINT_RADIUS)
  const p = this.pointToSatVal(x, y)      // FIX: these three lines ran
  this.sat = p[0]                         // FIX: after this.color = this.getColor()
  this.val = p[1]                         // FIX: in the shipped code
  this.color = this.getColor()
})
```

`getColor()` is `ColorUtils.hsvToHex(this.hue, this.sat, this.val)` - it reads
the model, it does not take the touch point. The shipped order calls it before
`sat` and `val` have been updated from the new point, so the hex field and the
swatch always show the colour from the *previous* touch event while the tracker
circle sits on the current one. The hue bar handler thirty lines above gets the
same sequence right (`this.hue = this.pointToHue(x)` then
`this.color = this.getColor()`), which is what makes this a slip rather than a
design.

The two canvases are one model: `drawColorPanel()` fills the panel with
`hsvToHex(this.hue, 1, 1)` and then lays a white left-to-right gradient and a
black top-to-bottom gradient over it, so moving the hue bar re-tints the whole
saturation/value square. `invalidateHuePanel()` redraws both; the panel's own
handler only redraws the tracker, because the panel's pixels do not depend on
`sat` or `val`.

## Permissions & config

**None.** The sample declares no `requestPermissions` - window orientation and
immersive layout are not permission-gated.

`deviceTypes` is `phone`, `tablet`, `2in1`.

`EntryAbility.onCreate` calls
`this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`,
pinning the app to light mode. That is defensible here - the config form hard-codes
`Color.White` card backgrounds and `#fff7f7f7` page backgrounds, so it would be
unreadable in dark mode - but it is a workaround for unthemed colours, not a
feature.

User-visible strings are English resources (`display content`, `font size`,
`I love you` as the default banner text) with no `zh_CN` variants, despite the
document being Chinese.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `Marquee` scrolls its `src` as a single line. There is no wrapping, no
  per-character styling and no bidi handling; `step` is in vp per frame, so the
  perceived speed depends on the refresh rate.
- The default font size is `screenWidth / scaledDensity`, i.e. one character
  roughly the width of the screen. The font-size field accepts any number
  through `Number(value)` with no clamp, so an empty field yields `NaN` and a
  pasted `999999` is accepted.
- Same for scroll speed: `Number(value)` with no floor, so `0` stops the
  marquee while the switch still says it is scrolling.
- `getWindowAvoidArea` is read once on entry. A device that changes its cutout
  geometry on rotation (a foldable being unfolded in display mode) will leave
  the back button in the wrong place.
- `hexToHsv` assumes a `#rrggbb` string of exactly seven characters. The hex
  `TextInput` accepts anything; a partial value typed by hand yields `NaN`
  channels and the tracker jumps to the origin.
- Nothing is persisted. Content, colours and speed reset on every launch.

## Pitfalls

- **`HW-15-0060` (B/medium, probable) — the first-run marquee renders
  `[object Object]`.** `contentStr` is initialised to
  `$r('app.string.display_content')`, and `getMarqueeSrc()` concatenates it with
  `+` while `MainPage` calls `.toString()` on the result; `Resource` objects do
  not coerce to their text. Fix: resolve the resource with
  `resourceManager.getStringSync` in `aboutToAppear` and type the field
  `string`.
- **`HW-15-0061` (B/low, confirmed) — the colour panel reports a
  one-touch-stale colour.** `this.color = this.getColor()` runs before `sat` and
  `val` are updated from the touch point, so the hex field and swatch lag the
  tracker by one event. Fix: reorder the statements, as the hue bar handler
  already does.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-marquee.md` - `Marquee`, `src`, `step`, `loop`, `start`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-marquee
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowLayoutFullScreen`, `setPreferredOrientation`, `setWindowSystemBarEnable`, `getWindowAvoidArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureText` and its `MeasureOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getDefaultDisplaySync`, `width`, `height`, `scaledDensity`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `onReady` and the rendering context
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `@CustomDialog` and `CustomDialogController` lifetime
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
