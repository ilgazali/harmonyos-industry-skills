---
id: LIFE-21
title: Photograph an address label and fill the form - Core Vision OCR into Natural Language entity extraction
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
sample: huawei_industry_tree/02_convenient_life/downloads/CourierAddressIdentification.zip
kits: ["@kit.CoreVisionKit", "@kit.NaturalLanguageKit", "@kit.MediaLibraryKit", "@kit.CameraKit", "@kit.ImageKit", "@kit.CoreFileKit", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["textRecognition.init", "textRecognition.recognizeText", "textRecognition.release", "textRecognition.VisionInfo", "textRecognition.TextRecognitionConfiguration", "textRecognition.TextRecognitionResult", "textProcessing.getEntity", EntityType, "photoAccessHelper.PhotoViewPicker", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.PhotoViewMIMETypes", "cameraPicker.pick", "cameraPicker.PickerProfile", "cameraPicker.PickerMediaType", "camera.CameraPosition", "fileIo.open", "fileIo.OpenMode", "image.createImageSource", "imageSource.createPixelMap", aboutToAppear, aboutToDisappear, TextArea, "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0150, HW-02-0151, HW-02-0152, HW-02-0153, HW-02-0154, HW-02-0155, HW-02-0156, HW-02-0157, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **photo to structured form**: the user snaps a shipping
label or picks one from the gallery, and the recipient name, phone number,
region and street land in separate fields.

It is a two-stage pipeline, and the split is the point - neither kit does the
whole job:

```
picker         -> a URI          gallery (PhotoViewPicker) or camera (cameraPicker.pick)
fileIo.open    -> a File
createImageSource -> createPixelMap -> a PixelMap
textRecognition.recognizeText(pixelMap)  -> one flat string     Core Vision Kit
textProcessing.getEntity(string, types)  -> typed entities      Natural Language Kit
                                         -> the four form fields
```

**Neither picker needs a permission.** `PhotoViewPicker` and `cameraPicker.pick`
are system UIs that hand back a URI the application is granted access to - which
is why this project's `module.json5` has no `requestPermissions` block at all,
and why it is the right way to reach the gallery or the camera when you do not
need a live preview.

The second stage is the same entity extraction as `LIFE-19`; take that card if
your input is already text. Take this one when it starts as an image.

## Feature checklist

- Two entry points: pick from the gallery, or take a photo.
- The chosen image is decoded and shown.
- OCR converts it to text, which appears in an editable `TextArea`.
- The user can also type or correct the text by hand.
- One button runs entity extraction and fills recipient name, phone, region and
  detailed address.
- The region is separated from the street by cutting at 区.
- The form is cleared before each extraction.
- The OCR engine is initialised when the page appears and released when it goes
  away.

## Architecture

One `entry` module, one page.

```
entry/src/main/ets
├── common/CommonConstants.ets   sizes, TOAST_DURATION 2000, DEFAULT_TEXT (a fake address)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/Index.ets              THE CARD: pickers, decode, OCR, extraction, the form
```

The documented tree matches the zip exactly.

**The OCR engine is a page-scoped resource.** `textRecognition.init()` in
`aboutToAppear` loads the model; `textRecognition.release()` in
`aboutToDisappear` unloads it. That pairing is right - the mistake is making
both lifecycle callbacks `async` (`HW-02-0152`).

**Two pickers, one loader.** Both entry points end at the same
`loadImage(uri)`, which opens the URI, decodes it and stores the `PixelMap`.
That is also where all three leaks live (`HW-02-0150`).

**The state is four strings plus the intermediate text:**

```
recognizeText   the OCR output, editable by the user in a TextArea
name / phone / address / detailAddress    the form, filled by extraction
```

`recognizeAddress()` clears the four, then extracts. Clearing first is correct -
the extractor only assigns what it finds - but the empty-input path substitutes
a hard-coded demo address instead of stopping (`HW-02-0154`).

## Implementation steps

1. **Initialise the OCR engine in `aboutToAppear` and release it in
   `aboutToDisappear`** - but do **not** make either callback `async`
   (`HW-02-0152`). Check the init result (`HW-02-0155`).
2. **Reach the image through a picker, not a permission.** `PhotoViewPicker` for
   the gallery, `cameraPicker.pick` for the camera; both return a URI with access
   already granted.
3. **Decode in a `try`/`finally`,** closing the descriptor and releasing the
   previous `ImageSource` and `PixelMap` (`HW-02-0150`).
4. **Call `recognizeText` immediately** - it is already asynchronous, so the
   two-second `setTimeout` buys nothing (`HW-02-0153`).
5. **Branch on the callback's `error` argument** (`HW-02-0151`). A callback API
   does not throw, so a surrounding `try` is dead code.
6. **Put the OCR output in an editable field.** OCR of a handwritten or creased
   label is imperfect; letting the user fix it before extraction is what makes
   the feature usable.
7. **Clear the form before extracting,** because the extractor is additive - and
   stop rather than inventing input when there is nothing to extract
   (`HW-02-0154`).
8. **Cut the region from the street at the first separator only**
   (`HW-02-0156`), keeping the separator on the region side (`HW-02-0157`).

## Verified snippets

All snippets are from `CourierAddressIdentification.zip`. Corrected forms are
marked.

**Engine lifecycle - `CourierAddressIdentification.zip#entry/src/main/ets/pages/Index.ets:57`** (corrected, see `HW-02-0152` and `HW-02-0155`)

```typescript
// FIX: the sample declares both of these async and awaits inside them.
aboutToAppear(): void {
  textRecognition.init()
    .then((initResult: number) => {
      this.ocrReady = (initResult === 0);          // FIX: sample only logs the result
      hilog.info(0x0000, 'OCR', 'init result: %{public}d', initResult);
    })
    .catch((err: BusinessError) => {
      hilog.error(0x0000, 'OCR', 'init failed: %{public}s', err.message);
    });
}

aboutToDisappear(): void {
  textRecognition.release()
    .then(() => { hilog.info(0x0000, 'OCR', 'released'); })
    .catch((err: BusinessError) => { hilog.error(0x0000, 'OCR', '%{public}s', err.message); });
}
```

**`init` / `release` is a real resource pair, not boilerplate.** `init` loads the
recognition model into the process; skipping `release` leaves it resident for the
life of the application.

**But do not `await` in `aboutToDisappear`.** The lifecycle guidance is explicit:
"You are not advised to use async await in aboutToDisappear. If asynchronous
operations (such as Promise or callback methods) are used in this lifecycle, the
custom component will be retained in the Promise closure until the callback
method is executed. This will prevent the custom component from being garbage
collected." The shipped version holds the whole page - including its undisposed
`PixelMap` - alive until the engine finishes unloading.

**Both pickers - same file, line 70 and line 96** (as shipped)

```typescript
// 选择图库识别
imagePicker(): void {
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  photoViewPicker.select(photoSelectOptions)
    .then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      let uris = photoSelectResult.photoUris;
      if (uris.length > 0) {
        await this.loadImage(uris[0]);
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.start_recognizing_images') });
        this.recognizeImageToText();
      }
    })
    .catch((err: BusinessError) => {
      hilog.error(0x0000, 'OCR', 'select failed, code %{public}d, message %{public}s', err.code, err.message);
    });
}

// 拍摄照片识别
async takePhoto(): Promise<void> {
  if (this.context) {
    let pickerProfile: cameraPicker.PickerProfile = {
      cameraPosition: camera.CameraPosition.CAMERA_POSITION_BACK
    };
    let pickerResult: cameraPicker.PickerResult = await cameraPicker.pick(this.context,
      [cameraPicker.PickerMediaType.PHOTO, cameraPicker.PickerMediaType.VIDEO], pickerProfile);
    if (pickerResult.resultCode === CommonConstants.CAMERA_PICKER_RESULT_CODE_FAILURE) {
      return;                                     // the user pressed X on the camera page
    } else if (pickerResult.resultCode === CommonConstants.CAMERA_PICKER_RESULT_CODE_SUCCESS) {
      await this.loadImage(pickerResult.resultUri);
      this.recognizeImageToText();
    }
  }
}
```

**Neither of these needs a permission,** and that is the reason to use them. The
system picker runs in its own process, shows its own UI, and returns a URI the
calling application has been granted read access to - so no
`ohos.permission.READ_IMAGEVIDEO` and no `ohos.permission.CAMERA`, and no
runtime request flow to get wrong. Reaching the gallery through
`photoAccessHelper`'s query APIs, or the camera through a live preview, would
need both.

`PhotoViewMIMETypes.IMAGE_TYPE` with `maxSelectNumber = 1` is what makes the
picker single-image and image-only.

`resultCode` is checked against named constants rather than a truthiness test -
`-1` for a cancelled pick is distinct from `0` for success, and treating cancel
as an error would show a failure toast every time the user backs out.

Note that `takePhoto` passes both `PHOTO` and `VIDEO` as accepted media types
even though only a photo can be OCR'd.

**Decoding - same file, line 119** (corrected, see `HW-02-0150`)

```typescript
private async loadImage(name: string): Promise<void> {
  // FIX: the sample releases nothing - each pick leaks a descriptor,
  //      an ImageSource and the previous PixelMap.
  this.chooseImage?.release();
  this.imageSource?.release();
  const fileSource = await fileIo.open(name, fileIo.OpenMode.READ_ONLY);
  try {
    this.imageSource = image.createImageSource(fileSource.fd);
    this.chooseImage = await this.imageSource.createPixelMap();
  } finally {
    fileIo.closeSync(fileSource);                 // FIX: the descriptor is never closed
  }
}
```

**`createImageSource(fd)` does not take ownership of the descriptor.** The
`File` returned by `fileIo.open` must be closed explicitly; the decode has
already read what it needs by the time `createPixelMap` resolves. A full-size
camera photo is several megabytes of decoded bitmap, so the `PixelMap` matters as
much as the descriptor.

**OCR - same file, line 126** (corrected, see `HW-02-0151` and `HW-02-0153`)

```typescript
recognizeImageToText(): void {
  if (!this.chooseImage) {
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.image_recognition_fail'), duration: CommonConstants.TOAST_DURATION });
    return;
  }
  const visionInfo: textRecognition.VisionInfo = { pixelMap: this.chooseImage };
  const textConfiguration: textRecognition.TextRecognitionConfiguration = {
    isDirectionDetectionSupported: false          // the label is upright; skip the rotation pass
  };
  // FIX: the sample wraps this in setTimeout(..., CommonConstants.TOAST_DURATION)
  //      and in a try that cannot see an asynchronous failure.
  textRecognition.recognizeText(visionInfo, textConfiguration,
    (error: BusinessError, data: textRecognition.TextRecognitionResult) => {
      if (error && error.code !== 0) {            // FIX: sample has only `if (error?.code === 0)`
        hilog.error(0x0000, 'OCR', 'recognizeText failed: %{public}d %{public}s', error.code, error.message);
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_fail') });
        return;
      }
      this.recognizeText = data.value;            // one flat string, all lines joined
    });
}
```

**`isDirectionDetectionSupported: false` is a deliberate cost saving.** Turning
it on makes the engine try rotated orientations, which is worth it for a photo
that may be sideways and wasted for a label the user framed deliberately.

**`data.value` is the whole recognised text as one string** - the result also
carries per-block and per-line geometry, but the flat string is all the entity
extractor needs.

**A callback API signals failure through its first argument.** The shipped `try`
around the call can only catch a synchronous throw, which is not how
`recognizeText` fails; and the callback's `if (error?.code === 0)` has no `else`,
so every failure is silent.

**Extraction and the form - same file, line 161** (corrected, see `HW-02-0154`, `HW-02-0156` and `HW-02-0157`)

```typescript
recognizeAddress(): void {
  this.clearHistoryData();                        // the extractor is additive - clear first
  if (!this.recognizeText) {
    // FIX: the sample assigns CommonConstants.DEFAULT_TEXT here, a fabricated
    //      name/phone/address that then fills the form as if it were a result.
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.image_recognition_fail') });
    return;
  }
  this.doEntityRecognition();
}

doEntityRecognition(): void {
  textProcessing.getEntity(this.recognizeText, {
    entityTypes: [EntityType.NAME, EntityType.PHONE_NO, EntityType.LOCATION]
  }).then((result) => {
    this.formatEntityResult(result);
  }).catch((err: BusinessError) => {              // this one the sample gets right
    hilog.error(0x0000, 'OCR', 'getEntity errorCode: %{public}d %{public}s', err.code, err.message);
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_fail') });
  });
}

formatEntityResult(entities: textProcessing.Entity[]): void {
  if (!entities || !entities.length) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_fail') });
    return;
  }
  for (let i = 0; i < entities.length; i++) {
    const entity = entities[i];
    if (entity.type === EntityType.NAME) {
      this.name = entity.text;
    } else if (entity.type === EntityType.PHONE_NO) {
      this.phone = entity.text;
    } else if (entity.type === EntityType.LOCATION) {
      // FIX: the sample uses split(DISTRICT) and takes [0] and [1], losing
      //      everything after a second 区 (and the document's version also
      //      drops the 区 itself - HW-02-0157)
      const at = entity.text.indexOf(DISTRICT);
      if (at >= 0) {
        this.address = entity.text.slice(0, at + 1);
        this.detailAddress = entity.text.slice(at + 1);
      } else {
        this.detailAddress = entity.text;
      }
    }
  }
}
```

**`doEntityRecognition`'s `.catch` is the one error path this sample handles
properly** - logged and surfaced to the user. It is worth contrasting with the
OCR callback above, which handles none.

**`clearHistoryData()` before extracting is required, not tidy.** The loop only
assigns fields it finds, so without the reset a second extraction of a label with
no name keeps the previous one's name.

**`split('区')` is the wrong cut.** 区 appears inside ordinary address
components - 小区 is the standard word for a residential compound - so
`杭州市西湖区某小区3号楼` splits into three fragments and the code reads only two.
`indexOf` cuts once and keeps the rest.

**The user can edit the OCR output before extracting,** which is the design
decision that makes the whole flow tolerable: `recognizeText` is bound to a
`TextArea`, so a misread digit in the phone number can be fixed before the
entity pass rather than after.

## Permissions & config

**None.** `CourierAddressIdentification.zip#entry/src/main/module.json5`
declares no `requestPermissions` block:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

That is the notable part of the configuration, and it is correct:

- `photoAccessHelper.PhotoViewPicker` is a system picker; the returned URI comes
  with access. No `ohos.permission.READ_IMAGEVIDEO`.
- `cameraPicker.pick` is a system camera UI. No `ohos.permission.CAMERA`.
- Core Vision text recognition and Natural Language entity extraction both run
  on device. No network permission, and no AppGallery Connect service to enable -
  unlike the Map Kit samples in this industry.

Root `build-profile.json5` targets `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 136-138).
- `textRecognition.init()` must complete before `recognizeText` is called; the
  sample does not enforce this (`HW-02-0155`).
- OCR runs on the whole image and returns one flat string, so a label with the
  sender's and the recipient's blocks side by side yields both, interleaved by
  reading order.
- The entity extraction is tuned for **Chinese** text, and the region/street cut
  is the literal character 区 - an address in a 县 or a 市辖区 without that
  character falls entirely into the detailed-address field.
- Only `NAME`, `PHONE_NO` and `LOCATION` are requested; a postcode or a company
  name in the label is not extracted.
- `isDirectionDetectionSupported: false` means a sideways photo is not
  auto-rotated before recognition.
- `takePhoto` accepts `VIDEO` as well as `PHOTO` from the camera picker, though
  only a photo can be recognised.
- The 保存地址 button has no handler; the sample ends at filling the form.

## Pitfalls

- **`HW-02-0150` - `loadImage` releases nothing.** The file descriptor is never
  closed and the previous `ImageSource` and `PixelMap` are never released, so
  every image the user tries leaks a descriptor and a decoded bitmap.
- **`HW-02-0151` - the OCR callback handles only `error?.code === 0`,** and the
  `try` around a callback API cannot see an asynchronous failure. A failed
  recognition changes nothing on screen and logs nothing.
- **`HW-02-0152` - both lifecycle callbacks are `async` with an `await` inside.**
  The guidance says this retains the component in the promise closure and
  prevents it being collected.
- **`HW-02-0153` - recognition is delayed two seconds by reusing
  `TOAST_DURATION` as a `setTimeout` delay.** One constant now means both the
  toast dwell time and the OCR start delay.
- **`HW-02-0154` - an empty input is replaced with a hard-coded fake address**
  (`隆隆，1888888888888，浙江省杭州市西湖区某小区`), which then fills the form
  indistinguishably from a real result - and the phone number in it has thirteen
  digits.
- **`HW-02-0155` - `textRecognition.init()`'s result is logged and never
  checked,** so a device that cannot run OCR fails silently later.
- **`HW-02-0156` - `split('区')` keeps only the first two fragments,** losing the
  tail of any address containing the character twice - which 小区 makes common.
- **`HW-02-0157` - the document's snippet drops the 区 that the sample
  re-appends,** so a reader who copies it gets a truncated region name.
- **Do not add camera or gallery permissions for this.** The pickers are system
  UIs; declaring `ohos.permission.CAMERA` here would be an unnecessary
  permission, not a fix.
- **Do not extract without clearing first.** The entity loop only assigns what it
  finds.
- **Do let the user edit the OCR text.** Recognition of a real shipping label is
  imperfect, and the `TextArea` between the two stages is what makes the pipeline
  usable.

## References

- `documentation/harmonyos-references/07_ai/core-vision-text-recognition-api.md` - `textRecognition.init`, `recognizeText`, `release`, `VisionInfo`, `TextRecognitionConfiguration.isDirectionDetectionSupported`, `TextRecognitionResult.value`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/core-vision-text-recognition-api
- `documentation/harmonyos-references/07_ai/natural-language-text-processing-api.md` - `textProcessing.getEntity`, `Entity`, `EntityType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/natural-language-text-processing-api
- `documentation/harmonyos-references/04_media/js-apis-photoaccesshelper.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`, `PhotoViewMIMETypes`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/04_media/js-apis-camerapicker.md` - `cameraPicker.pick`, `PickerProfile`, `PickerMediaType`, `PickerResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camerapicker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `createPixelMap`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `fileIo.open`, `OpenMode`, `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/03_application-framework/arkts-page-custom-components-lifecycle.md` - the warning against `async`/`await` in `aboutToDisappear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-page-custom-components-lifecycle
- `LIFE-19` - the same entity extraction starting from pasted text, with the same `split('区')` defect
- `LIFE-22`, `LIFE-23` - the industry's card-scanning scenarios, which use a purpose-built control instead of raw OCR
- `LIFE-01` - the industry shell this page would sit in
