---
id: COMMON-51
title: Persisted file permissions - keeping read/write access to picked files across launches
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PersistPermissionDemo.zip
kits: ["@kit.CoreFileKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: ["picker.DocumentViewPicker", "picker.DocumentSelectOptions", "DocumentViewPicker.select", "fileShare.persistPermission", "fileShare.activatePermission", "fileShare.PolicyInfo", "fileShare.OperationMode", canIUse, "preferences.getPreferencesSync", "preferences.getPreferences", "preferences.removePreferencesFromCacheSync", "Preferences.putSync", "Preferences.getSync", "Preferences.clearSync", "Preferences.flushSync", "taskpool.Task", "@Concurrent", "fileIo.openSync", "fileIo.copyFileSync", "fileIo.closeSync", "fileIo.rmdirSync", Navigation, NavPathStack, LazyForEach]
permissions: ["ohos.permission.FILE_ACCESS_PERSIST"]
min_api: 20
modules: [entry]
findings: [HW-19-0173, HW-19-0174, HW-19-0175, HW-19-0176, HW-19-0177, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the user picks files once and the application must be able to
read or write them **again on a later launch** without re-opening the picker -
an editor's recent-files list, a batch importer, a document manager.

The default behaviour is the opposite: a uri returned by `DocumentViewPicker`
grants access for the current session only. `fileShare.persistPermission` turns
that into a lasting grant, and `fileShare.activatePermission` re-arms it in a new
process. The pair is the whole feature.

Before using this sample's code, read HW-19-0177 - its import task deletes the
application's entire files directory when a single copy fails.

## Feature checklist

- Declare `ohos.permission.FILE_ACCESS_PERSIST` - a **restricted** permission
  requiring an approved application.
- `canIUse('SystemCapability.FileManagement.AppFileService.FolderAuthorization')`
  before relying on any of it.
- `DocumentViewPicker.select(options)` → array of uris.
- Build `PolicyInfo[]` with `READ_MODE | WRITE_MODE`, then
  `persistPermission(policies)` - and report a failure (HW-19-0173).
- Store the uris (Preferences), through the same instance you read them with
  (HW-19-0174).
- On a later launch, `activatePermission(policies)` before touching the files.

## Architecture

Single-module project (`entry` HAP), the largest in this industry:

| Directory | Contents |
| --- | --- |
| `common/` | `Constants.ets`, `FileUtil.ets`, `Logger.ets`, `PersistPermissionUtils.ets` |
| `components/` | `FileItem.ets` |
| `model/` | `FileDataSource.ets` (an `IDataSource` for `LazyForEach`), `FileModel.ets` |
| `pages/` | `Index.ets`, `QueryFilePage.ets` |
| `taskpool/base/` | `BaseTask.ets`, `TaskExecutor.ets`, `TaskManager.ets` |
| `taskpool/` | `ImportFileTask.ets`, `GenerateThumbnailTask.ets`, `QuerySandBoxFile.ets` |

**Two permissions, two moments.** This is the distinction the whole feature turns
on:

| | when | what it does |
| --- | --- | --- |
| `persistPermission(policies)` | once, right after the picker returns | records a lasting grant for those uris |
| `activatePermission(policies)` | every launch, before any file access | makes the recorded grant usable in this process |

Persisting without activating gives a grant the current process cannot use;
activating without having persisted fails. The uris must be stored somewhere
between the two - here, in a `Preferences` store named `Document`.

**`PolicyInfo` is a uri plus a mode.** The sample requests both directions with a
bitwise or: `fileShare.OperationMode.WRITE_MODE | fileShare.OperationMode.READ_MODE`.
Request only what you need - a viewer should persist `READ_MODE` alone.

**Capability is checked, not assumed.** `canIUse('SystemCapability.FileManagement.
AppFileService.FolderAuthorization')` guards the whole flow, with a distinct toast
(`设备不支持持久化能力！`, "this device does not support the persistence
capability") for the unsupported case. That check is the right pattern for any
`SystemCapability`-gated feature.

**Work happens on a taskpool worker.** `ImportFileTask` wraps an `@Concurrent`
function in a `taskpool.Task`, so the file copying does not touch the UI thread -
and note that the worker re-activates permissions itself
(`await getPersistPermissionUtils().activatePermissionExample(uris)` on entry),
because a taskpool worker is a separate ArkTS context and does not inherit the
main thread's activation.

**The singleton is initialized from the page.** `getPersistPermissionUtils().init(
this.getUIContext())` in `Index.aboutToAppear`, because the utility needs a
`UIContext` for toasts and a `Context` for `Preferences`.

## Implementation steps

1. **Declare the restricted permission** in `module.json5` and obtain approval for
   it - the document notes 需采用受限权限申请方式申请使用该权限 ("this permission
   must be applied for through the restricted-permission application process").
2. **Check the capability** with `canIUse(...FolderAuthorization)`.
3. **Pick files**:
   ```ts
   const options = new picker.DocumentSelectOptions();
   options.fileSuffixFilters = ['.txt', '.pdf', ...];
   const uris = await new picker.DocumentViewPicker().select(options);
   ```
4. **Persist**: map the uris to `PolicyInfo[]` and `await
   fileShare.persistPermission(policies)`. Treat a rejection as a failure the user
   must see (HW-19-0173).
5. **Store the uris** in `Preferences`, merging with what is already there
   (HW-19-0175), through the instance you also read with (HW-19-0174).
6. **On a later launch**, read the stored uris and `await
   fileShare.activatePermission(policies)` before any `fileIo` call - including
   inside every taskpool worker that will touch the files.
7. **Copy or edit**, cleaning up only what the failing operation created
   (HW-19-0177).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Picking files

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
import { picker } from '@kit.CoreFileKit';
import { fileShare } from '@kit.CoreFileKit';
import { preferences } from '@kit.ArkData';

// 文件选择器
public async fileSelectPickerExample() {
  let pathArray: Array<string> = [];
  try {
    let documentSelectOptions = new picker.DocumentSelectOptions();
    documentSelectOptions.fileSuffixFilters = ['.txt', '.pdf', '.docx', '.doc', '.xlsx', '.pptx', 'rtf', '.html',
      '.htm', '.md', '.json', '.epub', '.caj', '.zip'];
    let documentPicker = new picker.DocumentViewPicker();
    pathArray = await documentPicker.select(documentSelectOptions);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.error(TAG, `fileSelectPickerExample failed with err, Error code: ${err.code}, message: ${err.message}`);
  }
  return pathArray;
}
```

Returning `[]` on error rather than throwing is a reasonable choice for a picker -
cancellation and failure both mean "no files" - but it does mean the caller cannot
tell a cancel from a fault. Note `'rtf'` in the filter list is missing its leading
dot while every other entry has one.

### Building the policies

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
// 根据文件路径生成文件策略信息
public getFilePolicyInfo(pathArray: Array<string>) {
  let policies: Array<fileShare.PolicyInfo> = [];
  for (let index = 0; index < pathArray.length; index++) {
    policies.push({
      uri: pathArray[index],
      operationMode: fileShare.OperationMode.WRITE_MODE | fileShare.OperationMode.READ_MODE
    });
  }
  return policies;
}
```

### Persisting and activating

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
// 申请持久化权限
public async persistPermissionExample(pathArray: Array<string>) {
  let policies: Array<fileShare.PolicyInfo> = this.getFilePolicyInfo(pathArray);
  try {
    await fileShare.persistPermission(policies);
    if (this.dataPreferences) {
      this.dataPreferences.clearSync();          // FIX (HW-19-0175): replaces the whole list
      this.dataPreferences.putSync('pathArray', pathArray);
      this.dataPreferences.flushSync();
    }
    Logger.info(TAG, 'persistPermission successfully');
  } catch (err) {
    Logger.error(TAG, 'persistPermission failed with error message: ' + err.message + ', error code: ' + err.code);
    this.dataPreferences?.clearSync();           // FIX (HW-19-0173): failure only logged
  }
}

// 唤醒持久化权限
public async activatePermissionExample(pathArray: Array<string>) {
  try {
    let policies: Array<fileShare.PolicyInfo> = this.getFilePolicyInfo(pathArray);
    await fileShare.activatePermission(policies);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.error(TAG, `activatePermissionExample failed with err, Error code: ${err.code}, message: ${err.message}`);
  }                                              // FIX (HW-19-0173): failure only logged
}
```

### The capability check

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
// 检查系统能力
public checkSystemCapability() {
  if (!canIUse('SystemCapability.FileManagement.AppFileService.FolderAuthorization')) {
    console.error('this FolderAuthorization is not supported on this device');
    return false;
  }
  return true;
}
```

### The orchestration

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
public async persistPermission() {
  let pathArray = await this.getFilePathExist(this.context?.getHostContext() as Context);
  if (pathArray.length === 0) {                  // FIX (HW-19-0175): picker unreachable afterwards
    if (this.checkSystemCapability()) {
      pathArray = await this.fileSelectPickerExample();
      await this.persistPermissionExample(pathArray);
      if (pathArray && pathArray.length > 0) {
        this.showToast(this.TOAST_MSG);          // FIX (HW-19-0173): success regardless
      }
    } else {
      this.showToast(this.TOAST_MSG1);
    }
  } else {
    await this.activatePermissionExample(pathArray);
    this.showToast(this.TOAST_MSG2);             // FIX (HW-19-0173): unconditional
  }
  return pathArray;
}
```

The three toast strings encode the three outcomes the design intends -
`文件导入完毕并持久化读写授权！`, `设备不支持持久化能力！`,
`文件已持久化读写权限，无需拉起picker！` - and there is no fourth for
"authorization failed", which is the gap.

### Reading the stored uris

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/PersistPermissionUtils.ets`

```ts
public init(context: UIContext) {
  this.context = context;
  this.dataPreferences = preferences.getPreferencesSync(context.getHostContext() as Context, {
    name: 'Document'
  });
}

// 获取首选项中存储的文件路径
public async getFilePathExist(context: Context): Promise<Array<string>> {
  preferences.removePreferencesFromCacheSync(context, 'Document');   // FIX (HW-19-0174)
  const pf = await preferences.getPreferences(context, 'Document');
  const data = pf.getSync('pathArray', []) as Array<string>;
  return data;
}
```

Corrected, using the one cached instance:

```ts
public getFilePathExist(): Array<string> {
  return (this.dataPreferences?.getSync('pathArray', []) ?? []) as Array<string>;
}
```

### Copying on a worker

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/taskpool/ImportFileTask.ets`

```ts
import { taskpool } from '@kit.ArkTS';
import { fileIo as fs } from '@kit.CoreFileKit';

/**
 * 将图库文件保存到沙箱
 */
export class ImportFileTask extends BaseTask {
  constructor(path: string, uris: string[], callback: Function) {
    super('createFileTask');
    this.task = new taskpool.Task(startTask, path, uris);
    this.callback = callback;
  }
}

@Concurrent
async function startTask(path: string = '', uris: string[]) {
  const TAG = 'ImportFileTask';
  if (uris === undefined || uris.length === 0) {
    Logger.error(TAG, 'uri is null');
    return;
  }
  if (path === '') {
    Logger.error(TAG, 'path is null');
    return;
  }
  await getPersistPermissionUtils().activatePermissionExample(uris);
  for (let i = 0; i < uris.length; i++) {
    let nameIndex = uris[i].lastIndexOf('/');
    let name = uris[i].substring(nameIndex, uris[i].length);
    let srcFile: fs.File | undefined = undefined;
    let destFile: fs.File | undefined = undefined;
    try {
      srcFile = fs.openSync(uris[i], fs.OpenMode.READ_ONLY);
      destFile = fs.openSync(path + name, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
      fs.copyFileSync(srcFile.fd, destFile.fd);
      Logger.info(TAG, 'copy success!');
    } catch (error) {
      Logger.error(TAG, 'copy fail: ' + error.toString());
      fs.rmdirSync(path);                        // FIX (HW-19-0177): deletes filesDir
      Logger.error(TAG, 'rmdir ' + path);
    } finally {
      if (srcFile) {
        fs.closeSync(srcFile);
      }
      if (destFile) {
        fs.closeSync(destFile);
      }
    }
  }
}
```

Two things here are exactly right and worth copying: the `finally` closes both
descriptors on every path including the failure, and the worker calls
`activatePermissionExample` itself rather than assuming the main thread's
activation carries over. The `catch` is the part to replace:

```ts
} catch (error) {
  Logger.error(TAG, 'copy fail: ' + error.toString());
} finally {
  if (srcFile) { fs.closeSync(srcFile); }
  if (destFile) { fs.closeSync(destFile); }
  // remove only the partial destination this iteration created
}
```

Note that `name` keeps the leading `/` because `substring` starts at the index of
the separator, which is what makes `path + name` a valid path.

### Display helpers

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/ets/common/FileUtil.ets`

```ts
// 转换大小
static transferSizeToStr(size: number): string {
  const UNITS: string[] = ['B', 'KB', 'MB', 'GB', 'TB'];
  let count = 0;
  let unit: string = UNITS[count];
  let res = size;
  let resTmp = size / 1024;

  while (resTmp >= 1 && count < 4) {
    count++;
    res = Math.floor(resTmp * 10) / 10;
    unit = UNITS[count]
    resTmp = resTmp / 1024;
  }
  return res + unit;
}

static anonymousFileName(fileName: string) {
  const NUM = 6
  return fileName.substring(0, NUM) + '*'.repeat(NUM) + fileName.substring(fileName.length - NUM);
}
```

`anonymousFileName` is a good habit in a file manager - it is used where a name
must be shown but need not be fully legible. It assumes a name of at least 12
characters; a shorter one yields overlapping slices.

## Permissions & config

`PersistPermissionDemo.zip#PersistPermissionDemo/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  {"name": "ohos.permission.FILE_ACCESS_PERSIST"}
],
"pages": "$profile:main_pages",
"routerMap": "$profile:router_map",
```

- **`ohos.permission.FILE_ACCESS_PERSIST`** - a **restricted** permission, listed
  under `restricted-permissions` rather than `permissions-for-all`. It cannot
  simply be declared: the document's 说明 states
  需采用受限权限申请方式申请使用该权限 ("this permission must be applied for
  through the restricted-permission application process"), meaning an approval
  request to Huawei tied to the signing certificate. Without approval the build
  installs but `persistPermission` fails.
- **No storage permission is needed.** Access comes from the picker's grant plus
  the persisted policy, not from a broad file-system permission - which is the
  point of the whole mechanism.

`build-profile.json5` pins `6.0.0(20)` for both `compatibleSdkVersion` and
`targetSdkVersion`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`FILE_ACCESS_PERSIST` is restricted** and requires an approved application.
- **The capability may be absent.** Gate on
  `SystemCapability.FileManagement.AppFileService.FolderAuthorization`.
- **Persist and activate are both required.** Persisting records the grant;
  activating makes it usable in the current process.
- **Every ArkTS context needs its own activation** - a taskpool worker does not
  inherit the main thread's.
- **The uris must be stored by the application.** Nothing enumerates persisted
  grants for you.
- **A persisted grant can still fail** - the user can delete or move the file, or
  revoke access.
- **`fileSuffixFilters` entries need a leading dot** to match; `'rtf'` in this
  sample does not have one.
- **`removePreferencesFromCacheSync` invalidates an instance.** The reference:
  "Avoid using a removed Preferences instance to perform data operations."

## Pitfalls

- **A failed copy calls `fs.rmdirSync(context.filesDir)`, which is incorrect** -
  one unreadable uri deletes every file the application has ever imported, plus
  anything else under `filesDir`. Remove only the partial destination.
  (HW-19-0177)
- **`persistPermissionExample` and `activatePermissionExample` swallow their
  failures, which is incorrect** - the success toast fires regardless and the
  import task then runs on uris the process cannot open. (HW-19-0173)
- **`getFilePathExist` evicts the store from the cache and opens a second
  instance while all writes go through the first, which is incorrect** - the
  reference warns that using a removed instance causes data inconsistency.
  (HW-19-0174)
- **The picker is unreachable once anything is stored, and each import clears the
  list before writing - both incorrect** for a feature whose scenario is batch
  import with a history. (HW-19-0175)
- **The 工程目录 marks every child as the last one and writes `pages` with a
  malformed connector, and `PersistPermissionUtils.ets` is headed `// Index.ets`
  - all incorrect.** (HW-19-0176)
- **Do not request `WRITE_MODE` for a viewer.** `OperationMode` is a bitmask;
  persist the narrowest grant the feature needs.
- **Do not assume a persisted uri still resolves.** Check before use and drop
  dead entries from the stored list, or the recent-files view accumulates
  unopenable rows.
- **Do not persist without storing the uris.** The grant exists but nothing can
  name it, and there is no API to enumerate what you hold.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-fileshare -
  `persistPermission`, `activatePermission`, `PolicyInfo`, `OperationMode`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/file-persistpermission -
  the authorization-persistence guide, including the restricted-permission
  requirement.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-file-picker -
  `DocumentViewPicker`, `DocumentSelectOptions.fileSuffixFilters`.
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `getPreferencesSync`, `getPreferences`, `removePreferencesFromCacheSync` and its
  warning against reusing a removed instance, `putSync` / `getSync` /
  `clearSync` / `flushSync`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-data-preferences
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-file-fs -
  `openSync`, `copyFileSync`, `closeSync`, `rmdirSync`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-taskpool -
  `taskpool.Task` and `@Concurrent`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/restricted-permissions -
  `ohos.permission.FILE_ACCESS_PERSIST`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
