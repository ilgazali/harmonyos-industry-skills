---
id: PHOTO-20
title: Rotate and flip a still image - PixelMap rotateSync/flipSync behind a NavDestination editor
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
sample: huawei_industry_tree/18_photography/downloads/ImageRotateAndFlip.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, fileIo, hilog, image, photoAccessHelper, window, "PixelMap.rotateSync", "PixelMap.flipSync", createPixelMapSync, readPixelsToBufferSync, ImagePacker, SaveButton, NavPathStack, NavDestination]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0056, HW-18-0057, HW-18-0058, HW-18-0059, HW-18-0004, HW-18-0024, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card for the **four buttons every photo editor has** — rotate left 90°,
rotate right 90°, flip horizontal, flip vertical — and for the round trip that
carries an edited image back to the page that opened the editor.

The image work itself is two API calls, `rotateSync(angle)` and
`flipSync(horizontal, vertical)`, both of which mutate the PixelMap in place. That
in-place mutation is the interesting part and the trap: ArkUI's `@State` compares
references, and a PixelMap whose bytes changed under a reference ArkUI already
holds is not a state change. Any editor built on the `image` kit's `*Sync`
operators — crop, scale, translate, opacity, colour-space conversion — hits the
same wall, so the clone-per-operation discipline this card describes generalises
across the whole family.

The second reusable piece is the navigation: `pushPathByName(name, param, onPop)`
carries the PixelMap into a `NavDestination` and the `onPop` callback carries the
edited one back. That is the right shape whenever an editor is a separate route
rather than a modal over the same page.

## Feature checklist

- Main page: a tabbed shell with an 添加图片 (add image) card that opens
  `PhotoViewPicker`; cancelling the picker does nothing and raises nothing.
- The picked image appears as a 94x94 thumbnail in the 图文 tab.
- Tapping the thumbnail pushes the editor route, carrying the PixelMap.
- Editor: a large preview, a row of four transform icons, and a decorative second row.
- Rotate left / rotate right turn the preview by 90° in the pressed direction; the
  preview updates on every press.
- Flip horizontal / flip vertical mirror the preview.
- The transforms compose — three left rotations equal one right rotation.
- The icon shrinks from 24 to 22 vp while a touch is held.
- `SaveButton` writes the current image to the gallery as a PNG, without any
  album permission.
- The back arrow pops the edited PixelMap back to the main page, which shows it.

## Architecture

One `entry` module, `Navigation` + `NavDestination` with a `routerMap`, two pages
and two small utilities.

```
entry/src/main/ets
├── common/Constants.ets           layout maths, TURN_LEFT = -90, TURN_RIGHT = 90
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── IconModel.ets              ICON_MODELS1 (the four transforms), ICON_MODELS2, ICON_MAIN
│   └── ImageModel.ets             a one-field wrapper: { pixelMap }
├── pages
│   ├── ImageEditPage.ets          @Builder + NavDestination: preview, four icons, SaveButton
│   └── MainPage.ets               @Entry: Navigation host, tabs, picker, thumbnail
└── utils
    ├── ImageUtil.ets              picker, buffer read, rotate, flip, clonePixelMap
    └── Logger.ets                 hilog wrapper
```

The documented tree matches the zip exactly.
`resources/base/profile/route_map.json` maps the name `ImageEditPage` to
`imageEditPageBuilder`, so the editor is reachable by name without an import chain
back to the page module.

**The design decision worth copying** is `ImageModel` — a class with a single
`pixelMap` field and nothing else:

```typescript
export class ImageModel {
  pixelMap: image.PixelMap;
  constructor(pixelMap: image.PixelMap) { this.pixelMap = pixelMap; }
}
```

`NavPathStack` parameters are typed `Object`, and `PopInfo.result` likewise, so
both directions of the round trip need a nominal type to cast to. Wrapping the
PixelMap gives one — `popInfo.result as ImageModel` is checkable where
`popInfo.result as image.PixelMap` would be a lie about a native handle. The cost
is one allocation per navigation; the benefit is that adding a second field later
(the rotation angle, an undo stack) does not change either signature.

**The decision worth avoiding** is `ImageUtil.rotate`/`flip` returning the same
object they were handed. `let pixelMapTemp = pixelMap; pixelMapTemp.rotateSync(angle);
return pixelMapTemp;` reads like a functional transform and is not one — the
`Temp` name suggests a copy that never happens (`HW-18-0056`). The file already
contains the missing piece, `clonePixelMap`, and never calls it.

## Implementation steps

1. **Declare the route** in `route_map.json` with a `buildFunction`, and export a
   matching `@Builder` from the editor file.
2. **Provide the stack once** — `@Provide('pathStack')` on the main page,
   `@Consume('pathStack')` in the editor — so no page constructs its own.
3. **Pick with `PhotoViewPicker`** and guard on
   `photoSelectResult.photoUris.length > 0` before indexing (this sample is the one
   that gets it right; see `HW-18-0024`).
4. **Read the file into an `ArrayBuffer` and decode from the buffer,** closing the
   fd in a `finally`. Use `&&` in the URI guard, not `||` (`HW-18-0057`).
5. **Decode with `editable: true`** — `rotateSync` and `flipSync` throw on a
   read-only PixelMap.
6. **Push with `pushPathByName(name, new ImageModel(map), onPop)`** and read the
   result back through `popInfo.result as ImageModel`.
7. **Clone before or after each transform** so a new instance is assigned to
   `@State`, otherwise the preview does not refresh (`HW-18-0056`).
8. **Drive the transforms from `onTouch`, not `onClick`,** if you want a pressed
   state: `TouchType.Down` sets the flag, `TouchType.Up` runs the operation and
   clears it.
9. **Export with `SaveButton` + `createAsset`,** and remove the `WRITE_IMAGEVIDEO`
   declaration — this flow needs no permission (`HW-18-0004`, `HW-18-0058`).

## Verified snippets

All snippets are from `ImageRotateAndFlip.zip`. Corrected forms are marked.

**The transforms — `entry/src/main/ets/utils/ImageUtil.ets`** (corrected, see `HW-18-0056`)

```typescript
async rotate(angle: number, pixelMap: image.PixelMap): Promise<image.PixelMap> {
  let pixelMapTemp = await this.clonePixelMap(pixelMap);   // FIX: sample aliases instead of cloning
  try {
    pixelMapTemp.rotateSync(angle);
  } catch (e) {
    let error = e as BusinessError;
    LOGGER.error(`rotate pixelMapTemp error. code is ${error.code}, message is ${error.message}`);
  }
  return pixelMapTemp;
}

async flip(horizontal: boolean, vertical: boolean, pixelMap: image.PixelMap): Promise<image.PixelMap> {
  let pixelMapTemp = await this.clonePixelMap(pixelMap);   // FIX: same
  try {
    pixelMapTemp.flipSync(horizontal, vertical);
  } catch (e) {
    let error = e as BusinessError;
    LOGGER.error(`flip pixelMapTemp error. code is ${error.code}, message is ${error.message}`);
  }
  return pixelMapTemp;
}
```

**`rotateSync` and `flipSync` return `void` and mutate the receiver.** That is the
whole reason the shipped code looks correct and is not: `let pixelMapTemp = pixelMap`
is an alias, so the function mutates the caller's object and hands back the same
reference, and `this.currentImage = pixelMap` at the call site assigns a variable to
itself. ArkUI's `@State` observes reference identity, not the bytes behind a native
handle, so nothing re-renders. The preview does eventually update in the sample —
but only incidentally, when some other state change forces a rebuild.

`clonePixelMap` (already present, never called) is exactly the missing step: it
reads the pixels into a buffer, copies the source's `pixelFormat` and `size` into
`InitializationOptions`, and calls `createPixelMapSync` — a genuinely new instance.
Making `rotate` and `flip` async is the price of the clone being async; the call site
already lives in a `TouchType.Up` handler and can await.

Note `Constants.TURN_LEFT` is `-90` and `TURN_RIGHT` is `+90`: `rotateSync` takes
clockwise degrees, so "left" is negative.

**The clone — same file** (as shipped)

```typescript
async clonePixelMap(pixelMap: image.PixelMap, desiredPixelFormat?: image.PixelMapFormat) {
  let imageInfo: image.ImageInfo = pixelMap.getImageInfoSync();
  let buffer = new ArrayBuffer(pixelMap.getPixelBytesNumber());
  pixelMap.readPixelsToBufferSync(buffer);
  let options: image.InitializationOptions = {
    editable: true,
    srcPixelFormat: imageInfo.pixelFormat,                     // format the buffer is in
    pixelFormat: desiredPixelFormat ?? imageInfo.pixelFormat,  // format the new map should be in
    size: imageInfo.size
  };
  return image.createPixelMapSync(buffer, options);
}
```

**`srcPixelFormat` and `pixelFormat` are two different things and both must be set.**
The first tells `createPixelMapSync` how to read the bytes it was given; the second
says what the resulting PixelMap should hold. Defaulting the second to the first with
`??` makes the plain call a faithful copy while leaving the door open for a
conversion — pass `RGBA_8888` and the same helper becomes a format normaliser.
`editable: true` is not optional here: the whole point of the clone is that a
`*Sync` transform will be applied to it.

Copying `size` from `getImageInfoSync()` rather than tracking width and height
separately is what keeps the helper correct after a 90° rotation has already
swapped them.

**Applying a transform on touch — `entry/src/main/ets/pages/ImageEditPage.ets`** (corrected, see `HW-18-0056`)

```typescript
.onTouch(async (event: TouchEvent) => {
  if (event) {
    if (event.type === TouchType.Down) {
      this.isClick[index] = true;                     // drives the 24 -> 22 vp icon shrink
    } else if (event.type === TouchType.Up) {
      if (!this.currentImage) {                       // FIX: sample writes three || comparisons
        return;
      }
      let pixelMap: image.PixelMap | undefined = undefined;
      switch (index) {
        case 0:
          pixelMap = await IMAGE_UTIL.rotate(Constants.TURN_LEFT, this.currentImage);
          break;
        case 1:
          pixelMap = await IMAGE_UTIL.rotate(Constants.TURN_RIGHT, this.currentImage);
          break;
        case 2:
          pixelMap = await IMAGE_UTIL.flip(true, false, this.currentImage);
          break;
        case 3:
          pixelMap = await IMAGE_UTIL.flip(false, true, this.currentImage);
          break;
      }
      if (pixelMap) {
        this.currentImage = pixelMap;                 // now a genuinely new instance
      } else {
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.editing_failed') });
      }
      this.isClick[index] = false;
    }
  }
});
```

**`onTouch` rather than `onClick` is a deliberate choice, not an accident.** The
icon shrinks by 2 vp while held, which needs the down and up edges separately;
`onClick` only fires once, after both. The cost is that the handler must do the
click's job too — there is no cancel branch, so dragging off the icon before
lifting still applies the transform.

`isClick` is a `boolean[]` in `@State` indexed by icon. Element assignment into a
state array does trigger observation in ArkUI, which is why this works — but it is
worth noting that this is the one place the sample relies on deep observation, and
it is the same mechanism that does *not* apply to the PixelMap's internal bytes.

The four cases map one-to-one onto `ICON_MODELS1`, so the `switch` is really a
table lookup written out longhand. With the icon list already declared as data, a
`transform` field on `IconModel` would remove the switch entirely.

**Export — same file** (as shipped)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (result === SaveButtonOnClickResult.SUCCESS) {
      let imagePackerApi: image.ImagePacker = image.createImagePacker();
      let packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
      if (!this.currentImage) {
        return;
      }
      let arrayBuffer = await imagePackerApi.packToData(this.currentImage, packOpts);
      let context: Context = this.getUIContext().getHostContext() as common.UIAbilityContext;
      let helper = photoAccessHelper.getPhotoAccessHelper(context);
      // The createAsset interface is invoked to create an image file
      // within 10 seconds after the onClick is triggered.
      let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
      let file: fs.File | undefined = undefined;
      try {
        file = fs.openSync(uri, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
        fs.writeSync(file.fd, arrayBuffer);
        this.getUIContext().getPromptAction().showToast({
          message: $r('app.string.save_successfully'), duration: Constants.DURATION_2000
        });
      } catch (err) {
        let error = err as BusinessError;
        LOGGER.error(`Failed to save the image. code: ${error.code}, message: ${error.message}`);
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
      } finally {
        if (file !== undefined) {
          fs.closeSync(file.fd);                       // the fd IS closed here — unlike PHOTO-18
        }
      }
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
    }
  });
```

**This is the best-behaved export in the industry's photography samples.** The fd
is opened inside the `try` and closed in the `finally`, both toast paths are covered,
and the ten-second `createAsset` grant window is respected — the asset is created
immediately after the tap and the write, which is unbounded, happens afterwards
against the returned URI. `quality: 98` on `format: 'image/png'` is inert (PNG is
lossless) but harmless.

The one thing still missing is `imagePackerApi.release()` — the same systematic
`HW-18-0036` that hits eight samples, though this one was not filed against
`ImageRotateAndFlip`. And, as everywhere in this pack, the `WRITE_IMAGEVIDEO`
declaration in `module.json5` is dead weight: `SaveButton` grants the write
(`HW-18-0004`).

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.WRITE_IMAGEVIDEO",
    "reason": "$string:requestPermission",
    "usedScene": {
      "abilities": ["FormAbility"],        // <- no such ability exists in this module
      "when": "inuse"
    }
  }
]
```

- **Delete the whole block.** `WRITE_IMAGEVIDEO` is a restricted (ACL) permission,
  nothing requests it at runtime, and both the read (`PhotoViewPicker`) and the write
  (`SaveButton` + `createAsset`) are permission-free by construction (`HW-18-0004`).
- If it is kept for some other reason, `usedScene.abilities` must name a real
  ability. The module declares `EntryAbility` and `EntryBackupAbility`; `FormAbility`
  is copy-paste from a template and makes the scene binding meaningless
  (`HW-18-0058`).
- `routerMap: "$profile:route_map"` is required for `pushPathByName('ImageEditPage', ...)`
  to resolve.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `rotateSync` and `flipSync` need API 12+.
- `rotateSync`/`flipSync` operate on the pixel buffer, so a 90° rotation of a large
  image reallocates and re-lays-out the whole buffer synchronously on the UI thread.
  For camera-resolution images this is a visible hitch; there is no async variant, so
  a taskpool is the only way out (see `PHOTO-18` for the pattern).
- Both throw if the PixelMap was not decoded with `editable: true`.
- The preview also carries `.rotate({ angle: this.imageAngle })`, but `imageAngle` is
  a plain field initialised to 0 and never written — a leftover from a render-transform
  approach that was replaced by the pixel-level one.
- The preview height is `displayHeight - 350` vp, a magic constant that assumes one
  screen size; the icon row margins are likewise derived from
  `display.getDefaultDisplaySync().width` at `aboutToAppear` and never recomputed on
  a resize.
- The second icon row (裁剪 / 收藏 / 旋转 / 录音) and the scale bar above it are
  decoration — every one of them toasts 仅做展示 ("display only").
- The main page's three other bottom tabs and four of five inner tabs are the same
  placeholder.

## Pitfalls

- **`HW-18-0056` (B/medium, probable) — rotate/flip mutate in place and return the
  same reference.** `ImageUtil.rotate`/`flip` alias their argument, call the `*Sync`
  operator on it, and return it; `ImageEditPage` then assigns
  `this.currentImage = pixelMap` — the identical reference. `@State` observes
  references, and ArkUI does not watch a PixelMap's internal bytes, so the preview does
  not reliably re-render after a transform. Fix: use `clonePixelMap`, which is already
  in the file and never called.
- **`HW-18-0057` (B/low, confirmed) — an always-true guard.**
  `if (uri !== '' || uri !== undefined || uri !== null)` passes for every value,
  including `''` and `undefined`, because no single value can be unequal to all three
  — so the `else { return undefined; }` branch is dead and empty URIs fall into
  `openSync`, saved only by the surrounding try/catch. Fix: `if (!uri) return undefined;`.
- **`HW-18-0058` (B/low, confirmed) — `usedScene` names a nonexistent `FormAbility`.**
  The module declares only `EntryAbility` and `EntryBackupAbility`. The binding is
  meaningless and misleads a permission audit. Fix: point it at `EntryAbility`, or
  better, drop the declaration per `HW-18-0004`.
- **`HW-18-0059` (D/low, confirmed) — the doc's `flip` signature is wrong.** Step 3
  of the page declares `flip(...): Promise<image.PixelMap>` over a body that is
  identical to the sample's synchronous `flip(...): image.PixelMap`. Copying the
  documented signature yields a type error or a pointless `await` on a plain value.
  Fix: correct the doc's return type. (Once `HW-18-0056` is fixed by cloning, the
  function genuinely does become async — the doc would be right for the wrong reason.)
- **`HW-18-0004` (D/medium, confirmed) — restricted album permission declared but
  never used.** Systematic across nine photography samples, `20_image_rotate_and_flip`
  among them. Ordinary apps cannot ship `WRITE_IMAGEVIDEO`, and a sample built
  entirely around the permission-free security-control flow should not teach otherwise.
  Fix: delete the `requestPermissions` block.
- **`HW-18-0024` (B/low, confirmed) — not a defect here; this sample is the reference.**
  The systematic picker-cancel finding covers six samples that index `photoUris[0]`
  unconditionally. `ImageRotateAndFlip` is called out in that finding as the one
  containing the correct guard:
  `if (photoSelectResult && photoSelectResult.photoUris && photoSelectResult.photoUris.length > 0)`.
  Copy this version, not the others'.

## References

- `huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md` — the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` — `rotateSync`, `flipSync`, `readPixelsToBufferSync`, `getImageInfoSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-guides/05_media/image-pixelmap-operation.md` — the PixelMap operation set and the clone recipe
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-pixelmap-operation
- `documentation/harmonyos-guides/05_media/image-editing-arkts.md` — editable decoding and the in-place operators
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-editing-arkts
- `documentation/harmonyos-references/04_media/arkts-apis-image-ImagePacker.md` — `packToData`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` — `SaveButtonOnClickResult` and the grant window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/04_system/savebutton.md` — why this flow needs no permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` — `select` and `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` — `pushPathByName`, `PopInfo`, `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `PHOTO-18` — the same `SaveButton` export, and the taskpool pattern for moving heavy pixel work off the UI thread
