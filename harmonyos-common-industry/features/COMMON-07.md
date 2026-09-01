---
id: COMMON-07
title: Application cache clearing - enumerate every cache directory, delete its contents, report the freed size
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
sample: huawei_industry_tree/19_common_technical_solutions/downloads/CacheClean.zip
kits: ["@kit.CoreFileKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: ["application.createModuleContext", "Context.getApplicationContext", "Context.cacheDir", "Context.area", "contextConstant.AreaMode", "UIContext.getHostContext", "fileIo.listFile", "fileIo.statSync", "fileIo.rmdirSync", "fileIo.unlink", "fileIo.createStreamSync", "storageStatistics.getCurrentBundleStats", "PromptAction.openCustomDialog", ComponentContent, CustomDialogController, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0004, HW-19-0005, HW-19-0006, HW-19-0007, HW-19-0008, HW-19-0181, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a settings screen needs a **清除缓存 / clear cache** entry:
show how much space the application currently occupies, confirm with the user,
delete the cached files, and refresh the displayed size. The document calls this
one of the highest-frequency scenarios across all application categories.

The whole feature is built on `@ohos.file.fs` (`fileIo`, imported from
`@kit.CoreFileKit`) plus context path resolution - there is no dedicated
"clear cache" system API to call.

## Feature checklist

The application must:

- Display the current cache size on the settings row.
- Enumerate **every** cache directory the application owns: application-level and
  module-level, in both the EL1 and EL2 encryption areas.
- Restore each context's encryption area after enumerating, so the rest of the
  app keeps resolving paths correctly.
- Ask for confirmation before deleting anything.
- Delete the contents of each cache directory, handling files and subdirectories
  differently.
- Show a progress or completion dialog.
- Re-read and re-display the cache size afterwards.
- Log, not swallow, every file-system failure.

## Architecture

Single-module project (`entry` HAP only), three source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/CacheClean`, forces light colour mode, sets full-screen layout, publishes status-bar and navigation-indicator heights into `AppStorage` and keeps them current through `avoidAreaChange` |
| `model/SettingModel.ets` | the `SettingItem` interface and the `SETTING_LIST` constant backing the settings list |
| `pages/CacheClean.ets` | the whole feature: path resolution, deletion, size query, confirmation dialog and the settings UI |

Data flow:

1. `aboutToAppear` resolves the cache paths (`getPaths`), seeds a test file into
   each of them (`writeFile`, demo scaffolding only), reads the size
   (`getCache`), and publishes the `UIContext` / `PromptAction` into `AppStorage`
   so the `@CustomDialog` can reach them.
2. Tapping the 缓存管理 ("cache management") row opens `ConfirmCustomDialog`.
3. Confirming calls `clearAllCache()`, which walks `paths` and calls
   `deleteFile()` on each; `deleteFile` lists the directory, then
   `rmdirSync` for subdirectories and `unlink` for files.
4. A timer re-runs `getCache()`, whose
   `storageStatistics.getCurrentBundleStats` callback updates the `@State cache`
   string and therefore the row.

**Path resolution is where the shipped code is wrong.** The official context
guide gives `cacheDir` as `<prefix>/<level>/base/cache` for **ApplicationContext
only**, and `<prefix>/<level>/base/haps/<module-name>/cache` for
AbilityStageContext, UIAbilityContext and ExtensionContext. The sample derives
both of its "different" paths from module-level contexts
(`application.createModuleContext(...)` and the UIAbilityContext from
`getHostContext()`), so it collects the same module path twice per area and never
touches the application-level cache - see HW-19-0004. The corrected resolution is
in the implementation steps below.

## Implementation steps

1. **Get the two contexts.** `const ctx = this.getUIContext().getHostContext() as
   common.UIAbilityContext` for the module-level paths, and
   `ctx.getApplicationContext()` for the application-level ones.
2. **Collect the EL2 paths**: `appCtx.cacheDir`
   (`/data/storage/el2/base/cache`) and `ctx.cacheDir`
   (`/data/storage/el2/base/haps/entry/cache`).
3. **Save both current areas**, then set `appCtx.area` and `ctx.area` to
   `contextConstant.AreaMode.EL1` and collect the same two `cacheDir` values
   again - now the EL1 pair.
4. **Restore the saved areas.** The document is explicit:
   "获取完需要清理的路径之后，moduleContext.area和context.area需要切换回应用原本的
   加密分区，否则可能会影响其他业务功能。" ("after obtaining the paths to be
   cleaned, moduleContext.area and context.area must be switched back to the
   application's original encryption area, otherwise other business functions may
   be affected.") Restore the captured values, not a hardcoded `EL2`
   (HW-19-0008).
5. **Read the current size** with
   `storageStatistics.getCurrentBundleStats()` and render
   `bundleStats.cacheSize`.
6. **Confirm with the user** through a `@CustomDialog`, and open a progress
   dialog with `promptAction.openCustomDialog(componentContent, {...})` on
   confirm.
7. **Delete per directory**: `fileIo.listFile(dir)`, then for each entry
   `fileIo.statSync(path).isDirectory()` decides between `fileIo.rmdirSync(path)`
   - which "Removes a directory and all its subdirectories and files
   synchronously" - and `fileIo.unlink(path)`.
8. **Handle every rejection.** Chain `.catch` on the `listFile` promise
   (HW-19-0006) and log inside every `catch` block (HW-19-0005).
9. **Refresh the size** after deletion completes.

If you also want a settings list split into two groups, precompute the slices
with `slice`; do not call `splice` on the shared constant from inside `build()`
(HW-19-0007).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Path resolution (as shipped - see HW-19-0004 and HW-19-0008 before reusing)

`CacheClean.zip#CacheClean/entry/src/main/ets/pages/CacheClean.ets`

```ts
// 初始化所有缓存路径
async getPaths(): Promise<string[]> {
  let paths: string[] = [];
  let moduleContext: common.Context;
  moduleContext = await application.createModuleContext(this.context, 'entry');

  // 缓存路径 /data/storage/el2/base/cache
  hilog.info(0x0000, 'CacheClean', `moduleContext + el2: ${moduleContext.cacheDir}`);
  // 缓存路径 /data/storage/el2/base/haps/entry/cache
  hilog.info(0x0000, 'CacheClean', `UIAbilityContext + el2: ${this.context.cacheDir}`);

  paths.push(moduleContext.cacheDir);
  paths.push(this.context.cacheDir);

  // 缓存路径 /data/storage/el1/base/cache
  moduleContext.area = contextConstant.AreaMode.EL1;
  // 缓存路径 /data/storage/el1/base/haps/entry/cache
  this.context.area = contextConstant.AreaMode.EL1;

  paths.push(moduleContext.cacheDir);
  paths.push(this.context.cacheDir);

  moduleContext.area = contextConstant.AreaMode.EL2;
  this.context.area = contextConstant.AreaMode.EL2;
  return paths;
}
```

Corrected form:

```ts
async getPaths(): Promise<string[]> {
  const paths: string[] = [];
  const appCtx = this.context.getApplicationContext();
  const prevAppArea = appCtx.area;
  const prevCtxArea = this.context.area;

  // EL2: application level + module level
  paths.push(appCtx.cacheDir);        // /data/storage/el2/base/cache
  paths.push(this.context.cacheDir);  // /data/storage/el2/base/haps/entry/cache

  appCtx.area = contextConstant.AreaMode.EL1;
  this.context.area = contextConstant.AreaMode.EL1;
  paths.push(appCtx.cacheDir);        // /data/storage/el1/base/cache
  paths.push(this.context.cacheDir);  // /data/storage/el1/base/haps/entry/cache

  appCtx.area = prevAppArea;
  this.context.area = prevCtxArea;
  return paths;
}
```

### Deletion

`CacheClean.zip#CacheClean/entry/src/main/ets/pages/CacheClean.ets`

```ts
deleteFile(cacheDir: string) {
  fs.listFile(cacheDir)
    .then((filenames) => {
      filenames.forEach((name) => {
        let dirPath = cacheDir + '/' + name;
        hilog.info(0x0000, 'CacheClean', dirPath);
        // 判断是否为文件夹
        let isDirectory: boolean = false;
        try {
          isDirectory = fs
            .statSync(dirPath)
            .isDirectory();
        } catch (e) {
          hilog.error(0x0000, 'CacheClean', JSON.stringify(e));
        }
        if (isDirectory) {
          fs.rmdirSync(dirPath);
        } else {
          fs.unlink(dirPath)
            .then(() => {
              hilog.info(0x0000, 'CacheClean', 'Remove files successfully');
            })
            .catch((err: Error) => {
              let log = `Failed to remove files with error message: ${err.message}`;
              hilog.error(0x0000, 'CacheClean', log);
            });
        }
      });
    });
  // FIX (HW-19-0006): add
  // .catch((err: BusinessError) => hilog.error(0x0000, 'CacheClean',
  //   `Failed to list ${cacheDir}. Code: ${err.code}, message: ${err.message}`));
}

// 清理所有缓存
clearAllCache() {
  this.paths.forEach((cacheDir) => {
    hilog.info(0x0000, 'CacheClean', cacheDir);
    this.deleteFile(cacheDir);
  });
}
```

### Reading the cache size

`CacheClean.zip#CacheClean/entry/src/main/ets/pages/CacheClean.ets`

```ts
// 获取应用数据空间大小
getCache() {
  storageStatistics.getCurrentBundleStats((error: BusinessError, bundleStats: storageStatistics.BundleStats) => {
    if (error) {
      let log = `Failed to getCurrentBundleStats with error: ${JSON.stringify(error)}`;
      hilog.error(0x0000, 'CacheClean', log);
    } else {
      this.cache = `${bundleStats.cacheSize}B`;
      hilog.info(0x0000, 'CacheClean', this.cache); // 总缓存大小
    }
  });
}
```

### Progress dialog through PromptAction + ComponentContent

`CacheClean.zip#CacheClean/entry/src/main/ets/pages/CacheClean.ets`

```ts
function setDialog(uiContext: UIContext, promptAction: PromptAction) {
  let contentNode = new ComponentContent(uiContext, wrapBuilder(customDialogComponent));
  promptAction.openCustomDialog(contentNode, { maskColor: Color.Transparent });
}

// published in aboutToAppear so the @CustomDialog can reach them
AppStorage.setOrCreate('promptAction', this.promptAction);
AppStorage.setOrCreate('uiContext', this.uiContext);
```

### Full-screen layout and avoid-area heights

`CacheClean.zip#CacheClean/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let windowClass: window.Window = windowStage.getMainWindowSync();
let isLayoutFullScreen = true;
windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(() => {
  hilog.info(0x0000, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
}).catch((err: BusinessError) => {
  hilog.error(0x0000, 'testTag',
    'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
});
let avoidAreaTop = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
let avoidAreaBottom = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
AppStorage.setOrCreate('topRectHeight', avoidAreaTop.topRect.height);
AppStorage.setOrCreate('bottomRectHeight', avoidAreaBottom.bottomRect.height);
windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});
```

## Permissions & config

**No permissions are required.** Neither the document nor the sample declares
any, and none of the APIs used need one: `fileIo` operates inside the
application sandbox, and `storageStatistics.getCurrentBundleStats` reports only
this application's own usage.

`CacheClean.zip#CacheClean/entry/src/main/module.json5` - as shipped:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "description": "$string:module_desc",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "description": "$string:EntryAbility_desc",
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ]
  }
}
```

`CacheClean.zip#CacheClean/build-profile.json5`: `compatibleSdkVersion` is
`6.0.0(20)`, `runtimeOS` is `HarmonyOS`, and `strictMode` has both
`caseSensitiveCheck` and `useNormalizedOHMUrl` enabled.

## Constraints

- **API level.** The document states API Version 20 Release or later, HarmonyOS
  6.0.0 Release SDK or later, and DevEco Studio 6.0.0 Release or later; the
  project pins `compatibleSdkVersion: "6.0.0(20)"`.
- **`storageStatistics.getCurrentBundleStats` devices**: Phone, PC/2in1, Tablet,
  TV. The sample declares `phone`, `tablet`, `2in1`, all of which are covered.
- **`application.createModuleContext` is expensive.** The official reference
  warns: "Creating a module context involves resource querying and
  initialization, which can be time-consuming. In scenarios where application
  fluidity is critical, avoid frequently or repeatedly calling the
  **createModuleContext** API to create multiple context instances." Resolve the
  paths once, as the sample does in `aboutToAppear`.
- **`Context.area` is shared state.** Changing it changes path resolution for
  everything using that context, which is why it must be restored.
- **`rmdirSync` is recursive**: "Removes a directory and all its subdirectories
  and files synchronously." No manual recursion is needed - but the deletion runs
  on the calling thread, so a large cache tree will block it.
- **`cacheSize` is in bytes**; the sample renders it raw as `${cacheSize}B`
  without unit scaling.

## Pitfalls

- **The document says `moduleContext.cacheDir` is `/data/storage/el2/base/cache`,
  which is incorrect.** A module context - like the UIAbilityContext - resolves
  `cacheDir` to `/data/storage/el2/base/haps/entry/cache`; only an
  **ApplicationContext** resolves it to `/data/storage/el2/base/cache`. As
  shipped, the sample collects the same module path twice per encryption area and
  never clears the application-level cache, which is where ArkWeb keeps its cache
  (`/data/storage/el2/base/cache/web/`). Use `ctx.getApplicationContext()` for the
  application-level pair. (HW-19-0004)
- **The sample restores both contexts to a hardcoded `AreaMode.EL2`, which is
  incorrect.** The document requires switching back to 应用原本的加密分区 ("the
  application's original encryption area"); capture the previous value and
  restore that. (HW-19-0008)
- **`deleteFile` leaves the `fs.listFile` promise unhandled, which is
  incorrect.** The official reference example chains
  `.catch((err: BusinessError) => ...)`; without it a directory that cannot be
  listed is skipped silently while the UI reports success. (HW-19-0006)
- **`writeFile` has an empty `catch (e) {}`, which is incorrect.** Log the error
  with `hilog.error` like every other error path in the file. (HW-19-0005)
- **`ForEach(SETTING_LIST.splice(0, 2), ...)` inside `build()` is incorrect.**
  `splice` deletes from the exported constant; use `slice` and precompute the two
  groups outside `build()`. (HW-19-0007)
- **`writeFile()` is demo scaffolding, not part of the feature.** It creates a
  `test.txt` in every cache directory so the demo has something to delete. It is
  not described in the document's 实现思路 and must be removed from production
  code.
- **Deletion is synchronous for directories.** `rmdirSync` and `statSync` run on
  the UI thread here. For a large cache, move the walk to a TaskPool task - the
  sibling performance document (COMMON-04) states the rule directly:
  避免在自定义组件的生命周期内执行高耗时操作 ("avoid executing time-consuming
  operations inside custom component lifecycle callbacks").

## References

- `documentation/harmonyos-guides/03_application-framework/application-context-stage.md` -
  Table 1, the per-context-type application file path table that gives
  `cacheDir` as `base/cache` for ApplicationContext and
  `base/haps/<module-name>/cache` for module-level contexts.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/application-context-stage
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-application.md` -
  `application.createModuleContext(context, moduleName): Promise<Context>`, its
  "Creates the context for a module" description and the cost warning.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-application
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `listFile` (and its `.catch` example), `statSync`, `unlink`, and the recursive
  semantics of `rmdir` / `rmdirSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-file-storage-statistics.md` -
  `storageStatistics.getCurrentBundleStats`, its import from `@kit.CoreFileKit`,
  supported devices and error codes.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-storage-statistics
- `documentation/harmonyos-guides/03_application-framework/web-download.md` and
  `web-cookie-and-data-storage-mgmt.md` - ArkWeb cache and cookie storage under
  `/data/storage/el2/base/cache/web/`, the directory HW-19-0004 leaves untouched.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-download
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
