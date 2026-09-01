---
id: OFFICE-13
title: Open a file with another app - startAbility with viewData, a file URI and the default picker dialog
industry: 05_office
doc: huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
sample: huawei_industry_tree/05_office/downloads/demo_StartAbility.zip
kits: ["@kit.AbilityKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIAbilityContext.startAbility", Want, "wantConstant.Flags.FLAG_AUTH_READ_URI_PERMISSION", "wantConstant.Flags.FLAG_AUTH_WRITE_URI_PERMISSION", "ohos.want.action.viewData", "ohos.ability.params.showDefaultPicker", "fileUri.getUriFromPath", "fs.openSync", "fs.writeSync", "fs.closeSync", "fs.OpenMode", ForEach, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "window.off('avoidAreaChange')", "AppStorage.setOrCreate", "AppStorage.get", "UIContext.getHostContext", "UIContext.px2vp", "@StorageProp", skills, uris, linkFeature]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0076, HW-05-0077, HW-05-0078, HW-05-0079, HW-05-0080, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app holds a file it **cannot render itself** and
should hand it to whatever installed application can - the "open with" flow of a
file manager or an attachment list.

This is the alternative to Preview Kit (OFFICE-08, OFFICE-10). Preview Kit shows
the file inside a system previewer; this scenario hands the file to a **third-party
application** through an implicit Want and lets the user pick which one.

Three pieces have to line up:

| Side | What it does |
| --- | --- |
| Caller | builds a Want with `action: 'ohos.want.action.viewData'`, the file `uri`, its `type`, and URI-permission flags |
| System | matches installed applications against that Want and - with `showDefaultPicker` - shows the chooser |
| Target | declares a `skills` entry with the `viewData` action and a `uris` block carrying `scheme`, `type` and `linkFeature: "FileOpen"`, then reads `want.uri` in `onCreate` |

The shipped sample implements the **caller side only** (HW-05-0077).

No permission is required: authorization travels with the Want as URI flags.

## Feature checklist

The caller application must be able to:

- Hold the file inside its own sandbox and turn the sandbox path into a
  `file://` URI.
- Build a Want with the fixed `ohos.want.action.viewData` action.
- Declare the file's UTD type so the system matches the right applications.
- Grant the target read and, where needed, write access to the URI through
  `flags`.
- Force the "open with" chooser rather than silently deferring to a default
  application.
- Report a failure when no application matches.

A target application additionally has to declare the matching `skills` and read
the URI out of the incoming Want.

## Architecture

Single `entry` HAP, four source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | immersive layout, publishes `topRectHeight` / `bottomRectHeight` **and the `UIContext`** into `AppStorage`, loads `pages/Index` |
| `pages/Index.ets` | `@Entry`; the two "today"/"yesterday" file lists, each row calling the helper |
| `common/Utils.ets` | `startAbility()` - writes the sandbox file, builds the Want, calls `startAbility` |
| `common/Objects.ets` | the `File` row type (`image`, `description`, `time`) |
| `common/Constants.ets` | the hilog domain |

One structural detail is worth noting: `EntryAbility` stores the whole
`UIContext` in `AppStorage` under `uiContext`, and `Utils.startAbility` - a plain
static method with no component to call `this.getUIContext()` on - reads it back
to reach the `UIAbilityContext`:

```ts
let uiContext: UIContext = AppStorage.get('uiContext') as UIContext;
let uiAbilityContext = uiContext.getHostContext() as common.UIAbilityContext;
```

Caller flow:

```
row onClick -> Utils.startAbility()
  filesDir + '/1.txt'
  fs.openSync(READ_WRITE | CREATE) -> writeSync('1') -> [finally] closeSync
  fileUri.getUriFromPath(filepath)                  -> file:// URI
  uiAbilityContext.startAbility({
    action: 'ohos.want.action.viewData',
    parameters: { 'ohos.ability.params.showDefaultPicker': true },
    uri, type: 'general.plain-text',
    flags: FLAG_AUTH_WRITE_URI_PERMISSION | FLAG_AUTH_READ_URI_PERMISSION
  }).then(...).catch(...)
```

Target flow (documented, not shipped):

```
module.json5 skills: actions += 'ohos.want.action.viewData'
                     uris: [{ scheme: 'file', type: 'general.plain-text', linkFeature: 'FileOpen' }]
EntryAbility.onCreate(want, launchParam) -> want.uri -> fs.openSync(uri, READ_WRITE)
```

## Implementation steps

1. **Declare no permission.** The file lives in the caller's own sandbox and the
   target's access is granted per-URI through the Want flags. The sample's
   `module.json5` has no `requestPermissions` block, and the document has no
   权限说明 section - consistent.
2. **Put the file in the sandbox and take its URI.** `fs.openSync(path,
   READ_WRITE | CREATE)`, write, close in a `finally`, then
   `fileUri.getUriFromPath(path)`. The URI format the guide specifies is
   `file://bundleName/path`.
3. **Build the Want with the fixed action.** `action` must be
   `'ohos.want.action.viewData'` for file opening - the guide lists it as a fixed
   value.
4. **Declare the type.** Prefer a UTD such as `'general.plain-text'` or
   `'general.image'`; a MIME type also works. The guide warns that `type` must
   match the file behind the URI or nothing will be matched, and that `*/*` is
   not supported. Derive it from the file rather than hard-coding it
   (HW-05-0079).
5. **Force the chooser.** Add `parameters: { 'ohos.ability.params.showDefaultPicker': true }`
   - without it the default is `false` and the system may go straight to the
   default application (HW-05-0076).
6. **Grant URI access.** `flags: FLAG_AUTH_WRITE_URI_PERMISSION |
   FLAG_AUTH_READ_URI_PERMISSION`; drop the write flag when the target only
   needs to read.
7. **Handle the promise.** `startAbility` returns a promise; log the rejection so
   "no matching application" is visible. The sample does this correctly.
8. **On the target side**, declare the `skills` entry and read `want.uri` in
   `onCreate` - see the fragments below, which come from the document rather than
   the ZIP.
9. **Release the window listener.** `off('avoidAreaChange')` in
   `onWindowStageDestroy` (HW-05-0080).

## Verified snippets

Snippets in this section come from the sample project unless explicitly marked
as document-only.

### Caller: sandbox file, URI and the Want

`demo_StartAbility.zip#entry/src/main/ets/common/Utils.ets`

```ts
import { common, wantConstant } from '@kit.AbilityKit';
import { Constants } from './Constants';
import fs from '@ohos.file.fs';
import { fileUri } from '@kit.CoreFileKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

export class Utils {
  static startAbility() {
    let uiContext: UIContext = AppStorage.get('uiContext') as UIContext;
    let uiAbilityContext = uiContext.getHostContext() as common.UIAbilityContext;
    let filepath = uiAbilityContext.filesDir + '/1.txt';
    let file = fs.openSync(filepath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    try {
      fs.writeSync(file.fd, '1');
      let uri = fileUri.getUriFromPath(filepath);
      // 调用接口启动
      uiAbilityContext.startAbility({
        action: 'ohos.want.action.viewData', // 表示查看数据的操作，文件打开场景固定为此值
        parameters: {
          'ohos.ability.params.showDefaultPicker': true
        },
        uri: uri,
        type: 'general.plain-text', // 表示待打开文件的类型
        // 配置被分享文件的读写权限，例如对文件打开应用进行读写授权
        flags: wantConstant.Flags.FLAG_AUTH_WRITE_URI_PERMISSION | wantConstant.Flags.FLAG_AUTH_READ_URI_PERMISSION
      })
        .then(() => {
          hilog.info(Constants.DOMAIN, 'testTag', 'Succeed to invoke startAbility.');
        })
        .catch((err: BusinessError) => {
          hilog.error(Constants.DOMAIN, 'testTag',
            `Failed to invoke startAbility, code: ${err.code}, message: ${err.message}`);
        });
    } catch (e) {
      hilog.error(Constants.DOMAIN, 'testTag', `Failed, error: ${e}`);
    } finally {
      fs.closeSync(file.fd);
    }
  }
}
```

This is the part of the sample worth copying verbatim - it is the only helper in
this industry that gets `showDefaultPicker`, both URI flags and a handled
`startAbility` promise right in one place. Two things to change when adapting it:
take the path and `type` from the row (HW-05-0079), and move the `openSync` call
inside the `try` so its own failure is caught.

### Caller: the file list

`demo_StartAbility.zip#entry/src/main/ets/pages/Index.ets`

```ts
@State fileListYestoday: File[] = [
  { image: $r('app.media.txt'), description: '今日的琐事记录.txt', time: '2025年4月14日' },
  { image: $r('app.media.pdf'), description: '述职报告.pdf', time: '2025年4月14日' },
  { image: $r('app.media.image'), description: '家乡图片.jpg', time: '2025年4月14日' },
];

ForEach(this.fileListYestoday, (file: File) => {
  Row() {
    Image(file.image).objectFit(ImageFit.Contain).height(40).width('20%');
    Column() {
      Text(file.description).fontSize(15).height(25).width('80%');
      Text(file.time).fontSize(12).fontColor('#777777').height(25).width('80%');
    };
  }
  .onClick(() => {
    Utils.startAbility();          // ignores which row was tapped - HW-05-0079
  });
}, (index: number) => JSON.stringify(index));   // first parameter is the item - HW-05-0078
```

Corrected key generator and row handler:

```ts
ForEach(this.fileListYestoday, (file: File) => {
  Row() { /* ... */ }
    .onClick(() => {
      Utils.openFile(file);        // path and type derived from the row
    });
}, (file: File) => file.description);
```

### Caller: publishing the UIContext for non-component code

`demo_StartAbility.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
windowStage.loadContent('pages/Index', (err) => {
  // 全局获取uiContext
  let uiContext: UIContext = windowStage.getMainWindowSync().getUIContext();
  AppStorage.setOrCreate('uiContext', uiContext);
  if (err.code) {
    hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
    return;
  }
});
```

Note the ordering defect worth avoiding: the `UIContext` is stored **before** the
`err.code` check, so it is published even on a failed content load. Move the
store after the guard.

Immersive layout and the avoid-area listener that is never released:

```ts
let windowClass: window.Window = windowStage.getMainWindowSync();
windowClass.setWindowLayoutFullScreen(true).then(() => { /* ... */ })
  .catch((err: BusinessError) => { /* ... */ });

let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
let avoidArea = windowClass.getWindowAvoidArea(type);
AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

type = window.AvoidAreaType.TYPE_SYSTEM;
avoidArea = windowClass.getWindowAvoidArea(type);
AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

windowClass.on('avoidAreaChange', (data) => {      // no matching off - HW-05-0080
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});
```

This sample reads the top inset from `TYPE_SYSTEM`, which is the correct choice
for a status-bar inset - compare OFFICE-09 (HW-05-0053), where `TYPE_CUTOUT` was
used instead.

### Target side - from the document, not shipped in the ZIP

`module.json5` of the **target** application:

```json5
"skills": [{
  "entities": [
    "entity.system.home"
  ],
  "actions": [
    "ohos.want.action.viewData",
    "action.system.home"
  ],
  // 必填，声明数据处理能力
  "uris": [
    {
      // 允许打开uri中以file:// 协议开头标识的本地文件
      "scheme": "file",                 // 必填，声明协议类型为文件
      "type": "general.plain-text",     // 必填，表示支持打开的文件类型
      "linkFeature": "FileOpen"         // 必填且大小写敏感，表示此URI的功能为文件打开
    }
  ]
}]
```

Receiving the file in the target ability:

```ts
export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam) {
    // 从want信息中获取uri字段
    let uri = want.uri;
    // 根据待打开文件的URI进行相应操作
    let file = fs.openSync(uri, fs.OpenMode.READ_WRITE);
  }
}
```

The shipped `EntryAbility` declares `onCreate(): void` with no `want` parameter
and no `uris` in its `skills`, so this half of the scenario cannot be exercised
from the download alone (HW-05-0077).

## Permissions & config

`demo_StartAbility.zip#entry/src/main/module.json5` - the caller, as shipped:

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
          {
            "entities": ["entity.system.home"],
            "actions": ["action.system.home"]
          }
        ]
      }
    ]
    // no requestPermissions block
  }
}
```

Notes on the config:

- **No permission block**, and that is correct: the file is the caller's own, and
  the target's access is granted through `FLAG_AUTH_READ_URI_PERMISSION` /
  `FLAG_AUTH_WRITE_URI_PERMISSION` on the Want rather than by a declared
  permission.
- The caller's `skills` block is the plain launcher entry. To also *be* a target,
  an application adds `ohos.want.action.viewData` to `actions` and a `uris` array
  with `scheme`, `type` and `linkFeature: "FileOpen"` - `linkFeature` is
  case-sensitive.
- `deviceTypes` is `["phone", "tablet", "2in1"]`.
- `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`action` is fixed.** For file opening the guide specifies
  `'ohos.want.action.viewData'`; any other action will not match a `FileOpen`
  skill.
- **`type` must match the file.** The guide states that if `type` is passed it
  must be the same as the file type behind the URI, otherwise no application is
  matched; and `*/*` is not supported. Omitting `type` lets the system infer from
  the URI suffix.
- **UTD is preferred over MIME.** `'general.plain-text'`, `'general.image'` and
  friends; MIME types such as `'text/xml'` also work.
- **`showDefaultPicker` defaults to `false`.** Without it the system may open the
  user's default application directly instead of offering a choice.
- **Authorization is per-URI and per-launch.** The read/write flags grant the
  target access to that URI for that invocation; there is no persistent grant.
- **The result depends on what is installed.** If no application declares a
  matching `FileOpen` skill, `startAbility` rejects - which is why its promise
  must be handled.
- **The caller cannot be its own target by default.** The `showCaller` parameter
  (default `false`) controls whether the calling application appears in the
  chooser when it also matches.

## Pitfalls

- **The document's Want omits `showDefaultPicker`, which is incorrect** for a
  page titled "choose how to open a file": the parameter defaults to `false`, so
  without it the chooser the scenario demonstrates may never appear. The sample
  includes it. (HW-05-0076)
- **Steps 2 and 3 describe code the download does not contain, and the document
  does not say so.** The shipped `module.json5` declares no `viewData` action and
  no `uris`, and `EntryAbility.onCreate` takes no `want`. Treat the target side as
  specification, not as shipped sample. (HW-05-0077)
- **`(index: number) => JSON.stringify(index)` as a `ForEach` key generator is
  incorrect.** The first parameter is the data item, not the index, so a `File`
  object is bound to a parameter declared `number`. Key on a stable per-item
  value instead - the same guide also warns against using the index as a key.
  (HW-05-0078)
- **Calling a no-argument `Utils.startAbility()` from every row is incorrect** for
  a list that advertises `.txt`, `.pdf` and `.jpg`: the helper hard-codes
  `1.txt` and `general.plain-text`, and the guide requires the declared type to
  match the file behind the URI. Pass the row in. (HW-05-0079)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect** -
  release it in `onWindowStageDestroy`. (HW-05-0080)
- Not recorded as a separate finding, but worth fixing when adapting the sample:
  `AppStorage.setOrCreate('uiContext', ...)` runs before the `err.code` guard in
  `loadContent`, and `fs.openSync` sits outside the `try` whose `catch` is meant
  to cover it.

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/03_application-framework/file-processing-apps-startup.md` -
  Table 1 (`uri`, `type`, `action`, `flags`), Table 2
  (`ohos.ability.params.showDefaultPicker`, `showCaller`, `ability.params.stream`)
  and Table 3 (the two URI-permission flags).
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/file-processing-apps-startup
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the `keyGenerator` signature `(item: Object, index: number)` and the warning
  against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` as the status-bar area.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `OpenMode`, `openSync`, `writeSync`, `closeSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
