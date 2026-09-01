---
id: COMMON-15
title: H5 image and file upload - route one HTML file input to the photo, camera or document picker
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/15_webview_picker.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/webview_picker-0000002257162014
sample: huawei_industry_tree/19_common_technical_solutions/downloads/WebviewPicker.zip
kits: ["@kit.ArkWeb", "@kit.MediaLibraryKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.ArkUI"]
apis: ["Web.onShowFileSelector", "FileSelectorResult.handleFileList", "WebviewController.runJavaScript", "webview.WebviewController.setWebDebuggingAccess", "Web.fileAccess", "Web.zoomAccess", "Web.javaScriptAccess", "Web.domStorageAccess", "Web.geolocationAccess", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.PhotoViewPicker", "PhotoViewPicker.select", "cameraPicker.pick", "cameraPicker.PickerProfile", "cameraPicker.PickerMediaType", "cameraPicker.PickerResult", "camera.CameraPosition", "picker.DocumentSelectOptions", "picker.DocumentViewPicker", "DocumentViewPicker.select", "$rawfile", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0034, HW-19-0035, HW-19-0036, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an embedded H5 page - a customer-service chat, a claim form,
a support ticket - offers **more than one way to attach something**: pick from
the gallery, take a photo now, or attach a document. One native handler has to
serve all three, and the page itself has to say which one it wants.

The document's framing: 客服页面发送文件、聊天页面发送照片、H5页面预览文件
("sending a file from a customer-service page, sending a photo from a chat page,
previewing a file from an H5 page").

The distinguishing technique is the **round trip**: the native handler asks the
page which picker to open by executing a JavaScript function in it, then launches
the corresponding system picker and hands the result back.

## Feature checklist

The application must:

- Host the chat/upload page as a local `$rawfile` and give each attachment
  affordance its own hidden `<input type="file">`.
- Record in the page which affordance the user touched (`setFileType('photo' |
  'camera' | 'document')`) and expose it through a `getFileType()` function.
- Intercept the file input with `Web.onShowFileSelector` and return `true`.
- Read the wanted type back with `WebviewController.runJavaScript('getFileType()')`.
- Launch `PhotoViewPicker.select` for `photo`, `cameraPicker.pick` for `camera`,
  and `DocumentViewPicker.select` for `document`.
- Hand the resulting URIs back with `FileSelectorResult.handleFileList`.
- **Resolve the pending input on every path**, including errors, cancellation and
  unknown types (HW-19-0035).
- Log every picker failure on the error channel (HW-19-0036).
- Keep web debugging out of the released build (HW-19-0034).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup, publishes the two insets, loads `pages/MainPage` |
| `entrybackupability/EntryBackupAbility.ets` | backup ExtensionAbility |
| `common/Constants.ets` | layout constants and `WEB_PAGE = 'WebPicker.html'` |
| `utils/Logger.ets` | thin `hilog` wrapper |
| `pages/MainPage.ets` | the title bar plus the `Web` component and the whole `onShowFileSelector` handler |
| `resources/rawfile/WebPicker.html` | the chat page: three labelled icons, each bound to its own hidden file input and each calling `setFileType(...)` on click |

**The type round trip is the design.** ArkWeb's `onShowFileSelector` reports
*that* a file input fired, not *which* one. The page therefore records the type
itself:

```html
<label class="photo-input-icon" for="photoInput" onclick="setFileType('photo')"></label>
<input type="file" id="photoInput" name="photoInput" style="display:none">
<label class="camera-input-icon" for="cameraInput" onclick="setFileType('camera')"></label>
<input type="file" id="cameraInput" name="cameraInput" style="display:none">
<label class="document-input-icon" for="documentInput" onclick="setFileType('document')"></label>
<input type="file" id="documentInput" name="documentInput" style="display:none">
```

and the native side reads it back with `runJavaScript('getFileType()')` before
choosing a picker.

**Control flow.**

1. The user taps one of the three icons; the label's `onclick` stores the type
   and then activates the associated hidden input.
2. ArkWeb fires `onShowFileSelector(event)`; the handler returns `true`, taking
   ownership of the selection.
3. `this.controller.runJavaScript('getFileType()', cb)` returns the stored type
   as a JSON string.
4. A `switch` launches the matching picker.
5. Each success handler stores the URI in `this.uris` and calls
   `event.result.handleFileList(this.uris)`, which completes the page's file
   input.

Note that steps 3-5 are asynchronous while step 2's `return true` is
synchronous - that gap is why every failure path has to resolve the request
explicitly (HW-19-0035).

## Implementation steps

1. **Ship the page as a rawfile** and give each attachment type its own hidden
   `<input type="file">` plus a `setFileType(...)` / `getFileType()` pair in
   page JavaScript.
2. **Enable JavaScript on the web view** (`javaScriptAccess(true)`) - the round
   trip depends on it - and lock down the rest: `.fileAccess(false)`,
   `.geolocationAccess(false)`, `.zoomAccess(false)`.
3. **Intercept and take ownership**: `onShowFileSelector((event) => { ...; return
   true; })`. The reference: "If it returns **true**, the application can
   customize the response behavior for **Select file**."
4. **Ask the page for the type** with `runJavaScript('getFileType()', (error,
   result) => ...)`, checking `error` first and `JSON.parse`-ing `result`.
5. **Photo branch**: `PhotoSelectOptions` with `MIMEType =
   PhotoViewMIMETypes.IMAGE_TYPE` and `maxSelectNumber = 1`, then
   `new photoAccessHelper.PhotoViewPicker().select(options)`; hand
   `photoSelectResult.photoUris` to `handleFileList`.
6. **Camera branch**: `cameraPicker.pick(context, [PickerMediaType.PHOTO],
   { cameraPosition: camera.CameraPosition.CAMERA_POSITION_BACK })` inside
   `try/catch`; hand `[pickerResult.resultUri]` to `handleFileList`.
7. **Document branch**: `DocumentSelectOptions` with `maxSelectNumber = 1`, then
   `new picker.DocumentViewPicker().select(options)`; hand
   `[documentSelectResult[0]]` to `handleFileList`.
8. **Resolve every other path** with `handleFileList([])`: the `runJavaScript`
   error, a falsy result, the `default` type, and each picker rejection
   (HW-19-0035).
9. **Log failures with `Logger.error`** (HW-19-0036), and drop
   `setWebDebuggingAccess(true)` from the release build (HW-19-0034).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The whole handler (as shipped)

`WebviewPicker.zip#WebviewPicker/entry/src/main/ets/pages/MainPage.ets`

```ts
import { webview } from '@kit.ArkWeb';
import { BusinessError } from '@kit.BasicServicesKit';
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { camera, cameraPicker } from '@kit.CameraKit';
import Logger from '../utils/Logger';
import { picker } from '@kit.CoreFileKit';
import { common } from '@kit.AbilityKit';
import { Constants } from '../common/Constants';

@Entry
@Component
struct WebComponent {
  controller: webview.WebviewController = new webview.WebviewController();
  context = this.getUIContext().getHostContext() as common.Context;
  //定义一个全局的string类型的数组，用来存放通过picker拉起后选择完图片后图片的uri
  @State uris: Array<string> = [];
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageProp('topRectHeight') topRectHeight: number = 0;

  aboutToAppear() {
    webview.WebviewController.setWebDebuggingAccess(true); // FIX (HW-19-0034)
  }

  build() {
    Column() {
      // ... title row ...
      Web({ src: $rawfile(Constants.WEB_PAGE), controller: this.controller })
        .height(Constants.CONTENT_HEIGHT)
        .width(Constants.FULL)
        .zoomAccess(false)
        .javaScriptAccess(true)
        .domStorageAccess(true)
        .fileAccess(false)
        .geolocationAccess(false)
        .onShowFileSelector((event) => {
          this.controller.runJavaScript('getFileType()', async (error, result) => {
            if (error) {
              Logger.error(`run JavaScript error, ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
              return;   // FIX (HW-19-0035): event?.result.handleFileList([]) before returning
            }
            if (result) {
              let type = JSON.parse(result) as string;
              switch (type) {
                case 'photo':
                  let photoSelectOptions: photoAccessHelper.PhotoSelectOptions =
                    new photoAccessHelper.PhotoSelectOptions();
                  // 过滤选择媒体文件类型为IMAGE_TYPE
                  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
                  // 选择媒体文件的最大数目
                  photoSelectOptions.maxSelectNumber = 1;
                  let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
                  photoViewPicker.select(photoSelectOptions)
                    .then((photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
                      this.uris = photoSelectResult.photoUris;
                      if (event) {
                        event.result.handleFileList(this.uris);
                      }
                    })
                    .catch((err: BusinessError) => {
                      // FIX (HW-19-0036): Logger.error; FIX (HW-19-0035): resolve with []
                      Logger.info(`Invoke photoViewPicker.select failed, code is ${err.code}, message is ${err.message}`);
                    });
                  break;
                case 'camera':
                  try {
                    let pickerProfile: cameraPicker.PickerProfile = {
                      cameraPosition: camera.CameraPosition.CAMERA_POSITION_BACK
                    };
                    let pickerResult: cameraPicker.PickerResult = await cameraPicker.pick(this.context,
                      [cameraPicker.PickerMediaType.PHOTO], pickerProfile);
                    this.uris[0] = pickerResult.resultUri.toString();
                    if (event) {
                      event.result.handleFileList(this.uris);
                    }
                  } catch (error) {
                    let err = error as BusinessError;
                    Logger.error('the pick call failed. error code: ' + err.code);
                  }
                  break;
                case 'document':
                  let documentSelectOptions = new picker.DocumentSelectOptions();
                  documentSelectOptions.maxSelectNumber = 1;
                  let documentViewPicker = new picker.DocumentViewPicker();
                  documentViewPicker.select(documentSelectOptions).then((documentSelectResult) => {
                    this.uris[0] = documentSelectResult[0];
                    if (event) {
                      event.result.handleFileList(this.uris);
                    }
                  }).catch((err: BusinessError) => {
                    Logger.error('Invoke documentViewPicker.select failed, code is ' + err.code);
                  });
                  break;
                default:
                  Logger.error('No Such Type');   // FIX (HW-19-0035): resolve with []
                  break;
              }
            }
          });
          return true;
        });
    }
    .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
    .width(Constants.FULL)
    .height(Constants.FULL);
  }
}
```

### The corrected completion discipline

```ts
.onShowFileSelector((event) => {
  const finish = (files: string[]) => { event?.result.handleFileList(files); };
  this.controller.runJavaScript('getFileType()', (error, result) => {
    if (error || !result) {
      Logger.error(`getFileType failed: ${(error as BusinessError)?.code}`);
      finish([]);
      return;
    }
    const type = JSON.parse(result) as string;
    switch (type) {
      // each branch calls finish(uris) on success and finish([]) on failure
      default:
        Logger.error('No Such Type');
        finish([]);
    }
  });
  return true;
})
```

### The page side of the round trip

`WebviewPicker.zip#WebviewPicker/entry/src/main/resources/rawfile/WebPicker.html`

```html
<div class="file-input-container">
    <label class="photo-input-icon" for="photoInput" onclick="setFileType('photo')"></label>
    <input type="file" id="photoInput" name="photoInput" style="display:none">
    <label class="camera-input-icon" for="cameraInput" onclick="setFileType('camera')"></label>
    <input type="file" id="cameraInput" name="cameraInput" style="display:none">
    <label class="document-input-icon" for="documentInput" onclick="setFileType('document')"></label>
    <input type="file" id="documentInput" name="documentInput" style="display:none">
</div>
```

### Page constant

`WebviewPicker.zip#WebviewPicker/entry/src/main/ets/common/Constants.ets`

```ts
export class Constants {
  static readonly FULL = '100%';
  static readonly TITLE_HEIGHT = '12%';
  static readonly CONTENT_HEIGHT = '88%';
  static readonly TITLE_COLOR = '#F1F3F5';
  static readonly WEB_PAGE = 'WebPicker.html';
}
```

## Permissions & config

**No permissions are required, and none are declared.** All three entry points
are system pickers - `PhotoViewPicker`, `cameraPicker.pick` and
`DocumentViewPicker` - so the user grants access to exactly the item they choose
and the application needs neither a media permission nor
`ohos.permission.CAMERA`. The page is a local `$rawfile`, so no
`ohos.permission.INTERNET` either.

`WebviewPicker.zip#WebviewPicker/entry/src/main/module.json5`:

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
- **`javaScriptAccess(true)` is mandatory here.** The whole type-selection
  mechanism is a `runJavaScript` call into the page; with JavaScript disabled the
  handler cannot tell the three inputs apart.
- **`onShowFileSelector` ownership is all-or-nothing.** "If this function is not
  called or returns **false**, the **Web** component provides the default
  **Select file** UI. If it returns **true**, the application can customize the
  response behavior." There is no partial handover.
- **`cameraPicker.PickerResult.resultUri`** is "a public media path" when
  `saveUri` is empty; if `saveUri` is set but the application lacks write
  permission on it, "`resultUri` cannot be obtained".
- **`cameraPicker.pick` devices**: Phone, PC/2in1, Tablet, TV, Wearable.
- **`fileAccess(false)` does not block `$rawfile`**, so the local page and its
  assets load with file-system access disabled.
- **`setWebDebuggingAccess` defaults to `false`** and should stay that way in
  release builds.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`webview.WebviewController.setWebDebuggingAccess(true)` in `aboutToAppear` is
  incorrect for a released app.** "Enabling web debugging allows users to check
  and modify the internal status of the web page, which poses security risks."
  (HW-19-0034)
- **Returning `true` and then abandoning the request on failure is incorrect.**
  Five paths - `runJavaScript` error, falsy result, `default` type, photo
  rejection, document rejection - leave the HTML file input pending forever.
  Call `handleFileList([])` on each. (HW-19-0035)
- **Logging the photo picker rejection with `Logger.info` is incorrect** where the
  camera and document branches use `Logger.error`; it buries the most common
  failure among this file's routine INFO trace. (HW-19-0036)
- **Cancellation arrives as a rejection, not as an empty result.** Both
  `PhotoViewPicker.select` and `DocumentViewPicker.select` reject when the user
  backs out, so the cancel path is the `.catch` - which is exactly where
  `handleFileList([])` belongs.
- **The `switch` cases share one block scope.** `let photoSelectOptions`,
  `let photoViewPicker`, `let documentSelectOptions` and `let documentViewPicker`
  are all declared directly inside the same `switch` body; wrap each case in
  braces if you add more declarations.
- **`this.uris[0] = ...` in the camera and document branches keeps whatever the
  previous selection left in the array.** The photo branch replaces the array
  wholesale. Build a fresh list per selection rather than patching index 0.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-events.md` -
  `onShowFileSelector`, the `true`/`false` return contract, and the official
  document-picker and photo-picker examples.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-events#onshowfileselector9
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-fileselectorresult.md` -
  `handleFileList(fileList: Array<string>)`, "Instructs the Web component to
  select a file."
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-fileselectorresult
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `runJavaScript` and `setWebDebuggingAccess` (default `false`, security note).
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/04_media/js-apis-camerapicker.md` -
  `cameraPicker.pick`, `PickerProfile`, `PickerMediaType`, and the `resultUri`
  semantics.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-camerapicker
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker -
  `PhotoViewPicker.select`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-file-picker#documentviewpicker -
  `DocumentViewPicker.select` and `DocumentSelectOptions`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/webview_picker-0000002257162014
