---
id: OFFICE-24
title: Application background watermark - a tiled rotated Canvas attached as an overlay over the content
industry: 05_office
doc: huawei_industry_tree/05_office/docs/24_app_watermark.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
sample: huawei_industry_tree/05_office/downloads/AppWatermark.zip
kits: ["@kit.ArkUI"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, "CanvasRenderingContext2D.fillText", "CanvasRenderingContext2D.translate", "CanvasRenderingContext2D.rotate", "CanvasRenderingContext2D.resetTransform", "CanvasRenderingContext2D.width", "CanvasRenderingContext2D.height", "overlay()", HitTestMode, Stack, Scroll, Scroller, "Scroller.scrollEdge", ForEach, "UIContext.setKeyboardAvoidMode", KeyboardAvoidMode, "window.getLastWindow", "window.on('keyboardHeightChange')", "window.off('keyboardHeightChange')", "UIContext.px2vp", "resourceManager.getStringSync", "@Watch", "@StorageProp", Navigation]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0133, HW-05-0134, HW-05-0135, HW-05-0136, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office screen shows **internal or confidential content**
and policy requires a **repeating identity watermark behind it** - the user's
name or employee id tiled diagonally across the background so any screenshot or
photograph carries attribution.

The whole feature is two ArkUI pieces:

- a `Canvas` that draws the tiled, rotated text once in `onReady`;
- the universal **`overlay()`** attribute, which layers that canvas on top of a
  content component without changing its layout.

Compare with OFFICE-04, which uses the same `overlay(createWaterMarkView())`
idea over a `Web` component: there the watermark protects a single previewed
document, here it protects a whole page of application content.

No permission and no system service is involved - it is pure drawing.

## Feature checklist

The application must be able to:

- Read the watermark text from a string resource so it can be localised or
  replaced by an identity at runtime.
- Draw the text repeatedly across the full width and height of the content area.
- Rotate every instance by a fixed angle so the pattern reads as a watermark
  rather than as body text.
- Keep the watermark faint enough not to interfere with reading.
- Let touches pass through the watermark to the content beneath it.
- Attach the watermark without displacing or resizing the content.
- Keep the chat content scrolled to the bottom when the keyboard opens.

## Architecture

Single `entry` HAP, one page:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes `topRectHeight` / `bottomRectHeight`, loads `pages/WatermarkPage` |
| `pages/WatermarkPage.ets` | **the whole feature**: title bar, chat content with the watermark overlay, input bar, keyboard handling |
| `model/ChatFormat.ets` | the message type and the `CHAT_CONTENT` seed |
| `util/DateUtil.ets` | the date header string |
| `constants/Constants.ets` | bar heights, the watermark tile size and the degree constant |

Note the directory is `util/` (singular) - the document's tree says `utils/`
(HW-05-0136).

The layering is a `Stack` in which the watermark is attached to a **sibling**
of the content, not to the content itself:

```
Stack()
  Column().layoutWeight(1).overlay(this.waterMark())    // an empty full-size column, carrying the watermark
  Scroll(scroller) { ...the message list... }           // the real content, drawn on top of it
```

Attaching the overlay to an empty `Column` rather than to the `Scroll` is what
keeps the watermark **static while the content scrolls** - an overlay on the
scrolling component would scroll with it.

The tiling itself is a nested loop over the canvas in the drawing context's own
units:

```
for each row (posY = WATERMARK_HEIGHT * row) while posY < ctx.height
    resetTransform()                    drop the previous row's transform
    translate(0, posY)                  move to the row origin
    rotate(-30deg)                      tilt the whole row
    for each column across ctx.width
        fillText(text, 0, H/2)          draw one tile at the current origin
        translate(WATERMARK_WIDTH, 0)   step along the rotated axis
    rotate(+30deg)
```

`resetTransform()` at the top of each row is the key call: without it the
translations and rotations of the previous row would accumulate.

## Implementation steps

1. **Declare no permission.** The feature is drawing only; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Create one rendering context** for the page:
   `new CanvasRenderingContext2D(new RenderingContextSettings(true))`.
3. **Load the watermark text from a resource** so it can be swapped for a real
   identity: `context.resourceManager.getStringSync($r('app.string.watermark').id)`.
4. **Draw once in `onReady`.** Set `fillStyle` to a very low alpha
   (`'#0D000000'` here), a small `font`, and `textAlign = 'start'`, then run the
   row/column loop using `this.context.width` and `this.context.height` so the
   pattern adapts to the component size.
5. **Reset the transform per row.** `resetTransform()` before each row's
   `translate` / `rotate` pair.
6. **Let touches through.** `hitTestBehavior(HitTestMode.Transparent)` on the
   `Canvas`, so the watermark never intercepts a tap meant for the content.
7. **Attach with `overlay()`** to a full-size sibling of the content inside a
   `Stack`, so the watermark stays put while the content scrolls.
8. **Handle the keyboard.** `setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE)` so
   the content area shrinks rather than shifting, plus a
   `keyboardHeightChange` subscription that scrolls the list to the end. Restore
   the mode (HW-05-0134) and release the subscription (HW-05-0133) when the page
   goes away.
9. **Key the message list** on a stable id (HW-05-0135).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The watermark canvas

`AppWatermark.zip#entry/src/main/ets/pages/WatermarkPage.ets`

```ts
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);

@Builder
waterMark() {
  Canvas(this.context)
    .width('100%')
    .height('100%')
    .hitTestBehavior(HitTestMode.Transparent)
    .onReady(() => {
      this.context.fillStyle = '#0D000000';
      this.context.font = '14vp';
      this.context.textAlign = 'start';
      let posY = 0;
      let row = 0;
      while (posY < this.context.height) {
        this.context.resetTransform();
        posY = WATERMARK_HEIGHT * row;
        this.context.translate(0, posY);
        this.context.rotate(-30 * DEGREE);
        for (let i = 0; i <= Math.ceil(this.context.width / WATERMARK_WIDTH); i++) {
          this.context.fillText(this.waterMarkText, 0, WATERMARK_HEIGHT / 2);
          this.context.translate(WATERMARK_WIDTH, 0);
          posY -= WATERMARK_WIDTH * Math.sin(30 * DEGREE);
        }
        this.context.rotate(30 * DEGREE);
        row++;
      }
    });
}
```

`AppWatermark.zip#entry/src/main/ets/constants/Constants.ets`

```ts
export const TITLE_BAR_HEIGHT = 56;
export const INPUT_BAR_HEIGHT = 52;
export const WATERMARK_HEIGHT = 100;   // row pitch
export const WATERMARK_WIDTH = 160;    // column pitch, along the rotated axis
export const DEGREE = Math.PI / 180;   // degrees -> radians
```

Three details carry the effect: `fillStyle: '#0D000000'` is black at alpha `0x0D`
(about 5%), the `+ 1` in the column bound (`i <= Math.ceil(...)`) draws one tile
past the right edge so the rotated row reaches the corner, and the column count
is derived from `this.context.width` rather than a constant, so the pattern fills
whatever size the component ends up.

One line to read carefully when adapting this: `posY -= WATERMARK_WIDTH *
Math.sin(30 * DEGREE)` inside the **inner** loop mutates the variable that the
**outer** `while` tests, and the value is then overwritten by
`posY = WATERMARK_HEIGHT * row` at the top of the next iteration. The net effect
is that the outer loop runs a few rows past `ctx.height` - which is plausibly
deliberate, since a row anchored below the bottom edge still rises into the
canvas on the right after the -30 degree rotation.

### Attaching the overlay

`AppWatermark.zip#entry/src/main/ets/pages/WatermarkPage.ets`

```ts
@Builder
chatContent() {
  Stack() {
    Column()
      .width('100%')
      .layoutWeight(1)
      .overlay(this.waterMark());

    Scroll(this.scroller) {
      Column({ space: 12 }) {
        Text(DateUtil.getCurrentDateString())
          .fontColor($r('sys.color.font_secondary'));

        Column({ space: 24 }) {
          ForEach(this.chatMessages, (messages: ChatFormat) => {
            if (messages.isSelf) {
              this.rightBubble(messages.message);
            } else {
              this.leftBubble(messages.message);
            }
          });                                  // no key generator - HW-05-0135
        };
      }
      .width('100%');
    }
    .scrollBar(BarState.Off)
    .align(Alignment.Top)
    .width('100%')
    .layoutWeight(1);
  };
}
```

This is the part worth copying: an empty `Column` sized to the content area
carries the `overlay()`, and the scrolling content is a separate `Stack` child
drawn over it. The watermark therefore stays fixed relative to the viewport while
the messages scroll underneath it, and adding the overlay costs the content no
layout at all.

### Keyboard handling

`AppWatermark.zip#entry/src/main/ets/pages/WatermarkPage.ets`

```ts
@State @Watch('onKeyboardChange') keyboardHeight: number = 0;
private scroller: Scroller = new Scroller();

onKeyboardChange() {
  if (this.keyboardHeight !== 0) {
    this.scroller.scrollEdge(Edge.End);
  }
}

aboutToAppear(): void {
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);   // never restored - HW-05-0134

  let context = this.getUIContext().getHostContext();
  if (context) {
    window.getLastWindow(context).then(currentWindow => {
      currentWindow.on('keyboardHeightChange', data => {                // never off - HW-05-0133
        this.keyboardHeight = this.getUIContext().px2vp(data);
      });
    });                                                                 // no .catch()

    this.waterMarkText = context.resourceManager.getStringSync($r('app.string.watermark').id);
  }
}
```

The `@State @Watch` pair is a clean idiom: the keyboard height is stored as
observable state, and the watcher reacts to any change by scrolling the
transcript to the end - so the newest message stays visible when the keyboard
opens. `KeyboardAvoidMode.RESIZE` is what makes that work, since it shrinks the
content area instead of pushing the whole page up.

Corrected lifecycle:

```ts
private currentWindow?: window.Window;
private keyboardCallback: Callback<number> = (data: number) => {
  this.keyboardHeight = this.getUIContext().px2vp(data);
};
private previousAvoidMode: KeyboardAvoidMode = KeyboardAvoidMode.OFFSET;

aboutToAppear(): void {
  this.previousAvoidMode = this.getUIContext().getKeyboardAvoidMode();
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
  const context = this.getUIContext().getHostContext();
  if (!context) {
    return;
  }
  this.waterMarkText = context.resourceManager.getStringSync($r('app.string.watermark').id);
  window.getLastWindow(context)
    .then((currentWindow) => {
      this.currentWindow = currentWindow;
      currentWindow.on('keyboardHeightChange', this.keyboardCallback);
    })
    .catch((err: BusinessError) => {
      hilog.error(0x0000, 'WatermarkPage', 'getLastWindow failed: %{public}s', JSON.stringify(err));
    });
}

aboutToDisappear(): void {
  this.currentWindow?.off('keyboardHeightChange', this.keyboardCallback);
  this.getUIContext().setKeyboardAvoidMode(this.previousAvoidMode);
}
```

## Permissions & config

`AppWatermark.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the watermark is drawn from a
string resource onto a `Canvas`, with no system service, no file and no network
involved. The document has no 权限说明 section, which matches.

The watermark text itself comes from `app.string.watermark`; replacing it with a
runtime identity (employee id, account name) is the one change a production
deployment needs, and it belongs in `aboutToAppear` alongside the existing
`getStringSync` call.

`build-profile.json5` pins the SDK to `6.0.0(20)` and enables
`caseSensitiveCheck: true`, which is why the `util/` vs `utils/` discrepancy in
the document matters (HW-05-0136).

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`onReady` fires once per layout.** The pattern is drawn when the canvas is
  ready and is not redrawn on a state change, so a watermark text that changes at
  runtime needs an explicit redraw.
- **The canvas draws in vp**, and `this.context.width` / `height` report the
  component size - which is why the loop bounds must come from the context rather
  than from constants if the pattern is to fill an arbitrary container.
- **`resetTransform()` per row is required.** `translate` and `rotate` compose
  into the current transform, so without the reset each row would inherit the
  previous row's offset and angle.
- **The overlay must not take touch.** `HitTestMode.Transparent` on the `Canvas`
  is what lets taps reach the content underneath; without it the watermark
  swallows every gesture on the page.
- **Attach the overlay to a sibling, not to the scroller.** An `overlay()` on the
  scrolling component scrolls with its content and the watermark would drift.
- **This is a deterrent, not a control.** A background watermark attributes a
  screenshot; it does not prevent one. For that, pair it with the privacy-window
  approach in OFFICE-04 (`window.setWindowPrivacyMode`).
- **Alpha matters.** `'#0D000000'` is deliberately faint; a stronger fill makes
  the content hard to read, a weaker one may not survive a photograph.

## Pitfalls

- **Subscribing to `keyboardHeightChange` with no `off` is incorrect**, and the
  page declares no `aboutToDisappear` at all, so the callback outlives the page
  and keeps writing into its state; the `getLastWindow` promise also has no
  `.catch()`. (HW-05-0133)
- **Setting `KeyboardAvoidMode.RESIZE` without restoring it is incorrect.**
  `setKeyboardAvoidMode` is called on the `UIContext`, which is window-scoped, so
  the mode this chat page wants persists for every page shown afterwards. Record
  the previous mode and put it back. (HW-05-0134)
- **`ForEach` over the message list with no key generator is incorrect.** The
  default key embeds the index and a `JSON.stringify` of each `ChatFormat`, so
  every send re-serialises the whole transcript on the UI thread. Key on a stable
  id. (HW-05-0135)
- **The project tree's `utils/` is incorrect** - the sample ships `util/`, which
  is also what `WatermarkPage.ets` imports, and the build enables
  `caseSensitiveCheck`. (HW-05-0136)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-overlay.md` -
  the `overlay()` universal attribute used to layer the canvas over the content.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-overlay
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` -
  `fillText`, `translate`, `rotate`, `resetTransform`, `fillStyle`, `font`,
  `textAlign` and the context's `width` / `height`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` -
  the `Canvas` component and its `onReady` callback.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `getLastWindow`, `on('keyboardHeightChange')` and its `off` counterpart.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the default key generator and the warning against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `setKeyboardAvoidMode` / `getKeyboardAvoidMode` and `px2vp`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
