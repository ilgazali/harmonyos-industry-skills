---
id: COMMON-49
title: PC system tray - keeping an application alive in the background through a status-bar item
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PCStatusBar.zip
kits: ["@kit.DeskTopExtensionKit", "@kit.AbilityKit", "@kit.ImageKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: ["statusBarManager.addToStatusBar", "statusBarManager.removeFromStatusBar", "statusBarManager.on('statusBarIconClick')", StatusBarItem, StatusBarIcon, QuickOperation, StatusBarMenuItem, StatusBarSubMenuItem, StatusBarGroupMenu, "contextConstant.ProcessMode.NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM", "contextConstant.StartupVisibility.STARTUP_HIDE", StartOptions, "UIAbilityContext.startAbility", "UIAbilityContext.showAbility", "UIAbilityContext.hideAbility", "UIAbilityContext.terminateSelf", "UIAbility.onPrepareToTerminate", "commonEventManager.publish", "commonEventManager.createSubscriber", "commonEventManager.subscribe", "resourceManager.getRawFileContentSync", "image.createImageSource", "ImageSource.createPixelMap"]
permissions: ["ohos.permission.PREPARE_APP_TERMINATE"]
min_api: 17
modules: [entry]
findings: [HW-19-0163, HW-19-0164, HW-19-0165, HW-19-0166, HW-19-0167, HW-19-0168, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a **HarmonyOS PC (2in1)** application must keep running after
its window is closed - a sync agent, a security token daemon, a monitoring tool.

The platform position is stated plainly in the document:
`HarmonyOS PC上不允许后台私自运行程序` ("HarmonyOS PC does not permit programs to
run in the background on their own"). The sanctioned route is the system tray: an
application that owns a status-bar item may attach a process to it, and that
process is allowed to live. Background survival is therefore **a consequence of
having a visible tray icon**, not a permission you request - the user can always
see that the application is there and remove it.

## Feature checklist

- Two tray icons (light and dark) in `resources/rawfile`, decoded to `PixelMap`
  (HW-19-0167).
- `statusBarManager.addToStatusBar(context, statusBarItem)` - and fail loudly if
  it fails (HW-19-0165).
- A second `UIAbility` started with
  `ProcessMode.NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM` and
  `StartupVisibility.STARTUP_HIDE`.
- `on('statusBarIconClick')` → `showAbility()`, with a matching `off`
  (HW-19-0164).
- `onPrepareToTerminate()` returning `true` in the UI ability, so closing the
  window hides it instead.
- A shutdown handshake so both abilities die together - restricted to this bundle
  (HW-19-0163).
- `ohos.permission.PREPARE_APP_TERMINATE`.

## Architecture

Single-module project (`entry` HAP), `deviceTypes: ["2in1"]` only:

| File | Responsibility |
| --- | --- |
| `utils/StatusBarUtil.ets` | `getPixelMap`, `loadStatusBar`, `holdStatusBar` |
| `entryability/EntryAbility.ets` | the UI ability; tray click handler, shutdown subscriber, `onPrepareToTerminate` |
| `backgroundability/BackGroundAbility.ets` | the process that is attached to the tray item; publishes the shutdown event |
| `pages/Index.ets` | a placeholder page |
| `resources/rawfile/{white,black}.png` | the tray icons |

**Two abilities, two jobs.** `EntryAbility` owns the window and the tray item.
`BackGroundAbility` exists only to be a process that the tray item keeps alive -
it has no window, is started hidden, and its single method is
`onPrepareToTerminate`.

**`NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM` is the whole mechanism.** Passed as
`StartOptions.processMode`, it tells the system to create a new process, start
the ability there, and **bind that process to the status-bar icon**. Combined
with `StartupVisibility.STARTUP_HIDE` - documented in the sample's own comment as
`目标Ability启动后，进入隐藏状态。不会调用Ability的onForeground生命周期`
("after the target ability starts it enters the hidden state; the ability's
onForeground lifecycle is not called") - the result is an invisible process whose
lifetime is the tray icon's lifetime.

**Icons are `PixelMap`s, not resource references.** `StatusBarIcon` takes
`white` and `black` `PixelMap`s - one per system theme - so the raw PNGs are read
from `rawfile`, wrapped in an `ImageSource`, and decoded. That is why they live in
`rawfile` rather than `media`: the code needs the bytes, not a `$r()` handle.

**`QuickOperation.abilityName: ''` is deliberate.** The sample's comment explains:
`此处为空，是由于点击左键无须弹出页面，想要直接showability` ("left over empty
because a left click should not pop up a page - we want to call showAbility
directly"). An empty ability name suppresses the default popup so the
`statusBarIconClick` handler can decide what happens.

**The closing dance is the subtle part**, and it is why this sample has three
interacting pieces:

1. `EntryAbility.onPrepareToTerminate()` calls `hideAbility()` and returns
   `true`. Returning `true` **cancels** the termination, so closing the window
   hides the application instead of ending it - which is what lets the tray icon
   bring it back.
2. But that also means Dock-quit or tray-quit destroys `BackGroundAbility` while
   `EntryAbility` refuses to die. The document names this problem in step 8.
3. So `BackGroundAbility.onPrepareToTerminate()` publishes a common event and
   returns `false` (allowing itself to die), and `EntryAbility` subscribes, and on
   receipt removes the tray item and calls `terminateSelf()`.

That handshake is correct in shape. Its two problems are that the channel is open
to the whole device (HW-19-0163) and that the subscription is never released
(HW-19-0164).

**`globalContext` is a module-level mutable** assigned in `onCreate`, because the
tray click handler is a free function outside the ability class. It is the reason
a stale registration is dangerous rather than merely untidy.

## Implementation steps

1. **Put two 24×24 icons in `resources/rawfile`** - one for a light tray, one for
   a dark one (HW-19-0167).
2. **Decode them to `PixelMap`**: `getRawFileContentSync(name)` →
   `image.createImageSource(fileData.buffer)` → `createPixelMap()`.
3. **Build the `StatusBarItem`**: `icons`, and a `QuickOperation` with an empty
   `abilityName` if you want to handle the left click yourself. Add
   `statusBarGroupMenu` if you want a right-click menu.
4. **`addToStatusBar(context, statusBarItem)`** inside `try/catch`, and propagate
   a failure to the caller (HW-19-0165).
5. **Declare a second `UIAbility`** in `module.json5` with no home skill.
6. **Start it** with `startAbility(want, { processMode:
   NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM, startupVisibility: STARTUP_HIDE })`.
7. **Register the click handler** with `statusBarManager.on('statusBarIconClick',
   fn)` and call `showAbility()` on `leftClick`. Release it in `onDestroy`
   (HW-19-0164).
8. **Return `true` from `EntryAbility.onPrepareToTerminate()`** after calling
   `hideAbility()`.
9. **Wire the shutdown handshake**: publish from `BackGroundAbility`, subscribe in
   `EntryAbility`, `removeFromStatusBar` then `terminateSelf`. Restrict both ends
   to this bundle (HW-19-0163).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Decoding a rawfile PNG into a PixelMap

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/utils/StatusBarUtil.ets`

```ts
import { common, contextConstant, StartOptions, Want } from '@kit.AbilityKit';
import { image } from '@kit.ImageKit';
import { statusBarManager } from '@kit.DeskTopExtensionKit';

const getPixelMap = async (name: string, context: common.UIAbilityContext) => {
  // 获取resourceManager资源管理器
  const resourceMgr = context.resourceManager;
  // 获取rawfile文件夹下icon.png的ArrayBuffer
  const fileData = resourceMgr.getRawFileContentSync(name);
  // 获取图片的ArrayBuffer
  const buffer = fileData.buffer;
  // 创建imageSource
  const imageSource = image.createImageSource(buffer);
  // 创建PixelMap
  let pixMap = await imageSource.createPixelMap();
  return Promise.resolve(pixMap);
};
```

Note the name passed in is `'white.png'` / `'black.png'` - resolved from the
`rawfile` root, which is where the files actually are.

### Building and adding the status-bar item

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/utils/StatusBarUtil.ets`

```ts
export const loadStatusBar = async (context: common.UIAbilityContext): Promise<void> => {
  // 设置系统托盘图标信息
  let icon: statusBarManager.StatusBarIcon = {
    white: await getPixelMap('white.png', context),
    black: await getPixelMap('black.png', context),
  };

  //构建添加到状态栏的图标详细信息
  const operation: statusBarManager.QuickOperation = {
    abilityName: '', //此处为空，是由于点击左键无须弹出页面，想要直接showability
    title: 'backgroundWindowShow', // 左键弹窗标题
    height: 30, // 左键弹窗高度
    moduleName: 'entry' // 可缺省,但不建议，缺省后hover托盘时不会显示abilityName
  };

  //构建右键菜单项内容
  let subMenus: Array<statusBarManager.StatusBarSubMenuItem> = [];
  let subMenuItemAction: statusBarManager.StatusBarMenuAction = {
    abilityName: 'EntryAbility',
    moduleName: 'entry',
    menuCode: 'sub1',
    notifyOnly: true
  };
  let subMenu: statusBarManager.StatusBarSubMenuItem = {
    subTitle: '子菜单项',
    menuAction: subMenuItemAction
  };
  subMenus.push(subMenu);

  let statusBarMenuItems: Array<statusBarManager.StatusBarMenuItem> = [];
  let menuItem: statusBarManager.StatusBarMenuItem = {
    title: '一级菜单项',
    // 一级menuAction和subMenu两项不可都缺省
    subMenu: subMenus
  };
  statusBarMenuItems.push(menuItem);

  let statusBarGroupMenus: Array<statusBarManager.StatusBarGroupMenu> = [];
  statusBarGroupMenus.push(statusBarMenuItems);

  //构建状态栏信息
  const statusBarItem: statusBarManager.StatusBarItem = {
    icons: icon, // 指定图标
    quickOperation: operation, // 指定启动参数
    // statusBarGroupMenu: statusBarGroupMenus //可选，右键菜单项   FIX (HW-19-0167): dead code above
  };
  try {
    statusBarManager.addToStatusBar(context, statusBarItem); // 调用 addToStatusBar 设置系统托盘
    hilog.info(0, 'testTag', 'addToStatusBar success');
    Promise.resolve();                                        // FIX (HW-19-0165): discarded
  } catch (error) {
    hilog.error(0, 'testTag', `addToStatusBar failed. error code: ${error.code}, error message: ${error.message}`);
  }                                                           // FIX (HW-19-0165): swallowed
};
```

The menu structure is worth keeping even though it is not wired up - it shows the
three nesting levels the API expects: `StatusBarGroupMenu` is an array of
`StatusBarMenuItem`, each of which has either a `menuAction` or a `subMenu` of
`StatusBarSubMenuItem`, and the sample's own comment records the rule
`一级menuAction和subMenu两项不可都缺省` ("the top-level menuAction and subMenu
cannot both be omitted"). `notifyOnly: true` means selecting the item notifies the
ability rather than starting it.

### Attaching a process to the tray item

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/utils/StatusBarUtil.ets`

```ts
export const holdStatusBar = (context: common.UIAbilityContext | common.UIExtensionContext): Promise<void> => {
  return new Promise((resolve, reject) => {
    let want: Want = {
      bundleName: 'com.example.pcstatusbar', // 指定当前应用名
      abilityName: 'BackGroundAbility' // 指定应用的Ability
    };
    let options: StartOptions = {
      // 创建一个新进程，在该进程上启动Ability，并绑定该进程到状态栏图标上
      processMode: contextConstant.ProcessMode.NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM,
      // 目标Ability启动后，进入隐藏状态。不会调用Ability的onForeground生命周期
      startupVisibility: contextConstant.StartupVisibility.STARTUP_HIDE
    };
    context.startAbility(want, options, (err) => {
      if (err.code) {
        hilog.error(0, 'testTag', `startError:: ${JSON.stringify(err)}}`);
        reject();                    // FIX (HW-19-0166): rejects with no reason
      } else {
        hilog.info(0, 'testTag', `backgroundAbility Success`);
        resolve();
      }
    });
  });
};
```

The bundle name is hard-coded rather than read from the context - fine for a
sample, a rename hazard in a real project.

### The tray click handler

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
import { common, ConfigurationConstant, UIAbility } from '@kit.AbilityKit';
import { statusBarManager } from '@kit.DeskTopExtensionKit';
import { loadStatusBar, holdStatusBar } from '../utils/StatusBarUtil';
import { emitter, BusinessError, commonEventManager } from '@kit.BasicServicesKit';

let globalContext: common.UIAbilityContext;

export function myShowUiAbility() {
  if (globalContext) {
    globalContext.showAbility().then(() => {
      hilog.info(0x0000, 'testTag', 'Succeeded in myShowUiAbility.');
    }).catch((err: Error) => {
      hilog.error(0x0000, 'testTag', 'Failed to load myShowUiAbility. Cause: %{public}s', JSON.stringify(err) ?? '');
    });
  }
}

export const onStatusBarIconClick = (eventData: emitter.EventData) => {
  // 自定义图标点击业务
  let data = eventData.data;
  if (data) {
    switch (data.iconClickType) {
      case 'leftClick':
        myShowUiAbility(); //点击托盘图标，showAbility
        break;
      default:
        break;
    }
  }
};
```

The event arrives as an `emitter.EventData` whose `data.iconClickType` is
`'leftClick'` - the `switch` with a `default` is the right shape for an API that
may add click types later.

### Start-up

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onCreate(): void {
  try {
    this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);
  } catch (err) {
    hilog.error(DOMAIN, 'testTag', 'Failed to set colorMode. Cause: %{public}s', JSON.stringify(err));
  }
  globalContext = this.context;
  //加载系统托盘
  loadStatusBar(this.context).then(() => {
    setTimeout(() => {                                   // FIX (HW-19-0166)
      //注册状态栏图标左键点击事件
      statusBarManager.on('statusBarIconClick', onStatusBarIconClick);   // FIX (HW-19-0164)
      setTimeout(() => {
        //维系系统托盘
        holdStatusBar(this.context).then(() => {         // FIX (HW-19-0166): no .catch
          setInterval(() => {
            console.log('托盘启动中。。。');               // FIX (HW-19-0166): never cleared
          }, 1000);
        });
      }, 200);
    }, 500);
  });
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onCreate');
}
```

The corrected shape:

```ts
async onCreate(): Promise<void> {
  globalContext = this.context;
  try {
    await loadStatusBar(this.context);
    statusBarManager.on('statusBarIconClick', onStatusBarIconClick);
    await holdStatusBar(this.context);
  } catch (err) {
    hilog.error(DOMAIN, 'testTag', `status bar setup failed: ${JSON.stringify(err)}`);
  }
}
```

### Refusing to terminate

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onPrepareToTerminate(): boolean {
  hilog.info(0x0000, 'testTag', '%{public}s', 'EntryAbility onPrepareToTerminate');
  this.context.hideAbility().then(() => {
    hilog.info(0, 'testTag', 'entry hideAbility success');
  }).catch((err: BusinessError) => {
    hilog.error(0, 'testTag', `hideAbility fail, err: ${JSON.stringify(err)}`);
  });
  return true;     // true cancels the termination
}
```

`return true` is the operative line and the reason
`ohos.permission.PREPARE_APP_TERMINATE` is required.

### The shutdown handshake

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/backgroundability/BackGroundAbility.ets`

```ts
export default class BackGroundAbility extends UIAbility {
  onPrepareToTerminate(): boolean {
    // 公共事件相关信息
    let options: commonEventManager.CommonEventPublishData = {
      code: 1, // 公共事件的初始代码
      data: 'initial data', // 公共事件的初始数据
    };                                          // FIX (HW-19-0163): no bundleName
    // 发布BackgroundAbility onPrepareToTerminate公共事件
    commonEventManager.publish('BackgroundAbility_onPrepareToTerminate_event', options, (err) => {
      if (err) {
        hilog.error(0x0000, 'testTag',
          `Failed to publish common event. Code is ${err.code}, message is ${err.message}`);
      } else {
        hilog.info(0x0000, 'testTag', `Succeeded in publishing common event.`);
      }
    });
    return false;    // false allows this ability to terminate
  }
};
```

`PCStatusBar.zip#PCStatusBar/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
// 用于保存创建成功的订阅者对象，后续使用其完成订阅及退订的动作
let subscriber: commonEventManager.CommonEventSubscriber | null = null;
let subscribeInfo: commonEventManager.CommonEventSubscribeInfo = {
  events: ['BackgroundAbility_onPrepareToTerminate_event'],   // FIX (HW-19-0163): no publisherBundleName
};
commonEventManager.createSubscriber(subscribeInfo,            // FIX (HW-19-0164): module scope
  (err: BusinessError, data: commonEventManager.CommonEventSubscriber) => {
    if (err) {
      hilog.error(0, 'testTag', `Failed to create subscriber. Code is ${err.code}, message is ${err.message}`);
      return;
    }
    subscriber = data;
    if (subscriber !== null) {
      commonEventManager.subscribe(subscriber, (err: BusinessError) => {
        if (err) {
          hilog.error(0, 'testTag', `Failed to subscribe common event. Code is ${err.code}, message is ${err.message}`);
          return;
        }
        statusBarManager.removeFromStatusBar(globalContext); //移除托盘
        setTimeout(() => {
          globalContext.terminateSelf(); //订阅BackgroundAbility onPrepareToTerminate公共事件后，主动关闭EntryAbility
        }, 200);
      });
    } else {
      hilog.error(0, 'testTag', `Need create subscriber`);
    }
  });
```

Note the ordering: `removeFromStatusBar` **before** `terminateSelf`. Terminating
first would leave an orphaned tray icon.

## Permissions & config

`PCStatusBar.zip#PCStatusBar/entry/src/main/module.json5`:

```json5
"deviceTypes": ["2in1"],
"requestPermissions": [
  { "name": "ohos.permission.PREPARE_APP_TERMINATE" }
],
"abilities": [
  { "name": "EntryAbility",     "srcEntry": "./ets/entryability/EntryAbility.ets", ... },
  { "name": "BackGroundAbility", "srcEntry": "./ets/backgroundability/BackGroundAbility.ets", ... }
]
```

- **`ohos.permission.PREPARE_APP_TERMINATE`** - required for
  `onPrepareToTerminate` to be called at all, which is the hook this whole
  design turns on. Correctly declared and correctly documented.
- **`deviceTypes: ["2in1"]`** only - the tray is a PC concept and the sample does
  not claim phone or tablet support.
- **No permission is needed for the status bar itself.** Owning a tray item is
  what grants the background lifetime.

`build-profile.json5` sets `compatibleSdkVersion: "5.0.5(17)"` and
`targetSdkVersion: "6.0.0(20)"`, which matches the document's 约束与限制
(API 17 / HarmonyOS 5.0.5 SDK / DevEco Studio 6.0.0) - the split floor is
deliberate, not an inconsistency.

## Constraints

- **API level.** API Version 17 Release or later, HarmonyOS 5.0.5 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **PC (2in1) only.** `statusBarManager` comes from `@kit.DeskTopExtensionKit`.
- **HarmonyOS PC does not allow unsanctioned background execution.** The tray item
  is the sanctioned route, and it is user-visible by design.
- **`StatusBarIcon` needs `PixelMap`s**, so icons must be decoded from `rawfile`
  rather than referenced with `$r()`.
- **`onPrepareToTerminate` requires `ohos.permission.PREPARE_APP_TERMINATE`**, and
  returning `true` cancels the termination.
- **Returning `true` forever means the application can never close by the normal
  route** - a deliberate termination path is mandatory, which is what step 8
  exists to provide.
- **`removeFromStatusBar` must precede `terminateSelf`**, or the icon outlives the
  process.
- **`NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM` requires a status-bar item to attach
  to**, so `addToStatusBar` must have succeeded first.

## Pitfalls

- **The shutdown common event is unrestricted in both directions, which is
  incorrect** - any application that publishes
  `BackgroundAbility_onPrepareToTerminate_event` makes this one drop its tray icon
  and terminate. Set `publisherBundleName` and `bundleName`, or use an in-process
  channel. (HW-19-0163)
- **The subscriber is created at module scope and never unsubscribed, and
  `on('statusBarIconClick')` has no matching `off` - both incorrect.** The click
  handler acts on a module-level `globalContext` that outlives the ability that
  set it. (HW-19-0164)
- **`loadStatusBar` swallows an `addToStatusBar` failure and resolves anyway,
  which is incorrect** - the click listener, the background process and the
  interval all start against a tray item that does not exist. The stray
  `Promise.resolve();` statement is discarded. (HW-19-0165)
- **Start-up is sequenced by nested 500 ms / 200 ms delays inside a promise that
  already sequenced it, `holdStatusBar` rejects with no reason and no handler, and
  a one-second log interval is never cleared - all incorrect.** (HW-19-0166)
- **Step 1 says `rawfile/resources` while the icons are in `resources/rawfile`,
  and step 2's twenty-line right-click menu is commented out of the
  `StatusBarItem` - both incorrect.** (HW-19-0167)
- **The 工程目录 tree writes `entrybackupablility` and gives `pages` and `utils`
  the same last-child marker, which is incorrect** - the same template defect as
  COMMON-46. (HW-19-0168)
- **Do not return `true` from `onPrepareToTerminate` without building the exit
  path.** The document's step 8 is not optional; without it the application cannot
  be quit at all.
- **Do not assume the tray icon survives a crash.** It is bound to the process; if
  the attached process dies the item goes with it.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/statusbar-extension-manager -
  `addToStatusBar`, `removeFromStatusBar`, `on('statusBarIconClick')`,
  `StatusBarItem`, `StatusBarIcon`, `QuickOperation`, the menu types.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext -
  `startAbility` with `StartOptions`, `showAbility`, `hideAbility`,
  `terminateSelf`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-uiability -
  `onPrepareToTerminate` and the meaning of its return value.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-contextconstant -
  `ProcessMode.NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM`,
  `StartupVisibility.STARTUP_HIDE`.
- `documentation/harmonyos-references/03_system/js-apis-commoneventmanager.md` -
  `publish`, `createSubscriber`, `subscribe`, `unsubscribe`, and the
  `bundleName` / `publisherBundleName` restrictions.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-commoneventmanager
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all -
  `ohos.permission.PREPARE_APP_TERMINATE`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
