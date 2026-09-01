---
id: PHOTO-16
title: Colour adjustment - brightness, contrast and saturation as Image attributes, exported by component snapshot
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/16_image_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_effect-0000002408004045
sample: huawei_industry_tree/18_photography/downloads/ImageEffect.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fs, hilog, image, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0046, HW-18-0036, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you need **basic photo colour editing without an image
pipeline** - the brightness / contrast / saturation row under a preview, with a
centre-zero slider and an export button. It is the entry-level editor in a
camera app, and the same technique applies anywhere a live "before/after" needs
to be cheap: avatar tuning, theme previews, video thumbnails.

The idea worth taking is that **the adjustment is not image processing at all**.
`.brightness()`, `.contrast()` and `.saturate()` are ArkUI universal attributes
applied by the render backend to the component, so dragging a slider re-renders
one node on the GPU and touches no pixel buffer. Export then happens once, at
the end, by asking the framework for a snapshot of that node. An editor built
this way stays at 60 fps on a 48 MP photo, where a `PixelMap`-per-frame approach
would not survive the first drag.

The trade is precision and reach: you get exactly the effects ArkUI exposes,
applied to a rendered component at its on-screen resolution. **The export is
where this sample is weakest** - read `HW-18-0046` before copying `PhotoUtils`.

## Feature checklist

- A dark full-screen editor with the photo centred and a title bar carrying a
  `SaveButton`.
- A row of five mode icons; three of them (亮度 brightness, 对比度 contrast,
  饱和度 saturation) are live, two are inert decoration.
- Selecting a mode highlights its icon and text in blue and loads that channel's
  stored value into the slider.
- A custom centre-zero slider runs from -100 to +100, drawn as a bar that grows
  left or right from the middle.
- The numeric value is shown to the left of the slider and updates while
  dragging.
- Tapping anywhere on the slider track jumps the handle there.
- A reset icon on the right returns the current channel to 0.
- Adjustments to the three channels are independent and all remain applied at
  once.
- The `SaveButton` exports the adjusted image to the gallery and toasts on
  success or failure.

## Architecture

One `entry` module, one page, one component, one utility. 506 lines total.

```
entry/src/main/ets
├── component/SliderComponent.ets   the centre-zero slider: track, fill, handle, PanGesture + TapGesture
├── constants/CommonConstants.ets   the three mode ids
├── entryability/EntryAbility.ets   full screen; status bar height -> AppStorage
├── entrybackupability/
├── pages/EffectPage.ets            @Entry: the image, the mode row, the save flow (226 lines)
└── utils/PhotoUtils.ets            createAsset + write the packed buffer
```

The documented tree matches the zip (it names the backup ability
`EntryBackAbility.ets` where the zip has `EntryBackupAbility.ets` - naming drift
only).

**The design decision worth copying** is the four-variable state model. One
"live" value plus one stored value per channel:

```typescript
@State @Watch('onValueChange') adjustValue: number = 0;   // what the slider is on right now
@State lightValue: number = 0;                            // stored brightness
@State contrastRatioValue: number = 0;                    // stored contrast
@State saturateValue: number = 0;                         // stored saturation
@State selectMode: number = 1;                            // which channel adjustValue belongs to
```

`@Watch('onValueChange')` writes `adjustValue` back into the stored value for the
current mode on every change, and each mode button copies the stored value back
into `adjustValue` when selected. So the slider is a single reusable control that
is *rebound* rather than duplicated, and the three effects all stay applied
simultaneously because the `Image` reads the stored values for the channels that
are not selected.

**What to avoid** is the conditional expression this forces into the `Image`
attributes - each of the three attributes carries a ternary choosing between
`adjustValue` and its stored value. It works, but a small `getChannel(mode)`
helper would say the same thing once instead of three times.

## Implementation steps

1. **Keep one live value and one stored value per channel,** with `@Watch` on
   the live one writing through to the stored one.
2. **Map the -100..+100 slider domain onto the attribute domain** with
   `value / 200 + 1`, giving 0.5 at -100, 1.0 at 0 and 1.5 at +100 - a range
   where all three attributes stay visually sane.
3. **Bind `.brightness()`, `.contrast()` and `.saturate()` on the same `Image`,**
   each reading the live value when its mode is selected and the stored value
   otherwise.
4. **Give the `Image` an `.id('mainImage')`** - that id is the only handle the
   export has on the rendered result.
5. **Build the centre-zero slider from three `Row`s in a `Stack`:** track, fill
   whose width is `|value - trackWidth/2|`, and a round handle positioned by
   margin.
6. **Two-way bind the slider with `@Link @Watch`** in both directions, so an
   external reset (the reset icon writing `adjustValue = 0`) moves the handle.
7. **Export with `getComponentSnapshot().get('mainImage', { scale: 1,
   waitUntilRenderFinished: true })`** - without `waitUntilRenderFinished` the
   snapshot can precede the last attribute change.
8. **Pack the snapshot and create the gallery asset with a matching extension**
   (`HW-18-0046`), and release the packer afterwards (`HW-18-0036`).
9. **Put `createAsset` and `fs.open` inside the try** that shows the failure
   toast, and add a `.catch` to the snapshot/pack chain (`HW-18-0046`).

## Verified snippets

All snippets are from `ImageEffect.zip`. Corrected forms are marked.

**The three effects — `entry/src/main/ets/pages/EffectPage.ets`** (as shipped)

```typescript
onValueChange() {
  if (this.selectMode === CommonConstants.SELECT_MODE_LIGHT) {
    this.lightValue = this.adjustValue;
  } else if (this.selectMode === CommonConstants.SELECT_MODE_CONTRAST_RATIO) {
    this.contrastRatioValue = this.adjustValue;
  } else {
    this.saturateValue = this.adjustValue;
  }
}

Image($r('app.media.main_image'))
  .width('100%')
  .brightness(this.selectMode === CommonConstants.SELECT_MODE_LIGHT ? this.adjustValue / 200 + 1 :
    this.lightValue / 200 + 1)
  .contrast(this.selectMode === CommonConstants.SELECT_MODE_CONTRAST_RATIO ? this.adjustValue / 200 + 1 :
    this.contrastRatioValue / 200 + 1)
  .saturate(this.selectMode === CommonConstants.SELECT_MODE_EXPOSURE ? this.adjustValue / 200 + 1 :
    this.saturateValue / 200 + 1)
  .id('mainImage')
  .margin({ top: 16, bottom: 230 });

// selecting a mode swaps which channel the slider is editing
Column({ space: 5 }) {
  Image($r('app.media.contrast_ratio'))
    .fillColor(this.selectMode === CommonConstants.SELECT_MODE_CONTRAST_RATIO ? '#5291FF' : Color.White)
    .width(24);
  Text($r('app.string.main_contrast_ratio'));
}
.onClick(() => {
  this.selectMode = CommonConstants.SELECT_MODE_CONTRAST_RATIO;
  this.adjustValue = this.contrastRatioValue;
});
```

**`value / 200 + 1` is the whole colour maths.** All three attributes take a
multiplier where 1.0 is "unchanged", so one linear map serves all of them and the
slider needs no per-channel scaling. Halving to doubling (0.5 to 1.5) is a
deliberately conservative range - `brightness` and `contrast` accept much more,
but beyond about 1.5 a photo blows out and the slider stops feeling like an
adjustment.

The ternaries are what keep the three effects independent: only the selected
channel follows the live slider, the other two hold their stored values, so the
render always reflects all three at once. Note the `else` in `onValueChange` -
saturation is the fall-through case, so any future fourth mode would silently
write into `saturateValue`.

`SELECT_MODE_EXPOSURE` is a misnomer: the constant is named exposure but drives
`.saturate()`, matching the 饱和度 label in the UI. Rename it before this
becomes someone else's bug.

**The centre-zero slider — `entry/src/main/ets/component/SliderComponent.ets`** (as shipped)

```typescript
@Link @Watch('onReturnChange') returnValue: number;      // -100..100, owned by the page
@State @Watch('onChange') value: number = this.trackWidth / 2;   // 0..trackWidth, pixels

aboutToAppear(): void {
  this.value = this.returnValue * this.trackWidth / 200 + this.trackWidth / 2;
}

onChange() {                                  // pixels -> value, clamped
  let value = (this.value - this.trackWidth / 2) / (this.trackWidth / 200);
  if (value > 100) {
    this.returnValue = 100;
  } else if (value < -100) {
    this.returnValue = -100;
  } else {
    this.returnValue = Math.floor(value);
  }
}

onReturnChange() {                            // value -> pixels
  this.value = this.trackWidth / 2 + this.returnValue * (this.trackWidth / 200);
}

Row()                                         // the blue fill, growing from the centre
  .height(6)
  .width((this.value < 0 || this.value > this.trackWidth) ? this.trackWidth / 2 :
    Math.abs(this.value - this.trackWidth / 2))
  .margin({
    left: this.value < 0 ? 0 : this.value > this.trackWidth ? this.trackWidth / 2 :
      (this.value - this.trackWidth / 2) > 0 ? this.trackWidth / 2 : this.value
  })
  .backgroundColor('#5291FF');
```

**Two `@Watch`es facing each other is the interesting part.** `value` is
pixels, `returnValue` is the domain the page cares about, and each watcher
converts into the other. That makes the control genuinely two-way: dragging
moves `value` and updates the page, while the reset icon writing
`this.adjustValue = 0` moves the handle back to centre through `onReturnChange`.
The clamp lives on the pixel-to-value side only, and `value` is deliberately left
unclamped - the geometry expressions handle out-of-range positions instead,
which is why every width and margin carries those nested ternaries.

Why hand-roll this at all when ArkUI ships `Slider`? Because a centre-zero fill -
a bar that grows left *or* right from the midpoint - is not something the stock
`Slider` draws. The `PanGesture` captures `this.x = this.value` in
`onActionStart` and adds the cumulative `event.offsetX` in `onActionUpdate`, and
a sibling `TapGesture` sets `value` straight from `event.fingerList[0].localX` so
the track is tappable.

**The export — `entry/src/main/ets/pages/EffectPage.ets`** (corrected, see
`HW-18-0046`, `HW-18-0036`)

```typescript
SaveButton({ text: SaveDescription.EXPORT_TO_GALLERY })
  .width(65)
  .onClick(() => {
    this.uiContext.getComponentSnapshot().get('mainImage', { scale: 1, waitUntilRenderFinished: true })
      .then((pixelMap: image.PixelMap) => {
        this.savePixelMap = pixelMap;
        const imagePacker = image.createImagePacker();
        let packOptions: image.PackingOption = {
          format: 'image/png',                  // FIX: was image/jpeg, but the asset is created as png
          quality: 100
        };
        imagePacker.packToData(pixelMap, packOptions)
          .then((arrayBuffer: ArrayBuffer) => {
            PhotoUtils.saveImageToAlbum(this.uiContext, arrayBuffer);
          })
          .catch((err: BusinessError) => {      // FIX: the chain had no catch at all
            this.uiContext.getPromptAction().showToast({ message: $r('app.string.photo_save_false') });
          })
          .finally(() => {
            imagePacker.release();              // FIX: the packer was never released
          });
      });
  });
```

**`SaveButton` is a security component, and that is the reason there is no
permission in `module.json5`.** Clicking it grants the app a one-shot,
system-verified permission to write to the gallery - which only holds if the
button is genuinely visible and untampered; an obscured or zero-size
`SaveButton` fails at click time. So it must stay a real, visible control, not a
proxy triggered from elsewhere.

`waitUntilRenderFinished: true` is the load-bearing option in the snapshot call.
Without it `get()` can capture the node before the last attribute change has
been rendered, and the exported file silently misses the user's final slider
move. `scale: 1` captures at the component's on-screen size, which is also this
approach's ceiling: the export is a screen-resolution render of the photo, not
the original at full resolution.

The format mismatch is the shipped defect: the packer was told `image/jpeg`
while `PhotoUtils` creates the asset with extension `png`, so a JPEG bitstream
lands in a file called `.png`. Either half of the pair can be changed - packing
as PNG (above) keeps the lossless intent of `quality: 100`. The missing
`release()` is the same systematic that hits eight photography samples
(`HW-18-0036`).

**Writing the asset — `entry/src/main/ets/utils/PhotoUtils.ets`** (corrected,
see `HW-18-0046`)

```typescript
static async saveImageToAlbum(uiContext: UIContext, imageBuffer: ArrayBuffer) {
  let helper = photoAccessHelper.getPhotoAccessHelper(uiContext.getHostContext());
  let file: fs.File | undefined = undefined;
  try {
    let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');   // matches the packer
    file = await fs.open(uri, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    await fs.write(file.fd, imageBuffer);
    uiContext.getPromptAction().showToast({
      message: $r('app.string.photo_save_success'),
      duration: 2000
    });
  } catch (error) {                        // FIX: createAsset and fs.open were outside the try
    uiContext.getPromptAction().showToast({
      message: $r('app.string.photo_save_false'),
      duration: 2000
    });
  } finally {
    if (file) {                            // FIX: fs.close ran on a possibly-unopened file
      await fs.close(file.fd);
    }
  }
}
```

**`createAsset` is the first thing that can fail and was the one thing outside
the try.** It runs only because a `SaveButton` click authorised it, and it can
still reject - a full disk, a revoked grant, a call outside the security
component's short validity window - and in the shipped code that rejection skips
the failure toast entirely and surfaces as an unhandled rejection, because the
caller does not `.catch` either. Widening the try is a two-line change that turns
a silent failure into a message.

The extension passed to `createAsset` is what the gallery uses to type the
asset, which is why it has to agree with `PackingOption.format`. Two photography
samples get this wrong the same way; `ImageFilterProcessing` packs `image/jpeg`
into a `.png` temp path and a `png` asset as well (`HW-18-0046`).

## Permissions & config

**None declared.** `module.json5` has no `requestPermissions` block at all - the
gallery write is authorised entirely by the `SaveButton` security component at
click time. This is the pattern to prefer over
`ohos.permission.WRITE_IMAGEVIDEO` whenever saving is a user-initiated,
one-at-a-time action.

`EntryAbility` publishes the status-bar height into `AppStorage` under
`statusBar`, and the page reads it as `AppStorage.get<number>('statusBar')` into
a plain field, converting with `px2vp` for its top padding. Note this is read
once at construction, not `@StorageProp`, so it does not update if the avoid
area changes.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The edited photo is a bundled resource (`app.media.main_image`); there is no
  picker. Adapting this to a user photo means replacing the `Image` source only -
  everything else is unchanged.
- The export is a **component snapshot**, so its resolution is the rendered size
  of the `Image` on screen, not the source resolution. For a full-resolution
  export the same three effects have to be reapplied through an image-processing
  path (see `PHOTO-28`, ImageFilterProcessing).
- `SliderComponent.trackWidth` is passed as 280 from the page but defaults to
  350 in the component; the geometry is in raw vp with no responsive behaviour.
- The two right-hand mode icons (闪光灯, 光感) have no click handlers - they are
  layout only.

## Pitfalls

- **`HW-18-0046`** (B/low, confirmed): JPEG bytes are packed (`image/jpeg`) into
  a gallery asset created as `png`, so exported photos carry a wrong
  extension/MIME for format-sensitive consumers. On top of that `createAsset`
  and `fs.open` sit outside the try that raises the failure toast, and the
  snapshot/pack/save chain has no `.catch` - so a save failure is silent.
  Fix: match the format to the extension, widen the try, add the `.catch`. The
  same mismatch is in `ImageFilterProcessing`.
- **`HW-18-0036`** (B/low, confirmed): systematic - `image.createImagePacker()`
  is called on every save and never released, leaking a native packer instance
  per export. Eight photography samples share it; `CompressImages` is the one
  that releases correctly. Fix: `imagePacker.release()` in a `finally`.
- **`fs.close(file.fd)` runs in a `finally` on a file that may never have
  opened.** In the shipped code `file` is assigned outside the try, so if
  `fs.open` rejects the `finally` dereferences an undefined `file` - the same
  shape as `HW-18-0044` in `PHOTO-15`. Guard it.
- **`savePixelMap` is stored on the component and never released.** Every export
  overwrites the field with a fresh snapshot `PixelMap`; the old one is dropped
  without `release()`, so native bitmap memory accumulates per save. The field is
  not read anywhere - it can simply be deleted.
- **`SELECT_MODE_EXPOSURE` drives saturation.** The constant name and the effect
  disagree; the UI label (饱和度) follows the effect, not the name.
- **The reset icon resets only the selected channel,** which is correct, but
  there is no "reset all" and no way to see whether the other two channels are
  non-zero without visiting each mode.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-image-effect.md` - `brightness`, `contrast`, `saturate` and their value domains
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-image-effect
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton`, `SaveDescription`, the visibility rules that keep the grant valid
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentsnapshot.md` - `getComponentSnapshot().get`, `scale`, `waitUntilRenderFinished`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentsnapshot
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-photoAccessHelper-PhotoAccessHelper.md` - `createAsset` and the extension argument
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - saving to the gallery with a security component
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - the slider's drag handling
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `PHOTO-28` (ImageFilterProcessing) - the same adjustments applied to pixels for a full-resolution export, and the same format mismatch
