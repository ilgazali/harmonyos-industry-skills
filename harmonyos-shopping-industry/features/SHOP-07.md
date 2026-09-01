---
id: SHOP-07
title: Scratch card - erase a PixelMap cover on a Canvas with destination-out compositing
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/07_scratch_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/scratch_effect-0000002320935777
sample: huawei_industry_tree/16_shopping/downloads/scratch.zip
kits: ["@kit.ArkUI", "@kit.ImageKit", "@kit.PerformanceAnalysisKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.CoreFileKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, onReady, onTouch, globalCompositeOperation, beginPath, arc, fill, closePath, drawImage, "image.createImageSource", "ImageSource.createPixelMap", "ImageSource.release", "image.DecodingOptions", "resourceManager.getMediaContentSync", "UIContext.vp2px", Stack, expandSafeArea]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0008, HW-16-0013, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card for any **scratch-to-reveal** surface: lottery cards in an
anniversary campaign, blurred coupon codes, spoiler covers, "rub to unlock"
onboarding. The mechanic is a marketing staple in shopping apps and this is the
minimal correct implementation of it.

The idea is one line of the whole file: draw the cover image onto a `Canvas`,
then set `globalCompositeOperation = 'destination-out'` and keep drawing. From
that point every shape you draw does not add pixels — it **removes** the pixels
already there. The finger becomes an eraser and the result underneath, which is
an ordinary `Image` sitting lower in a `Stack`, shows through the holes.

What generalises is the layering discipline, not the lottery. Anything that
should be *progressively* revealed by a pointer — a spotlight over a map, a
smudge-to-clear frosted panel, an eraser tool in a drawing app — is the same
three parts: content underneath, an opaque canvas above, and `destination-out`
strokes. Nothing about the content layer needs to know it is being revealed.

## Feature checklist

- A full-bleed promotional background with an anniversary banner and a VIP
  graphic.
- A scratch panel: a rounded background plate, the prize text under it, and an
  opaque grey cover exactly on top of the prize.
- Touching the cover and dragging erases a round patch under the finger.
- The erased area accumulates across strokes; lifting and touching again keeps
  what was already scratched.
- Drawing only happens while a finger is down — a stray move event outside a
  press does nothing.
- The prize text becomes readable as the cover is removed.
- A line under the panel states how many draws are left.

## Architecture

One `entry` module, one page, one constants file. The entire feature is 182
lines.

```
entry/src/main/ets
├── constants/CommonConstants.ets      sizes and margins (296x80 canvas, 328x112 plate)
├── entryability/EntryAbility.ets      window setup
├── entrybackupability/EntryBackupAbility.ets
└── pages/ScratchPage.ets              @Entry: the Stack, the Canvas, the cover decode, the erase
```

The documented tree matches the zip.

The visual stack, innermost first:

```
Stack()                       the scratch panel
├── Image(scratch_bg)         328x112  rounded plate behind everything
├── Image(scratch_btn)        296x80   the cover artwork, as a static image
├── Image(prize_name)         296x80   THE RESULT — what the user is scratching for
└── Canvas(this.context)      296x80   the interactive cover, drawn with the same scratch_btn art
```

**The design decision worth copying** is that the *result* is a plain `Image`
and only the *cover* lives on the canvas. The canvas never draws the prize, never
knows what the prize is, and never needs a second compositing pass. Reveal
progress is the alpha channel of one bitmap, maintained by the graphics engine.
Swapping the prize is a resource change with no code change, and animating the
prize (a video, a Lottie, a `@State`-driven `Text`) works because it is an
ordinary component in an ordinary layout.

The one structural oddity to understand before copying: `Image(scratch_btn)` is
drawn *twice* — once as a static image in the stack and once as the pixel map
painted into the canvas. The static copy is what the user sees during the first
frame, before `onReady` has finished the asynchronous decode; without it the
prize would flash visible on entry. It is a deliberate anti-flash placeholder,
not a duplicate.

## Implementation steps

1. **Create the context with antialiasing on**:
   `new CanvasRenderingContext2D(new RenderingContextSettings(true))`. Erasing
   with hard-edged circles looks like a stencil; antialiased edges look like a
   coin scratch.
2. **Size the `Canvas` and the cover identically** — 296x80 vp here, from
   constants — and give the canvas `backgroundColor(Color.Transparent)` so
   unpainted regions show the layers below rather than white.
3. **Decode the cover to a `PixelMap` at the canvas's pixel size.** Read the
   raw bytes with `resourceManager.getMediaContentSync(resource.id)`, wrap them
   in an `ImageSource`, and pass `desiredSize` in **px** (`vp2px` the vp
   dimensions). Decoding at display size is what keeps the erase cheap.
4. **Set `editable: true`** in the `DecodingOptions`. A non-editable pixel map
   cannot be the target of destructive compositing.
5. **Release the `ImageSource` in a `finally`.** It holds the decoded buffer;
   the `PixelMap` does not depend on it after `createPixelMap` resolves.
6. **Draw the cover in `onReady`, then switch the composite mode** — in that
   order. The sample's own comment says it: 注意需要先绘制背景在设置 (note: draw
   the background first, then set it). Setting `destination-out` before the
   first `drawImage` erases instead of paints.
7. **Gate the strokes on a press flag.** `isDrawing` goes true on
   `TouchType.Down`, false on `TouchType.Up`, and `scratch()` returns early
   when it is false.
8. **Erase with filled arcs at the touch point** — `beginPath` / `arc(x, y, 20,
   0, Math.PI * 2)` / `fill` / `closePath`. `fill` is what removes pixels;
   `stroke` would only remove a ring.
9. **Do not copy the document's step-1 snippet** (`HW-16-0008`, `HW-16-0013`):
   it prints `getMediaContent(\n id: resource.id)`, a named-argument form that
   does not exist in ArkTS, and names the async API the sample does not use.

## Verified snippets

All snippets are from `scratch.zip`. The document's versions of both are
broken; these are the shipped, compiling forms.

**Decoding the cover — `entry/src/main/ets/pages/ScratchPage.ets`** (as shipped — the doc's version is `HW-16-0008`)

```typescript
import { image } from '@kit.ImageKit';

// 构建灰图（遮盖图）
private async getPixmapFromMedia(resource: Resource): Promise<image.PixelMap> {
  let imageSource: image.ImageSource | undefined = undefined;
  let unit8Array: Uint8Array | undefined = undefined;

  try {
    // 1.获取媒体资源
    unit8Array = this.getUIContext().getHostContext()?.resourceManager?.getMediaContentSync(resource.id);

    if (!unit8Array) {
      throw new Error('Failed to get media content: unit8Array is null or undefined');
    }

    // 2.创建ImageSource
    const BUFFER = unit8Array.buffer.slice(0, unit8Array.buffer.byteLength);
    imageSource = image.createImageSource(BUFFER);

    // 3.设置解码选项
    const DECODING_OPTIONS: image.DecodingOptions = {
      editable: true,
      desiredPixelFormat: 3,
      desiredSize: {
        width: this.getUIContext().vp2px(296),
        height: this.getUIContext().vp2px(296 / 3.5)
      }
    };

    // 4.创建PixelMap
    const CREATE_PIXEL_MAP: image.PixelMap = await imageSource.createPixelMap(DECODING_OPTIONS);
    return CREATE_PIXEL_MAP;
  } catch (error) {
    let errorMsg = (error as Error).message;
    hilog.error(0x0000, 'ScratchPage', 'Error in getPixmapFromMedia: %{public}s', errorMsg);
    throw new Error(`Failed to get pixmap: ${errorMsg}`);
  } finally {
    if (imageSource) {
      try {
        await imageSource.release();
      } catch (e) {
        hilog.error(0x0000, 'ScratchPage', 'Failed to release imageSource: %{public}s', (e as Error).message);
      }
    }
  }
}
```

**Three decoding options carry the design.** `editable: true` is mandatory —
the pixel map is about to be composited destructively, and a read-only decode
cannot back that. `desiredSize` converts the canvas's vp dimensions to px with
`vp2px`, so the cover decodes at exactly the density the canvas paints at:
decode larger and every erase costs a downscale, decode smaller and the cover
looks soft against the crisp prize text beneath it. `desiredPixelFormat: 3` is
the ordinal for RGBA_8888 — the alpha channel is not optional here, because
`destination-out` works by writing alpha, and an opaque format has nowhere to
write the hole. Use the named enum member rather than the literal `3`.

`.buffer.slice(0, byteLength)` looks redundant and is not: `getMediaContentSync`
returns a `Uint8Array` that may be a view into a larger `ArrayBuffer`, and
`createImageSource` takes a buffer, not a view. Slicing produces a standalone
buffer of exactly the resource's bytes.

The `finally` block is the part most implementations forget. `ImageSource` owns
the encoded bytes and the decoder; the `PixelMap` it produced is independent of
it, so releasing immediately after `createPixelMap` is both safe and necessary.
Note it is also correct on the *throw* path — a decode failure still releases.

**Painting the cover and arming the eraser — same file** (as shipped)

```typescript
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);

Canvas(this.context)
  .width(CommonConstants.SCRATCH_BTN_WIDTH)     // 296
  .height(CommonConstants.SCRATCH_BTN_HEIGHT)   // 80
  .backgroundColor(Color.Transparent)           // 限制长宽
  .onReady(() => {
    this.getPixmapFromMedia($r('app.media.scratch_btn')).then((image) => {
      this.context.drawImage(image, this.x, this.y);
      // 设置刮开涂层效果,注意需要先绘制背景在设置
      this.context.globalCompositeOperation = 'destination-out';
    });
  })
  .onTouch((event) => {
    switch (event.type) {
      case TouchType.Down:                      // shipped code uses the literal 0
        this.isDrawing = true;
        const TOUCH = event.touches[0];
        this.lastX = TOUCH.x;
        this.lastY = TOUCH.y;
        this.scratch(event);
        break;
      case TouchType.Up:                        // literal 1
        this.isDrawing = false;
        break;
      case TouchType.Move:                      // literal 2
        this.scratch(event);
        break;
    }
  });
```

**The ordering inside `onReady` is the entire trick.** The canvas starts empty.
`drawImage` runs under the default `'source-over'` and paints the cover
normally. Only then does `globalCompositeOperation` flip to `'destination-out'`,
which the reference defines as "displays the existing drawing outside the new
drawing" — every subsequent shape subtracts its own area from what is already
on the canvas. Flip it before the `drawImage` and the cover never appears,
because the first draw erases an empty canvas and then composites nothing.

Because the decode is asynchronous, the flip happens inside the `.then`, not
after it. A touch that arrives before the promise resolves finds a canvas with
`source-over` still set and would *paint* black circles instead of erasing —
academic in practice at this decode size, but the reason the static
`Image(scratch_btn)` sits underneath as the first-frame stand-in.

`TouchType.Down/Up/Move` are the named members for the literals 0, 1 and 2 the
sample compares against; the enum reads as intent and survives any future
reordering.

**The eraser stroke — same file** (as shipped)

```typescript
@State isDrawing: boolean = false;

// 刮开遮盖层
scratch(event: TouchEvent) {
  if (!this.isDrawing) {
    return;
  }
  const TOUCH = event.touches[0];
  this.context.beginPath();
  this.context.arc(TOUCH.x, TOUCH.y, 20, 0, Math.PI * 2);
  this.context.fill();
  this.context.closePath();
}
```

**`fill`, not `stroke`, and a fresh `beginPath` every time.** `fill` under
`destination-out` clears the disc's whole area; `stroke` would clear only its
outline, leaving an untouched island in the middle of every dab. `beginPath`
resets the path list — without it each call appends another arc to the same path
and `fill` re-fills every arc drawn since the touch started, which is quadratic
work per frame and produces visible slowdown after a few seconds of scratching.

The `isDrawing` guard exists because `onTouch` also delivers `Move` events in
sequences the app did not start (a drag entering the canvas from outside, a
second finger). Without it the cover erases under a finger that was never
pressed on the card.

The radius is a hardcoded `20`. That is the eraser's feel: too small and the
card takes ten strokes to clear, too large and one swipe reveals everything.
It belongs in `CommonConstants` with the rest of the geometry.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `getMediaContentSync`
reads the app's own resources through its `resourceManager`, which needs no
permission at all — that is a further reason to prefer bundled cover art over a
network image here.

`deviceTypes`: `phone`, `tablet`, `2in1`. No `routerMap`; `main_pages` lists
`pages/ScratchPage` only. The background image uses
`expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])`
so the promo art bleeds under the status bar and navigation indicator while the
content column stays inside them.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **`onReady` fires on initialisation *and* on every size change.** As written,
  a second `onReady` runs `drawImage` while `globalCompositeOperation` is
  already `'destination-out'` — so a resize (a foldable unfolding, a 2in1
  window drag) erases the cover in the shape of the cover instead of restoring
  it. Reset the mode to `'source-over'` at the top of the callback if the
  canvas can ever be resized.
- The canvas is also documented as not responding to drawing commands while it
  is not visible — page in background, scrolled out of the window, or
  `visibility` hidden. Do not issue the initial draw from a hidden state.
- **There is no completion detection.** Nothing measures how much of the cover
  is gone, so no "you won" state fires and the prize is only ever revealed by
  more scratching. A real implementation samples the pixel map's alpha
  periodically (or counts covered area) and auto-clears the rest past a
  threshold — that is the missing half of the feature, not an oversight in the
  compositing.
- **The card cannot be reset.** No code re-runs the decode, so once scratched it
  stays scratched for the lifetime of the page, and there is only ever one card.
- `lastX` / `lastY` are assigned on every `Down` and never read. The obvious
  intent was interpolating between successive touch points; without it a fast
  swipe leaves gaps between the dabs, because `onTouch` samples at frame rate
  and the arcs are discrete. Interpolating from `lastX/lastY` to the current
  point (a stroked thick line, or arcs stepped along the segment) is the fix.
- Only `event.touches[0]` is used, so a second finger is ignored rather than
  erasing in two places.
- `CommonConstants.FULL_WIDTH` is `'120%'` and the background image carries
  `margin({ left: 40 })`; the layout is tuned to one screen width and will not
  centre correctly on a tablet.
- `desiredSize` is computed from the literals `296` and `296 / 3.5` rather than
  from `SCRATCH_BTN_WIDTH` / `SCRATCH_BTN_HEIGHT`, so changing the constants
  silently desynchronises the cover from the canvas.

## Pitfalls

- **`HW-16-0008`** (E/low, confirmed): the document's `getPixmapFromMedia`
  snippet prints `await …resourceManager?.getMediaContent(\n id: resource.id)`
  — a named-argument syntax that does not exist in ArkTS/TypeScript, so the
  snippet cannot compile — and it names the asynchronous `getMediaContent`
  while the zip calls the synchronous `getMediaContentSync(resource.id)` with no
  `await`. Fix: print `getMediaContentSync(resource.id)` as shipped, or a valid
  awaited `getMediaContent(resource.id)` form.
- **`HW-16-0013`** (E/medium, confirmed, systematic): this document is one of
  the ~30 pages across the reviewed corpus whose published snippets are
  abridged by removing structurally necessary code rather than eliding bodies.
  Here the elision mangles the argument list of step 1. The zip source is valid
  in every instance; only the excerpt is broken. Fix: regenerate excerpts with
  brace-balanced, syntax-preserving elision.

## References

- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` — `globalCompositeOperation` (the full mode table and the `destination-out` eraser example), `drawImage` with a `PixelMap`, `beginPath`/`arc`/`fill`, `RenderingContextSettings`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` — `onReady` semantics, including the resize re-fire and the invisible-canvas rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-guides/03_application-framework/arkts-drawing-customization-on-canvas.md` — the Canvas drawing workflow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-drawing-customization-on-canvas
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-stack-layout.md` — `Stack` layering and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-stack-layout
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` and `documentation/harmonyos-references/04_media/arkts-apis-image-f.md` — `PixelMap` and `createImageSource` (stub pages in our corpus; use the online reference for `DecodingOptions` and `PixelMapFormat`)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-f
- `huawei_industry_tree/16_shopping/docs/07_scratch_effect.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scratch_effect-0000002320935777
