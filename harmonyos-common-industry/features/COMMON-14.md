---
id: COMMON-14
title: Picture upload and preview - drive PhotoViewPicker from an HTML file input inside a Web component
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/14_upload_picture.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
sample: huawei_industry_tree/19_common_technical_solutions/downloads/UploadPicture.zip
kits: ["@kit.ArkWeb", "@kit.MediaLibraryKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: ["Web.onShowFileSelector", "FileSelectorResult.handleFileList", "webview.WebviewController", "webview.WebviewController.setWebDebuggingAccess", "Web.fileAccess", "Web.javaScriptAccess", "Web.domStorageAccess", "Web.geolocationAccess", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.PhotoViewMIMETypes", "photoAccessHelper.PhotoViewPicker", "PhotoViewPicker.select", "photoAccessHelper.PhotoSelectResult", "@Provide", "@Consume", "$rawfile", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0030, HW-19-0031, HW-19-0032, HW-19-0033, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an **HTML form inside a `Web` component has an
`<input type="file">`** and tapping it must open the system photo picker rather
than a browser file dialog - the classic "upload the front and back of your ID
card" flow - and the chosen image must also be previewed in the native UI around
the web view.

The key property is that **no permission is needed**: `PhotoViewPicker` is a
system picker, so the user grants access to exactly the picture they choose,
without the application holding a media permission.

## Feature checklist

The application must:

- Host the upload form as a local `$rawfile` HTML page inside a `Web` component.
- Intercept the form's file input with `Web.onShowFileSelector` and return `true`
  to say the application handled it.
- Configure `PhotoSelectOptions` - MIME type restricted to images, and a maximum
  selection count matching what the form expects.
- Launch `PhotoViewPicker.select`.
- Hand **the URIs the picker returned** back to the web page with
  `FileSelectorResult.handleFileList` (HW-19-0031).
- Keep a separate native-side array of picked URIs to render the preview
  `Image`s (HW-19-0032).
- Handle the picker's rejection - cancellation included - and log it at error
  level (HW-19-0033).
- Lock the web view down: no file-system access, no geolocation, and **no web
  debugging in a released build** (HW-19-0030).

## Architecture

Single-module project (`entry` HAP), one page file containing two components:

| Component | Responsibility |
| --- | --- |
| `WebComponent` (`@Entry`) | the whole screen: title row, two preview `Image`s, two `AddPicture` instances positioned over them, and the submit button; owns the shared `uris` array via `@Provide` |
| `AddPicture` | one `Web` component loading one rawfile HTML page, plus the `onShowFileSelector` handler; reads the shared array via `@Consume` |

Supporting files: `entryability/EntryAbility.ets`,
`entrybackupability/EntryBackupAbility.ets`, and the two rawfile pages
`UploadIDCardFront.html` / `UploadIDCardBack.html`.

**Layout trick.** The native `Image` renders the preview and the `Web` component
is laid over it with `.position(...)` and a `zIndex` - the web page supplies only
the transparent "add picture" affordance and the `<input type="file">` that
triggers the selector, while the picked image is drawn natively underneath. That
is why `Web` is given `.backgroundColor(Color.Transparent)`.

**Control flow.**

1. The user taps the transparent web overlay; the HTML file input fires.
2. ArkWeb calls `onShowFileSelector(event)`. Returning `true` tells the component
   the application will supply the files.
3. The handler builds `PhotoSelectOptions` (`IMAGE_TYPE`, `maxSelectNumber = 1`),
   constructs a `PhotoViewPicker` and calls `select`.
4. The system picker appears; on resolution the handler must give the selected
   URIs to `event.result.handleFileList(...)` **and** store them in the shared
   array so the native `Image` updates.
5. `@Provide`/`@Consume` propagates the array back to the parent, whose
   `Image(this.uris[n])` re-renders with the picked photo.

The `isFront: boolean` constructor parameter is what tells the child which slot
of the shared array it owns - the two `AddPicture` instances are otherwise
identical.

## Implementation steps

1. **Ship the form as a rawfile.** One HTML page per upload slot under
   `entry/src/main/resources/rawfile/`, loaded with
   `Web({ src: $rawfile('UploadIDCardFront.html'), controller })`.
2. **Share the picked URIs.** `@Provide uris: Array<string>` in the page,
   `@Consume uris: Array<string>` in the child - the *same* element type on both
   sides (HW-19-0032). Keep the placeholder artwork out of this array.
3. **Configure the web view.** `.backgroundColor(Color.Transparent)`,
   `.fileAccess(false)`, `.javaScriptAccess(true)`, `.domStorageAccess(true)`,
   `.geolocationAccess(false)`. Do **not** call `setWebDebuggingAccess(true)`
   outside a debug build (HW-19-0030).
4. **Intercept the file input**:
   ```ts
   .onShowFileSelector((event) => {
     // ... launch the picker ...
     return true;   // the application handles the selection
   })
   ```
5. **Build the picker options**: `MIMEType =
   photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE`, `maxSelectNumber = 1` for a
   single-file input.
6. **Select, then hand back the result.** In the resolution handler, check the
   result, call `event.result.handleFileList(photoSelectResult.photoUris)`, and
   write the same URI into the shared array (HW-19-0031).
7. **Handle rejection.** `select` rejects when the user cancels or the picker
   fails; log with `hilog.error` and matching format identifiers (HW-19-0033).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The page: shared array, native preview, web overlay

`UploadPicture.zip#UploadPicture/entry/src/main/ets/pages/UploadPicture.ets`

```ts
import { webview } from '@kit.ArkWeb';
import { BusinessError } from '@kit.BasicServicesKit';
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

@Entry
@Component
struct WebComponent {
  context = this.getUIContext().getHostContext() as Context;
  controller: webview.WebviewController = new webview.WebviewController();
  //定义一个全局的string类型的数组，用来存放通过picker拉起后选择完图片后图片的uri
  @Provide uris: Array<string | Resource> = [$r('app.media.front'), $r('app.media.back')];
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;

  aboutToAppear() {
    webview.WebviewController.setWebDebuggingAccess(true); // FIX (HW-19-0030): remove from release builds
  }

  build() {
    Column() {
      // ... title row ...
      Stack() {
        Image(this.uris[0])
          .autoResize(true)
          .imageStyle(this.context)
          .objectFit(ImageFit.Contain);
      }
      .height($r('app.string.stack_height'));

      AddPicture({
        src: $rawfile('UploadIDCardFront.html'),
        isFront: true
      })
        .position({ x: $r('app.string.web_position_x'), y: $r('app.string.web_position_y_front') })
        .zIndex(this.context.resourceManager.getNumber($r('app.float.web_zindex').id));

      Stack() {
        Image(this.uris[1])
          .autoResize(true)
          .imageStyle(this.context)
          .objectFit(ImageFit.Contain);
      }
      .height($r('app.string.stack_height'));

      AddPicture({
        src: $rawfile('UploadIDCardBack.html'),
        isFront: false
      })
        .position({ x: $r('app.string.web_position_x'), y: $r('app.string.web_position_y_back') })
        .zIndex(this.context.resourceManager.getNumber($r('app.float.web_zindex').id));

      Button($r('app.string.button'));
    }
    .padding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight)
    });
  }
}
```

### The file-selector interception (as shipped - see HW-19-0031)

`UploadPicture.zip#UploadPicture/entry/src/main/ets/pages/UploadPicture.ets`

```ts
@Component
struct AddPicture {
  @State src: ResourceStr = '';
  isFront: boolean = true;
  controller: webview.WebviewController = new webview.WebviewController();
  @Consume uris: Array<string>;   // FIX (HW-19-0032): must match the @Provide type

  build() {
    Column() {
      Web({ src: this.src, controller: this.controller })
        .backgroundColor(Color.Transparent)
        .size({ width: $r('app.string.web_width'), height: $r('app.string.web_height') })
        .fileAccess(false)
        .javaScriptAccess(true)
        .domStorageAccess(true)
        .geolocationAccess(false)
        .onShowFileSelector((event) => {
          hilog.info(0xFF00, 'testTag', 'MyFileUploader onShowFileSelector invoked');
          let photoSelectOptions: photoAccessHelper.PhotoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
          photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
          photoSelectOptions.maxSelectNumber = 1;
          let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
          photoViewPicker.select(photoSelectOptions).then((photoSelectResult) => {
            if (event) {
              event.result.handleFileList(this.uris);   // FIX (HW-19-0031): pass photoSelectResult.photoUris
            }
            if (photoSelectResult !== null) {
              if (this.isFront) {
                this.uris[0] = photoSelectResult.photoUris[0];
              } else {
                this.uris[1] = photoSelectResult.photoUris[0];
              }
            }
          }).catch((err: BusinessError) => {
            hilog.isLoggable(0xFF00, 'testTag', hilog.LogLevel.ERROR);   // FIX (HW-19-0033): no-op
            hilog.info(0xFF00, 'testTag', 'Invoke photoViewPicker.select failed, code is %{public}s' +
              ', message is %{public}s', err.code, err.message);
          });
          return true;
        });
    }
    .height($r('app.float.web_column_height'));
  }
}
```

Corrected handler, following the official `onShowFileSelector` example:

```ts
.onShowFileSelector((event) => {
  const options = new photoAccessHelper.PhotoSelectOptions();
  options.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  options.maxSelectNumber = 1;
  new photoAccessHelper.PhotoViewPicker().select(options)
    .then((chooseFile: photoAccessHelper.PhotoSelectResult) => {
      if (!chooseFile || chooseFile.photoUris.length === 0) {
        return;
      }
      event?.result.handleFileList(chooseFile.photoUris);
      this.uris[this.isFront ? 0 : 1] = chooseFile.photoUris[0];
    })
    .catch((err: BusinessError) => {
      hilog.error(0xFF00, 'UploadPicture',
        'photoViewPicker.select failed, code %{public}d, message %{public}s', err.code, err.message);
    });
  return true;
})
```

## Permissions & config

**No permissions are required, and none are declared.** `PhotoViewPicker` is a
system picker: the user picks the image in a system UI and the application
receives a URI for that one file, so no media-library permission is needed. The
web pages are local `$rawfile`s, so `ohos.permission.INTERNET` is not needed
either - and correctly, unlike the sibling sample in COMMON-09, this project does
not declare it.

`UploadPicture.zip#UploadPicture/entry/src/main/module.json5`:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
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
  }
}
```

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`onShowFileSelector` must return `true`** for the application to own the
  selection; returning `false` (or nothing) hands the input back to the default
  behaviour.
- **`handleFileList` takes file URIs**, and its purpose is to complete the web
  page's pending file input. It is not a general-purpose channel for application
  state.
- **`maxSelectNumber` should match the form.** The sample sets 1 because each
  HTML page has a single-file input.
- **`setWebDebuggingAccess` defaults to `false`** and the reference advises
  against enabling it in released applications.
- **`fileAccess(false)` does not block `$rawfile`.** The reference states the
  setting "does not affect the access to the files specified through `$rawfile`",
  so the local pages and their relative assets still load.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`webview.WebviewController.setWebDebuggingAccess(true)` in `aboutToAppear` is
  incorrect for a released app.** The reference: "Enabling web debugging allows
  users to check and modify the internal status of the web page, which poses
  security risks. Therefore, you are advised not to enable this feature in the
  officially released version of the application." Remove it or gate it on build
  mode. (HW-19-0030)
- **`event.result.handleFileList(this.uris)` is incorrect.** It runs before the
  new selection is stored, so the web form receives the previous values - on the
  first use, two `$r('app.media.*')` Resource objects rather than file URIs. The
  official example passes `chooseFile.photoUris`. (HW-19-0031)
- **`@Provide uris: Array<string | Resource>` with `@Consume uris:
  Array<string>` is incorrect.** Both decorators bind one object; the narrower
  consumer type hides the Resource placeholders from the type checker. Use the
  same type on both sides and keep placeholders out of the URI array.
  (HW-19-0032)
- **`hilog.isLoggable(...)` as a bare statement does nothing, and the failure is
  logged at info level, which is incorrect.** Log the rejection with
  `hilog.error` and format identifiers matching the argument types.
  (HW-19-0033)
- **`photoSelectResult !== null` is not the cancellation check.** `select`
  *rejects* when the user cancels, so cancellation lands in `.catch`; the null
  test in the resolution handler never fires. Guard on
  `photoUris.length` instead.
- **The `Web` component is an overlay, not the preview.** The picked image is
  rendered by the native `Image` beneath it; if you drop the `zIndex` /
  `position` layering the transparent web view stops receiving the taps.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-events.md` -
  `onShowFileSelector`, `FileSelectorResult.handleFileList`, and the two official
  examples (document picker and photo picker) that pass the freshly selected URIs.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-events#onshowfileselector9
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `setWebDebuggingAccess`, its default of `false` and the security note.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  `fileAccess`, `javaScriptAccess`, `domStorageAccess`, `geolocationAccess`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker -
  `PhotoViewPicker.select` and `PhotoSelectResult.photoUris`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-class#photoselectoptions -
  `PhotoSelectOptions.MIMEType` and `maxSelectNumber`.
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - the format
  identifier / argument mapping rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
