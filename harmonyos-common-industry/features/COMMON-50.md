---
id: COMMON-50
title: Navigation continuation - migrating the route stack and its data to another device
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/50_navigation_continue.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
sample: huawei_industry_tree/19_common_technical_solutions/downloads/NavigationContinue.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIAbility.onContinue", "UIAbility.onCreate", "UIAbility.onNewWant", "UIAbility.onWindowStageRestore", "AbilityConstant.OnContinueResult", "AbilityConstant.LaunchReason.CONTINUATION", "distributedDataObject.create", "distributedDataObject.genSessionId", "DataObject.setSessionId", "DataObject.on('status')", "DataObject.save", "bundleManager.getBundleInfoForSelfSync", Navigation, NavPathStack, "NavPathStack.getAllPathName", "NavPathStack.getParamByName", "NavPathStack.pushPathByName", "@StorageLink", "@StorageProp", routerMap, onPageShow]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0169, HW-19-0170, HW-19-0171, HW-19-0172, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the user must be able to pick up on a second device exactly
where they left off - a note being edited, a document being read, a form half
filled - and the application uses `Navigation` rather than the legacy router.

The hard part is not the continuation itself; the system does the handover. The
hard part is that **a `NavPathStack` cannot be migrated**. What crosses the wire
is a route *name* and a *parameter blob*, and the target device rebuilds a stack
from them. Getting that reconstruction to happen exactly once is where this
sample goes wrong (HW-19-0169).

## Feature checklist

- `"continuable": true` on the ability in `module.json5`.
- `onContinue`: version-check, read the top route from the `NavPathStack`, write
  it to the `wantParam` or into a distributed data object, return `AGREE`.
- `onCreate` / `onNewWant`: guard on `LaunchReason.CONTINUATION`, restore the
  route data into `AppStorage`.
- `onWindowStageRestore`: `loadContent` the Navigation root page.
- Root page `onPageShow`: push the migrated route **and clear the flag**
  (HW-19-0169).
- Release the distributed object's listener (HW-19-0170).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | the whole continuation protocol - `PageModel`, `DATA_MIGRATE_TYPE`, `onContinue`, `restoreData`, `onWindowStageRestore` |
| `pages/Index.ets` | Navigation root; consumes the migrated route |
| `views/PageOne.ets`, `views/PageTwo.ets` | `NavDestination`s reached by name |
| `resources/base/profile/route_map.json` | the `routerMap` binding names to builders |

**Two migration channels, chosen at runtime.** The sample implements both and
lets the user toggle between them, which is the most useful thing it teaches:

| | `Want` parameters | distributed data object |
| --- | --- | --- |
| enum | `DATA_MIGRATE_TYPE.WANT_TYPE` (1) | `DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE` (0) |
| carries | small values in `wantParam` | an object synchronised across devices |
| suited to | lightweight data | the sample's own comment: 数据较大（100KB以上）或需要迁移文件 ("data larger than 100 KB, or file migration") |
| target reads it | synchronously in `restoreData` | asynchronously, on `status === 'restored'` |
| needs | nothing extra | a session id passed through the want |

**The session id is the bridge between the two channels.** Even the
distributed-object path uses the `Want` - it carries `dataSessionId`, generated
by `genSessionId()` on the source, so the target can join the same object
session. And `save(targetDevice)` persists the object so the data survives the
source application exiting.

**`setSessionId` needs no permission here.** The sample flags this twice:
`setSessionId接口在应用接续场景下无需申请ohos.permission.DISTRIBUTED_DATASYNC权限，
在其他场景需要` ("in the application-continuation scenario setSessionId does not
require the DISTRIBUTED_DATASYNC permission; in other scenarios it does"). That
is why `module.json5` declares no permissions at all.

**Why the `DataObject` is an ability field.** The comment on the declaration is
the load-bearing insight: `数据恢复时，分布式数据对象状态监听有延迟，需要在此处定义
该对象，避免对象销毁导致未接收到对端迁移数据` ("during restoration the status
listening is delayed, so the object must be declared here, to prevent its
destruction causing the peer's migrated data not to be received"). A local
variable would be collected before `'restored'` arrives.

**The route stack is flattened to one entry.** `getAllPathName()` returns the
whole stack and the code takes `allPaths[allPaths.length - 1]` - the top page
only. A three-deep stack arrives as root plus one page; the intermediate history
is not migrated. That matches the document's stated scope (栈顶页面, "the top
page") and suits detail-page scenarios, but it is a real limit to know about.

**`onContinue` may be async from API 18.** The sample relies on it - it `await`s
`distributedObject.save(...)` - and records the constraint in a comment:
`onContinue() 对于API version 18（不含18）之前版本仅支持同步调用` ("before API 18
onContinue supports synchronous calls only").

**Version compatibility is checked first.** `wantParam.version` is the peer's
version code; the sample compares it against its own `versionCode` from
`getBundleInfoForSelfSync` as a placeholder threshold and returns `MISMATCH` -
the documented way to refuse a migration for a reason other than failure.

## Implementation steps

1. **Mark the ability continuable** in `module.json5`: `"continuable": true`.
2. **On the source, in `onContinue`:**
   - Read `wantParam.version`, compare against your minimum supported version,
     `return AbilityConstant.OnContinueResult.MISMATCH` if too old (and tell the
     user why).
   - Get the top route: `pageStack.getAllPathName()`, take the last element, and
     `JSON.stringify(pageStack.getParamByName(name))`.
   - Either write the values into `wantParam`, or build a distributed data
     object, `genSessionId()`, `setSessionId()`, put the id in `wantParam`, and
     `await save(wantParam.targetDevice)`.
   - `return AbilityConstant.OnContinueResult.AGREE`.
3. **On the target, in `onCreate` and `onNewWant`:** return unless
   `launchParam.launchReason === LaunchReason.CONTINUATION`, then restore into
   `AppStorage` - synchronously from `want.parameters`, or from the
   `on('status')` callback when `status === 'restored'`. Keep the object in a
   field; release the listener when it is superseded (HW-19-0170).
4. **In `onWindowStageRestore`,** `windowStage.loadContent` the Navigation root.
5. **In the root page's `onPageShow`,** if a migrated path is present, push it -
   and **clear it** so it is consumed once (HW-19-0169).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The migration model and the channel enum

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
import { AbilityConstant, bundleManager, ConfigurationConstant, UIAbility, Want } from '@kit.AbilityKit';
import { promptAction, window } from '@kit.ArkUI';
import { distributedDataObject } from '@kit.ArkData';

export enum DATA_MIGRATE_TYPE {
  DISTRIBUTED_OBJECT_TYPE,
  WANT_TYPE
}

class PageModel {
  pagePath: string | undefined = '';
  pageParams: string | undefined = '';
  continueMsg: string | undefined = '';

  constructor(pagePath: string | undefined, pageParams: string | undefined, continueMsg: string | undefined) {
    this.pageParams = pageParams;
    this.pagePath = pagePath;
    this.continueMsg = continueMsg;
  }
}

export default class EntryAbility extends UIAbility {
  // 数据恢复时，分布式数据对象状态监听有延迟，需要在此处定义该对象，避免对象销毁导致未接收到对端迁移数据
  distributedObject?: distributedDataObject.DataObject;
```

All three `PageModel` fields are `| undefined` with defaults, because the target
side constructs an empty one as the shape for the incoming object:
`new PageModel(undefined, undefined, undefined)`.

### Saving state on the source

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
// 注意：onContinue() 对于API version 18（不含18）之前版本仅支持同步调用，从API version 18及后续版本可支持异步调用。
async onContinue(wantParam: Record<string, Object>) {
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onWindowStageRestore');   // FIX (HW-19-0172)
  try {
    const targetVersion = wantParam.version; // 获取迁移对端应用的版本号

    let bundleFlags = bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION;
    let bundleInfo = bundleManager.getBundleInfoForSelfSync(bundleFlags);
    // 应用可根据源端版本号设置支持接续的最小兼容版本号
    const versionThreshold: number = bundleInfo.versionCode; // 替换为应用自己支持兼容的最小版本号
    // 兼容性校验
    if (targetVersion < versionThreshold) {
      promptAction.openToast({
        message: '目标端应用版本号过低，不支持接续，请您升级应用版本后再试',
        duration: 2000
      });
      // 在兼容性校验不通过时返回MISMATCH
      return AbilityConstant.OnContinueResult.MISMATCH;
    }

    // 从NavPathStack中获取页面及参数信息
    let pageParams: string = '';
    let pagePath: string = '';
    let continueMsg: string = '迁移的数据';
    if (AppStorage.get<NavPathStack>('pageStack')) {
      let pageStack: NavPathStack = AppStorage.get<NavPathStack>('pageStack') as NavPathStack;
      let allPaths = pageStack.getAllPathName();
      if (allPaths && allPaths.length > 0) {
        let firstPath = allPaths[allPaths.length -1];      // FIX (HW-19-0172): this is the top
        pagePath = firstPath;
        pageParams = JSON.stringify(pageStack.getParamByName(firstPath));
      }
    }
    let dataMigrateType = AppStorage.get<number>('dataMigrateType');
    wantParam.dataMigrateType =
      dataMigrateType ? DATA_MIGRATE_TYPE.WANT_TYPE : DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE;  // FIX (HW-19-0171)
    if (dataMigrateType === DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE) {
      // 方案一：保存在分布式数据对象中，更适用于要迁移的数据较大（100KB以上）或需要迁移文件场景
      let pageModel = new PageModel(pagePath, pageParams, continueMsg);
      let distributedObject = distributedDataObject.create(this.context, pageModel);
      // 生成数据对象组网id，激活分布式数据对象
      let dataSessionId: string = distributedDataObject.genSessionId();
      // 注意：setSessionId接口在应用接续场景下无需申请ohos.permission.DISTRIBUTED_DATASYNC权限
      distributedObject.setSessionId(dataSessionId);
      // 将组网id存在want中传递到对端
      wantParam.dataSessionId = dataSessionId;
      // 数据对象持久化，确保迁移后即使应用退出，对端依然能够恢复数据对象
      // 从wantParam.targetDevice中获取到对端设备的networkId作为入参
      await distributedObject.save(wantParam.targetDevice as string).then((result:
        distributedDataObject.SaveSuccessResponse) => {
        hilog.info(DOMAIN, 'testTag', '%{public}s',
          `distributedObject.save success result: ${JSON.stringify(result)}`);
      }).catch((err: BusinessError) => {
        hilog.error(DOMAIN, 'testTag', '%{public}s', `distributedObject.save failed err: ${JSON.stringify(err)}`);
      });
    } else {
      // 方案二：保存在want中，更适用于轻量级数据
      wantParam.pagePath = pagePath;
      wantParam.pageParams = pageParams;
      wantParam.continueMsg = continueMsg;
    }
    return AbilityConstant.OnContinueResult.AGREE;
  } catch (e) {
    hilog.info(DOMAIN, 'testTag', '%{public}s', `onContinue failed, code: ${e.code}, message: ${e.message}`);
    return AbilityConstant.OnContinueResult.REJECT;
  }
}
```

Three return values, three meanings: `AGREE` proceed, `MISMATCH` refuse for a
compatibility reason the user should be told about, `REJECT` refuse because
something failed. Wrapping the whole body in `try/catch` and returning `REJECT`
is the right default - a throw out of `onContinue` would leave the handover in an
undefined state.

`wantParam.targetDevice` is supplied by the framework and is the peer's network
id - it is the only way to address `save()`.

### Restoring on the target

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
  try {
    this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);
  } catch (err) { ... }
  try {
    // 恢复流转数据
    this.restoreData(want, launchParam);
  } catch (e) {
    hilog.error(DOMAIN, 'testTag', '%{public}s', `restoreData failed, code: ${e.code}, message: ${e.message}`);
  }
}

onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
  try {
    this.restoreData(want, launchParam);
  } catch (e) { ... }
}

// 恢复流转数据
restoreData(want: Want, _launchParam: AbilityConstant.LaunchParam) {
  if (_launchParam.launchReason !== AbilityConstant.LaunchReason.CONTINUATION) {
    return;
  }
  // 获取对端数据迁移方式
  let dataMigrateType = DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE;
  if (want.parameters !== undefined) {
    dataMigrateType = want.parameters.dataMigrateType as number;
  }
  AppStorage.setOrCreate('dataMigrateType', dataMigrateType);
  if (dataMigrateType === DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE) {
    // 方案一：从分布式数据对象中获取页面数据及参数
    let remotePageModel = new PageModel(undefined, undefined, undefined);
    this.distributedObject = distributedDataObject.create(this.context, remotePageModel);
    // 读取分布式数据对象组网id
    let dataSessionId = '';
    if (want.parameters !== undefined) {
      dataSessionId = want.parameters.dataSessionId as string;
    }

    // 添加数据变更监听
    this.distributedObject.on('status',                          // FIX (HW-19-0170): no off()
      (sessionId: string, networkId: string, status: 'online' | 'offline' | 'restored') => {
        if (status === 'restored') {
          if (this.distributedObject) {
            // 收到迁移恢复的状态时，可以从分布式数据对象中读取数据
            AppStorage.setOrCreate('pagePath', this.distributedObject['pagePath']);
            AppStorage.setOrCreate('pageParams', this.distributedObject['pageParams']);
            AppStorage.setOrCreate('continueMsg', this.distributedObject['continueMsg']);
          }
        }
      });

    // 激活分布式数据对象
    this.distributedObject.setSessionId(dataSessionId);
  } else {
    // 方案二：从want中获取页面数据及参数
    if (want.parameters !== undefined) {
      let pagePath = want.parameters.pagePath as string;
      let pageParams = want.parameters.pageParams as string;
      let continueMsg = want.parameters.continueMsg as string;
      AppStorage.setOrCreate('pagePath', pagePath);
      AppStorage.setOrCreate('pageParams', pageParams);
      AppStorage.setOrCreate('continueMsg', continueMsg);
    }
  }
}
```

Note the ordering in the distributed-object branch: **register `on('status')`
before `setSessionId`**. Activating the session is what triggers the sync, so a
listener attached afterwards can miss the `'restored'` event.

The listener is attached to the object, but the values are read through
`this.distributedObject['pagePath']` rather than from the callback's arguments -
the object is the source of truth, and the event only says when it is ready.

### Loading the root page after a restore

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onWindowStageRestore(windowStage: window.WindowStage): void {
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onWindowStageRestore');
  windowStage.loadContent('pages/Index', (err) => {
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
    hilog.info(DOMAIN, 'testTag', 'Succeeded in loading the content.');
  });
}
```

Identical to `onWindowStageCreate` - the migrated device still starts at the
Navigation root, and the route is rebuilt on top of it by the page.

### Rebuilding the route

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/pages/Index.ets`

```ts
import { DATA_MIGRATE_TYPE } from '../entryability/EntryAbility';

@Entry
@Component
struct Index {
  // 0:方式一，通过分布式数据对象；1：方案二，通过want
  @StorageProp('dataMigrateType') dataMigrateType: DATA_MIGRATE_TYPE = DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE;
  @StorageLink('pageStack') pageStack: NavPathStack = new NavPathStack();
  @StorageProp('pagePath') pagePath: string = '';
  @StorageProp('pageParams') pageParams: string = '';
  @StorageProp('continueMsg') continueMsg: string = 'Hello world!';

  onPageShow(): void {
    console.info(`onPageShow, pagePath: ${this.pagePath}, pageParams:${this.pageParams}`);
    if (this.pagePath) {
      this.pageStack.pushPathByName(this.pagePath, this.pageParams);   // FIX (HW-19-0169)
    }
  }
```

`@StorageLink('pageStack')` is the key line: the stack lives in `AppStorage`, not
in the component, which is how `onContinue` in the ability can reach it. The
migrated fields are `@StorageProp` (one-way) because the ability writes them and
the page only reads.

The corrected handler:

```ts
onPageShow(): void {
  if (this.pagePath) {
    const path = this.pagePath;
    const params = this.pageParams;
    AppStorage.setOrCreate('pagePath', '');
    AppStorage.setOrCreate('pageParams', '');
    this.pageStack.pushPathByName(path, params);
  }
}
```

### A destination reclaiming the stack

`NavigationContinue.zip#NavigationContinue/entry/src/main/ets/views/PageOne.ets`

```ts
NavDestination() {
  // ...
}
.title(`Navigation页面1`)
.onReady((ctx: NavDestinationContext) => {
  // NavDestinationContext获取当前所在的导航控制器
  this.pageStack = ctx.pathStack;
});
```

Taking the stack from `NavDestinationContext` rather than assuming a shared
reference is what keeps a destination usable after a continuation, when the stack
it was pushed onto is a different instance from the one it was built with.

### The route map

`NavigationContinue.zip#NavigationContinue/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "pageOne",
      "pageSourceFile": "src/main/ets/views/PageOne.ets",
      "buildFunction": "PageOneBuilder",
      "data": { "description": "this is pageOne" }
    },
    {
      "name": "pageTwo",
      "pageSourceFile": "src/main/ets/views/PageTwo.ets",
      "buildFunction": "PageTwoBuilder",
      "data": { "description": "this is pageTwo" }
    }
  ]
}
```

The `name` values are exactly what crosses to the other device, which is why a
`routerMap` is a prerequisite for this pattern: the target resolves the name to a
builder without the source having to say anything about files or components.

## Permissions & config

**No permissions are declared and none are needed.** The sample states the reason
twice in comments: `setSessionId` does not require
`ohos.permission.DISTRIBUTED_DATASYNC` in the continuation scenario, only in
other uses of distributed data objects. Continuation itself is gated by the user
enabling it in system settings and by both devices being signed into the same
account, not by an application permission.

`NavigationContinue.zip#NavigationContinue/entry/src/main/module.json5` -
the one line that enables the feature:

```json5
"continuable": true,
```

The document's 说明 section lists the runtime prerequisites: two devices signed
into the same Huawei account, WLAN and Bluetooth on (or 多设备协同增强服务
enabled), and 多设备协同 > 接续 turned on in Settings.

`build-profile.json5` pins `6.0.0(20)`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `onContinue` supports async only
  from API 18.
- **A `NavPathStack` cannot be serialised.** Only route names and parameters
  cross; the target rebuilds.
- **This sample migrates the top route only.** Deeper history is lost.
- **Parameters cross as a string.** `JSON.stringify` on the way out, and the
  restored push passes that string where the original push passed a typed value -
  the destination must accept both shapes.
- **`on('status')` must be registered before `setSessionId`**, or the `'restored'`
  event can be missed.
- **The distributed object must outlive the restore.** Hold it in a field, not a
  local.
- **`save(targetDevice)` needs the peer's network id** from
  `wantParam.targetDevice`.
- **Both devices must be on the same account** with continuation enabled in
  Settings.
- **`OnContinueResult.MISMATCH` is for version incompatibility**; `REJECT` is for
  failure.

## Pitfalls

- **`pagePath` is never cleared after being consumed, which is incorrect** -
  `onPageShow` fires on every return to the root, so the migrated destination is
  pushed again each time and the home page becomes unreachable. (HW-19-0169)
- **`distributedObject.on('status')` has no matching `off`, which is incorrect** -
  `restoreData` runs from both `onCreate` and `onNewWant`, so repeated
  continuations leave live listeners that can overwrite the current migration's
  values with an earlier session's. (HW-19-0170)
- **The migration type written into the want comes from a ternary that maps each
  value to itself while the branch below tests the raw value, which is
  incorrect** - the two can disagree when the key is unset, announcing one channel
  and using the other. (HW-19-0171)
- **`onContinue` logs `'Ability onWindowStageRestore'`, `firstPath` holds the last
  element, and the project tree writes `entrybackupablility` - all incorrect.**
  (HW-19-0172)
- **Do not assume `onCreate` is the only restore path.** A running application
  receives the continuation through `onNewWant`; handling only one of the two
  works until the user has the app already open on the target.
- **Do not use the source's own `versionCode` as the compatibility threshold in
  production.** The sample marks it as a placeholder
  (`替换为应用自己支持兼容的最小版本号`); as written it refuses any target older
  than the source.
- **Do not put large payloads in `wantParam`.** That is what the distributed
  object branch is for - the sample's own guidance is 100 KB.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-uiability -
  `onContinue`, `onNewWant`, `onWindowStageRestore`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-abilityconstant -
  `OnContinueResult` (`AGREE` / `REJECT` / `MISMATCH`),
  `LaunchReason.CONTINUATION`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-data-distributedobject -
  `create`, `genSessionId`, `setSessionId`, `on('status')`, `save`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-navigation-navigation -
  `NavPathStack`, `getAllPathName`, `getParamByName`, `pushPathByName`,
  `NavDestinationContext`, the `routerMap` profile.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-want -
  `Want.parameters`, and `targetDevice` in the continuation `wantParam`.
- https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-continue-cast -
  the application-continuation development guide the document builds on.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
