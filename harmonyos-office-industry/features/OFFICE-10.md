---
id: OFFICE-10
title: Mail attachments - add from gallery, camera or files through system pickers, then preview
industry: 05_office
doc: huawei_industry_tree/05_office/docs/10_email_attachment.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
sample: huawei_industry_tree/05_office/downloads/EmailAttachment.zip
kits: ["@kit.MediaLibraryKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.PreviewKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["photoAccessHelper.PhotoViewPicker", "PhotoViewPicker.select", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.PhotoViewMIMETypes", "photoAccessHelper.PhotoSelectResult", "cameraPicker.pick", "cameraPicker.PickerProfile", "cameraPicker.PickerMediaType", "camera.CameraPosition", "picker.DocumentViewPicker", "DocumentViewPicker.select", "picker.DocumentSelectOptions", "filePreview.openPreview", "filePreview.PreviewInfo", "fileUri.FileUri", "fs.lstatSync", "uniformTypeDescriptor.getUniformDataTypeByFilenameExtension", "uniformTypeDescriptor.getTypeDescriptor", "TypeDescriptor.mimeTypes", CustomDialogController, "@CustomDialog", bindPopup, "resourceManager.getStringValue", "UIContext.px2vp", "UIContext.getPromptAction", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0057, HW-05-0058, HW-05-0059, HW-05-0060, HW-05-0061, HW-05-0062, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a compose screen needs an **attachment picker with three
sources** - photo gallery, live camera capture and the local file system - plus
an attachment list that shows name and size and opens each item in the system
previewer.

The point of the pattern is that all three sources are **system pickers**, so
the app needs **no permission at all**: no gallery permission, no camera
permission, no storage permission. Each picker runs outside the app, returns a
URI for exactly what the user chose, and that URI is the only thing the app ever
holds.

The second half - turning a URI into a display name, a human size and a MIME
type - is where the sample is worth reading, and where most of its defects are.

## Feature checklist

The application must be able to:

- Offer a bottom sheet with three attachment sources and a cancel entry.
- Pick a single image from the gallery with `PhotoViewPicker`.
- Capture a photo with `cameraPicker.pick` and take the result URI.
- Pick an arbitrary local file with `DocumentViewPicker`.
- Reject a duplicate URI with a toast instead of adding it twice.
- Show each attachment as name plus a human-readable size (B / KB / MB).
- Derive the MIME type from the file-name extension.
- Open an attachment in the system previewer, and report a preview failure.
- Remove an attachment from the list.
- Show a one-time hint popup on the attach icon.

## Architecture

Single `entry` HAP, three source files carrying the feature:

| File | Responsibility |
| --- | --- |
| `pages/EmailAttachment.ets` | `@Entry`; the compose form, the `fileList` state, the three picker callbacks, `addToFileList` / `cancel`, and the `CustomDialogController` |
| `components/PickerDialog.ets` | `@CustomDialog` bottom sheet; receives the three picker callbacks as `Function` props and closes itself after invoking one |
| `components/Attachment.ets` | one attachment row: resolves name, size and MIME type in `aboutToAppear`, opens the preview on tap, calls back `cancel` on the close icon |
| `constant/CommonConstant.ets` | size thresholds and unit labels |
| `utils/Logger.ets` | hilog wrapper |

The state contract is deliberately thin: the page owns
`@State fileList: Array<string>` - **URIs only** - and passes callbacks down.
`Attach` receives `uri` and the `cancel` function; everything else it derives
itself. Removal is a reassignment, not a mutation:
`this.fileList = this.fileList.filter(item => item !== uri)`.

Picker flow:

```
attach icon onClick -> dialogController.open()
  PickerDialog
    gallery -> photoPick()  -> new PhotoViewPicker().select(PhotoSelectOptions{IMAGE_TYPE, maxSelectNumber:1})
                            -> addToFileList(photoSelectResult.photoUris[0])
    camera  -> cameraPick() -> cameraPicker.pick(context, [PickerMediaType.PHOTO], {cameraPosition: BACK})
                            -> addToFileList(pickerResult.resultUri.toString())
    file    -> filePick()   -> new DocumentViewPicker().select(new DocumentSelectOptions())
                            -> addToFileList(documentSelectResult[0])
  each -> controller?.close()

Attach.aboutToAppear
  new fileUri.FileUri(uri) -> .path -> fs.lstatSync(path).size -> B / KB / MB label
                           -> .name -> filename
  filename extension -> utd.getUniformDataTypeByFilenameExtension -> typeId
                     -> utd.getTypeDescriptor(typeId).mimeTypes[0] -> mimeType

Attach row onClick -> filePreview.openPreview(context, { title, uri, mimeType })
```

## Implementation steps

1. **Declare no permission.** All three sources are system pickers and
   `filePreview` consumes a URI the user authorised, so `module.json5` needs no
   `requestPermissions` block - and the sample has none. The document likewise
   has no 权限说明 section; that is correct, not an omission.
2. **Hold URIs, not files.** Keep `@State fileList: Array<string>` and let each
   row derive its own metadata, so adding a source later costs one callback.
3. **Gallery.** `new photoAccessHelper.PhotoSelectOptions()` with
   `MIMEType = PhotoViewMIMETypes.IMAGE_TYPE` and `maxSelectNumber = 1`, then
   `new photoAccessHelper.PhotoViewPicker().select(options)`; take
   `photoUris[0]`.
4. **Camera.** `cameraPicker.pick(context, [cameraPicker.PickerMediaType.PHOTO],
   { cameraPosition: camera.CameraPosition.CAMERA_POSITION_BACK })`; take
   `pickerResult.resultUri.toString()`.
5. **Files.** `new picker.DocumentViewPicker().select(new
   picker.DocumentSelectOptions())`; take element `0`.
6. **Attach `.catch()` to every picker.** The sample already does this for all
   three - dismissing a picker rejects, and that is a normal outcome.
7. **De-duplicate on add.** `if (!this.fileList.includes(uri))` before pushing,
   with a toast on the duplicate branch.
8. **Resolve the row metadata defensively.** `new fileUri.FileUri(uri)` gives
   `.path` and `.name`; wrap `fs.lstatSync` in `try/catch` and default the size
   (HW-05-0058); wrap both UTD calls in `try/catch`, null-check the
   `TypeDescriptor` and check `mimeTypes.length` before indexing (HW-05-0057).
9. **Preview with a handled promise.** `filePreview.openPreview(context, {
   title, uri, mimeType })` returns `Promise<void>` - attach `.then()`/`.catch()`
   or await it in the already-`async` handler (HW-05-0059). An empty `mimeType`
   is a legitimate fallback: the reference says the system then infers the type
   from the URI suffix.
10. **Bottom sheet with `@CustomDialog`.** Pass the three picker functions in as
    props, call `this.controller?.close()` after each, and null the
    `CustomDialogController` in `aboutToDisappear` - the sample does this
    correctly.
11. **Make hint text stateful.** Anything bound into `bindPopup` must be `@State`
    if it is filled asynchronously (HW-05-0060), and the loading promise needs a
    `.catch()` (HW-05-0062).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The three pickers

`EmailAttachment.zip#entry/src/main/ets/pages/EmailAttachment.ets`

```ts
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { camera, cameraPicker } from '@kit.CameraKit';
import { picker } from '@kit.CoreFileKit';

@State fileList: Array<string> = [];

addToFileList: Function = (uri: string) => {
  if (!uri) {
    return;
  }
  if (!this.fileList.includes(uri)) {
    this.fileList.push(uri);
  } else {
    this.getUIContext()
      .getPromptAction()
      .showToast({ message: $r('app.string.toast'), duration: CommonConstant.TIP_DIALOG_DURATION });
  }
};

photoPick: Function = () => {
  let photoSelectOptions: photoAccessHelper.PhotoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  photoViewPicker.select(photoSelectOptions)
    .then((photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      this.addToFileList(photoSelectResult.photoUris[0]);
    })
    .catch((err: BusinessError) => {
      Logger.info(`photoPicker failed, code is ${err.code}, message is ${err.message}`);
    });
};

filePick: Function = () => {
  let documentSelectOptions = new picker.DocumentSelectOptions();
  let documentViewPicker = new picker.DocumentViewPicker();
  documentViewPicker.select(documentSelectOptions)
    .then((documentSelectResult) => {
      this.addToFileList(documentSelectResult[0]);
    })
    .catch((err: BusinessError) => {
      Logger.error(`filePicker failed, code is ${err.code}, message is ${err.message}`);
    });
};

cameraPick: Function = () => {
  let pickerProfile: cameraPicker.PickerProfile = {
    cameraPosition: camera.CameraPosition.CAMERA_POSITION_BACK
  };
  cameraPicker.pick(this.getUIContext().getHostContext(), [cameraPicker.PickerMediaType.PHOTO], pickerProfile)
    .then((pickerResult) => {
      this.addToFileList(pickerResult.resultUri.toString());
    })
    .catch((err: BusinessError) => {
      Logger.error(`the pick call failed. error code: ${err.code}`);
    });
};

cancel: Function = (uri: string) => {
  this.fileList = this.fileList.filter(item => item !== uri);
};
```

### The bottom-sheet dialog

`EmailAttachment.zip#entry/src/main/ets/pages/EmailAttachment.ets`

```ts
dialogController: CustomDialogController | null = new CustomDialogController({
  builder: PickerDialog({
    photoPick: this.photoPick,
    cameraPick: this.cameraPick,
    filePick: this.filePick
  }),
  autoCancel: true,
  customStyle: true,
  alignment: DialogAlignment.Bottom,
});

aboutToDisappear() {
  this.dialogController = null;
}
```

`EmailAttachment.zip#entry/src/main/ets/components/PickerDialog.ets`

```ts
@CustomDialog
export struct PickerDialog {
  controller?: CustomDialogController;
  photoPick: Function = () => {};
  cameraPick: Function = () => {};
  filePick: Function = () => {};

  build() {
    Column() {
      Text($r('app.string.hint'));

      Column() {
        Text($r('app.string.gallery')).itemStyle().onClick(() => {
          this.photoPick();
          this.controller?.close();
        });
        Text($r('app.string.camera')).itemStyle().onClick(() => {
          this.cameraPick();
          this.controller?.close();
        });
        Text($r('app.string.file')).itemStyle().onClick(() => {
          this.filePick();
          this.controller?.close();
        });
      };

      Column() {
        Text($r('app.string.cancel')).onClick(() => {
          this.controller?.close();
        });
      };
    };
  }
}
```

### The attachment row - as shipped

`EmailAttachment.zip#entry/src/main/ets/components/Attachment.ets`

```ts
import { fileUri, fileIo as fs } from '@kit.CoreFileKit';
import { filePreview } from '@kit.PreviewKit';
import { uniformTypeDescriptor as utd } from '@kit.ArkData';

@Component
export struct Attach {
  uri: string = '';
  filename: string = '';
  filesize: string = '';
  mimeType: string = '';
  cancel: Function = (uri: string) => {};

  aboutToAppear(): void {
    let fileUriObject = new fileUri.FileUri(this.uri);
    let size = fs.lstatSync(fileUriObject.path)?.size;              // HW-05-0058
    this.filesize = size > CommonConstant.MB_BOUNDARY ?
      (Math.round(size / CommonConstant.MB_DIVISOR) / CommonConstant.TWO_DECIMAL_PLACES) + CommonConstant.MB_TEXT
      : (size > CommonConstant.KB_BOUNDARY) ?
        (Math.round(size / CommonConstant.KB_DIVISOR) / CommonConstant.TWO_DECIMAL_PLACES) + CommonConstant.KB_TEXT :
        size + CommonConstant.BYTE_TEXT;
    this.filename = fileUriObject.name;

    let fileExtention = CommonConstant.FILE_SEPARATOR + this.filename.split(CommonConstant.FILE_SEPARATOR).pop();
    let typeId = utd.getUniformDataTypeByFilenameExtension(fileExtention);
    let typeObj = utd.getTypeDescriptor(typeId);
    this.mimeType = typeObj.mimeTypes[0];                            // HW-05-0057
  }

  // row onClick
  .onClick(async () => {
    let previewInfo: filePreview.PreviewInfo = {
      title: this.filename,
      uri: this.uri,
      mimeType: this.mimeType
    };
    filePreview.openPreview(this.getUIContext().getHostContext(), previewInfo);   // HW-05-0059
  });
}
```

Corrected `aboutToAppear` and preview:

```ts
aboutToAppear(): void {
  const fileUriObject = new fileUri.FileUri(this.uri);
  this.filename = fileUriObject.name;

  let size = 0;
  try {
    size = fs.lstatSync(fileUriObject.path).size;
  } catch (e) {
    Logger.error(`lstat failed for ${this.filename}: ${JSON.stringify(e)}`);
  }
  this.filesize = size > CommonConstant.MB_BOUNDARY
    ? (Math.round(size / CommonConstant.MB_DIVISOR) / CommonConstant.TWO_DECIMAL_PLACES) + CommonConstant.MB_TEXT
    : size > CommonConstant.KB_BOUNDARY
      ? (Math.round(size / CommonConstant.KB_DIVISOR) / CommonConstant.TWO_DECIMAL_PLACES) + CommonConstant.KB_TEXT
      : size + CommonConstant.BYTE_TEXT;

  this.mimeType = '';   // empty is valid: the system infers from the URI suffix
  try {
    const ext = CommonConstant.FILE_SEPARATOR + this.filename.split(CommonConstant.FILE_SEPARATOR).pop();
    const typeId = utd.getUniformDataTypeByFilenameExtension(ext);
    const typeObj = typeId ? utd.getTypeDescriptor(typeId) : null;
    if (typeObj && typeObj.mimeTypes.length > 0) {
      this.mimeType = typeObj.mimeTypes[0];
    }
  } catch (e) {
    Logger.error(`mime lookup failed for ${this.filename}: ${JSON.stringify(e)}`);
  }
}

.onClick(() => {
  const previewInfo: filePreview.PreviewInfo = {
    title: this.filename,
    uri: this.uri,
    mimeType: this.mimeType
  };
  filePreview.openPreview(this.getUIContext().getHostContext(), previewInfo)
    .catch((err: BusinessError) => {
      Logger.error(`Failed to open preview, err.code = ${err.code}, err.message = ${err.message}`);
    });
});
```

### Size thresholds

`EmailAttachment.zip#entry/src/main/ets/constant/CommonConstant.ets`

```ts
export class CommonConstant {
  static readonly TIP_DIALOG_DURATION: number = 2000;
  static readonly MB_BOUNDARY: number = 1000000;
  static readonly KB_BOUNDARY: number = 1000;
  static readonly MB_DIVISOR: number = 10000;
  static readonly KB_DIVISOR: number = 10;
  static readonly TWO_DECIMAL_PLACES: number = 100;
  static readonly MB_TEXT: string = 'MB';
  static readonly KB_TEXT: string = 'KB';
  static readonly BYTE_TEXT: string = 'B';
  static readonly FILE_SEPARATOR: string = '.';
}
```

The divisor/`TWO_DECIMAL_PLACES` pair is the two-decimal rounding trick:
`Math.round(size / 10000) / 100` yields megabytes to two decimals.

## Permissions & config

`EmailAttachment.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "extensionAbilities": [
      {
        "name": "EntryBackupAbility",
        "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup",
        "exported": false,
        "metadata": [
          { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
        ]
      }
    ]
    // no requestPermissions block
  }
}
```

Notes on the config:

- **No permission block at all**, and that is the correct outcome:
  `PhotoViewPicker`, `cameraPicker.pick` and `DocumentViewPicker` are system
  pickers that grant per-item access to what the user selected, and
  `filePreview.openPreview` consumes that same URI.
- Requesting `ohos.permission.READ_IMAGEVIDEO` or `ohos.permission.CAMERA` here
  would be an over-declaration - compare OFFICE-05 (HW-05-0022) and OFFICE-07
  (HW-05-0040) where exactly that happened.
- `deviceTypes` is `["phone", "tablet", "2in1"]`, which matches Preview Kit's
  supported device list.
- The document's project tree matches the shipped ZIP exactly, including
  `entrybackupability` - verified consistent.
- `build-profile.json5` pins `compatibleSdkVersion` / `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Preview Kit region.** "Currently, Preview Kit is available only in the
  Chinese mainland." The attachment list still works elsewhere; only the tap-to-
  preview does not.
- **Preview Kit devices.** Phones, tablets and 2-in-1 devices; the emulator does
  not preview `.pdf`, `.pptx`, `.xlsx` or `.docx`.
- **Supported preview types are fixed** by the Preview Kit table (images, video,
  audio, text, HTML, PDF and Office documents). An attachment outside that set
  cannot be opened, which is why a `canPreview` gate is worth adding.
- **One item per pick.** `maxSelectNumber = 1` for the gallery, and the file and
  camera paths both take element `0`; `addToFileList` accepts a single URI.
- **`getUniformDataTypeByFilenameExtension` and `getTypeDescriptor` are UTD
  APIs** documented for Phone, PC/2in1, Tablet and TV - one device class narrower
  than the window APIs used elsewhere in this industry.
- **The picker URIs are not all plain sandbox paths.** The three sources return
  different URI schemes, and `fileUri.FileUri(uri).path` is what the row feeds to
  `lstatSync`; treat a stat failure as expected rather than exceptional.
- **Nothing is sent.** The compose screen has no transport - the recipient,
  subject and body fields are inert and the send control is a label.

## Pitfalls

- **`typeObj.mimeTypes[0]` without a null check is incorrect.**
  `getTypeDescriptor` returns `null` when the type does not exist, and both
  official examples wrap the UTD calls in `try/catch`; an unusual extension from
  the document picker therefore breaks the attachment row before it renders.
  (HW-05-0057)
- **`fs.lstatSync(...)?.size` followed by unguarded arithmetic is incorrect.**
  The `?.` admits the value may be missing, but the very next expression compares
  and concatenates it, producing `undefinedB`; and `lstatSync` throws rather than
  returning undefined for a path it cannot reach. Wrap it and default to `0`.
  (HW-05-0058)
- **Calling `filePreview.openPreview(...)` as a bare statement is incorrect** -
  in the document snippet as well as the sample. It returns a `Promise<void>`, so
  a refused preview is an unhandled rejection and the tap appears to do nothing.
  (HW-05-0059)
- **Binding a plain `private addTip` into `bindPopup` is incorrect.** Only
  `@State` members trigger a rebuild, and the popup is opened from `onAppear`
  before the asynchronous `getStringValue` resolves, so the hint is always empty.
  (HW-05-0060)
- **`import { Context } from '@ohos.abilityAccessCtrl'` is the wrong home for
  that type.** The permission module's reference documents only
  `import { abilityAccessCtrl } from '@kit.AbilityKit';`; take the context type
  from `common` in `@kit.AbilityKit` as every sibling sample does. (HW-05-0061)
- **`resourceManager.getStringValue(...).then(...)` without `.catch()` is
  incorrect** - the same file attaches one to all three picker chains.
  (HW-05-0062)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/js-apis-data-uniformtypedescriptor.md` -
  `getUniformDataTypeByFilenameExtension11+`, `getTypeDescriptor` ("If the
  uniform data type does not exist, null is returned"), `TypeDescriptor.mimeTypes`
  and the guarded examples.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-uniformtypedescriptor
- `documentation/harmonyos-references/03_application-services/preview-arkts.md` -
  `openPreview` returning `Promise<void>`, its `.then()/.catch()` example, and the
  empty-`mimeType` inference rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/preview-arkts
- `documentation/harmonyos-guides/07_application-services/preview-introduction.md` -
  the Chinese-mainland region limit, supported devices, emulator restrictions and
  the supported extension/MIME table.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/preview-introduction
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  the `stat`/`lstat` family and the `13900012 Permission denied` throw.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` -
  `DocumentViewPicker.select` and `DocumentSelectOptions`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  the module's documented import surface, used to check the `Context` import.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
