---
id: COMMON-18
title: Load-time and call-count instrumentation - AOP hooks around a download method with per-page persisted totals
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
sample: huawei_industry_tree/19_common_technical_solutions/downloads/CodeInstrumentation.zip
kits: ["@kit.ArkTS", "@kit.NetworkKit", "@kit.ArkData", "@kit.ImageKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["util.Aspect.addBefore", "util.Aspect.addAfter", "http.createHttp", "HttpRequest.request", "HttpRequest.destroy", "http.ResponseCode", "image.createImageSource", "preferences.getPreferencesSync", "Preferences.putSync", "Preferences.flushSync", "Preferences.getSync", "Preferences.deleteSync", "fileIo.openSync", "fileIo.writeSync", "fileIo.closeSync", "fileUri.getUriFromPath", Navigation, NavPathStack, NavDestination, routerMap, "@Consume", "@Provide", "@StorageLink"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0045, HW-19-0046, HW-19-0047, HW-19-0048, HW-19-0049, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when you need **per-feature timing and call counts without
touching the feature's own code** - the classic "how long does image loading take
on each channel page, and how often does it run" question a site owner asks.

The technique is AOP code instrumentation through `util.Aspect`: hook a method of
a class from outside it, record a timestamp before and after, and accumulate the
result. The instrumented code is untouched.

**Read the findings before reusing the timing logic.** As shipped, the hook stops
the clock when the async download method returns its Promise, not when the
download completes (HW-19-0045), so the numbers the sample displays do not
measure what the document says they measure.

## Feature checklist

The application must:

- Provide a download method on a class - `Aspect` hooks class methods, not free
  functions.
- Hook it with `util.Aspect.addBefore` to record the start time.
- Hook it with `util.Aspect.addAfter` to record the end time, increment the call
  count, and **return the original return value** (HW-19-0046).
- Measure the time the asynchronous work actually takes, not the time to create
  its Promise (HW-19-0045).
- Accumulate rather than overwrite the counters (HW-19-0047).
- Persist the per-page totals with `Preferences` and read them back on the
  statistics page.
- Release each `HttpRequest` after use (HW-19-0049).
- Declare `ohos.permission.INTERNET`.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes the `UIAbilityContext` into `GlobalContext`, window setup, loads `pages/MainPage` |
| `utils/GlobalContext.ets` | singleton holding the ability context so non-component code can reach `filesDir` |
| `utils/DownloadUtils.ets` | `downloadNetworkImage(url)` plus **four identical wrapper classes** `DownloadImage`, `DownloadImage1`, `DownloadImage2`, `DownloadImage3` |
| `utils/PhotoUtil.ets` | `PhotoAlbumsUtil.createPixelImage(buffer)` |
| `utils/PreferenceUtils.ets` | `PreferencesClass` - `getStore`, `savePreferenceInfo`, `getPreferenceInfo`, `deletePreferenceInfo` |
| `utils/Logger.ets` | `hilog` wrapper |
| `model/ListOptionUtils.ets` | `LstOptionClass.countTimesInfo[]` and `delayTimeInfo[]` - the preference keys, one pair per channel |
| `model/CardInfo.ets`, `model/Tab.ets`, `common/Constants.ets` | channel metadata and `CARD_INFOS` |
| `component/ReUsePage.ets` | the news card that renders the downloaded `PixelMap` |
| `pages/MainPage.ets` | channel list, owns the `NavPathStack` and the two `@Provide` counter arrays |
| `pages/StatisticPage.ets`, `SciPage.ets`, `SportsPage.ets`, `FunPage.ets` | the four channel pages, each with its own instrumentation |
| `pages/SummaryPage.ets` | the statistics screen reading the persisted totals |

**Why four identical classes.** `util.Aspect.addBefore/addAfter` attach to a
*class object* and a method name. To time four channels separately, the sample
gives each channel its own class - `DownloadImage`..`DownloadImage3` - all
delegating to the same `downloadNetworkImage`. Hooking one shared class would mix
the four channels' measurements into one counter.

**Data flow per channel page.**

1. At module load, `addTimePrinterN(DownloadImageN, 'instanceMethod', false)`
   installs the two hooks on that channel's class. Module-level `beginN`, `endN`,
   `countN`, `requiredN` hold the in-flight measurement.
2. `aboutToAppear` calls `new DownloadImageN().instanceMethod(url)`.
3. `addBefore` stamps `beginN`; `addAfter` stamps `endN`, sets `countN`,
   computes `requiredN`, and returns the original result.
4. The page's `.then()` reads the persisted totals with
   `PreferencesClass.getPreferenceInfo`, adds the fresh `countN` / `requiredN`,
   and writes them back with `savePreferenceInfo`.
5. `aboutToDisappear` copies the persisted values into the `@Consume`
   `countTimesArr` / `delayTimeArr` shared with `SummaryPage`.
6. `SummaryPage.aboutToAppear` re-reads all four pairs from `Preferences` and
   renders them.

Note the two different parameter orders in `PreferencesClass`:
`savePreferenceInfo(context, storeKey, storeName, value)` but
`getPreferenceInfo(context, storeName, storeKey)`. Both call sites are correct
for their own signature, but the asymmetry is a trap when adding new call sites.

## Implementation steps

1. **Put the work in a class method.** `Aspect` targets `(targetClass,
   methodName, isStatic)`; a free function cannot be hooked.
2. **One class per measurement scope.** If several call sites must be measured
   separately, give each its own class.
3. **Install the hooks at module scope**, so they are in place before the first
   call:
   ```ts
   function addTimePrinter(targetClass: Object, methodName: string, isStatic: boolean) {
     util.Aspect.addBefore(targetClass, methodName, isStatic, () => { begin = Date.now(); });
     util.Aspect.addAfter(targetClass, methodName, isStatic, (instance: Object, ret: Object) => {
       // stop the clock where the work really ends - see HW-19-0045
       return ret;   // mandatory - see HW-19-0046
     });
   }
   addTimePrinter(DownloadImage, 'instanceMethod', false);
   ```
4. **Handle the asynchrony.** Because the method is `async`, `addAfter` receives
   a Promise. Wrap it rather than timing its creation:
   ```ts
   util.Aspect.addAfter(targetClass, methodName, isStatic,
     (instance: Object, ret: Promise<image.PixelMap | undefined>) => {
       return ret.then((v) => {
         required += Date.now() - begin;
         count += 1;
         return v;
       });
     });
   ```
5. **Accumulate, do not assign** (HW-19-0047), and reset the in-flight counters
   once they have been folded into the persisted totals.
6. **Persist with `Preferences`**: `getPreferencesSync(context, { name })`,
   `putSync(key, value)`, `flushSync()`, `getSync(key, 0)`.
7. **Release the HTTP request** in a `finally` (HW-19-0049).
8. **Supply a real image URL** - the shipped sample has it blanked
   (HW-19-0048).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The instrumentation (as shipped - see HW-19-0045 and HW-19-0047)

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/pages/StatisticPage.ets`

```ts
import { util } from '@kit.ArkTS';
import { DownloadImage } from '../utils/DownloadUtils';

let count: number = 0;
let begin: number = 0;
let end: number = 0;
let required: number = 0;

function addTimePrinter(targetClass: Object, methodName: string, isStatic: boolean) {
  util.Aspect.addBefore(targetClass, methodName, isStatic, () => {
    begin = new Date().getTime();
  });
  util.Aspect.addAfter(targetClass, methodName, isStatic, (instance: Object, ret: number) => {
    count = 1;                          // FIX (HW-19-0047): count += 1
    end = new Date().getTime();
    required = end - begin + required;  // FIX (HW-19-0045): measured at Promise creation
    return ret;                         // mandatory (HW-19-0046)
  });
}

addTimePrinter(DownloadImage, 'instanceMethod', false);
```

### The class-per-scope wrappers

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/utils/DownloadUtils.ets`

```ts
export class DownloadImage1 {
  async instanceMethod(url: string): Promise<image.PixelMap | undefined> {
    return await downloadNetworkImage(url);
  }
}

export class DownloadImage2 {
  async instanceMethod(url: string): Promise<image.PixelMap | undefined> {
    return await downloadNetworkImage(url);
  }
}

export class DownloadImage {
  async instanceMethod(url: string): Promise<image.PixelMap | undefined> {
    return await downloadNetworkImage(url);
  }
}

export class DownloadImage3 {
  async instanceMethod(url: string): Promise<image.PixelMap | undefined> {
    return await downloadNetworkImage(url);
  }
}
```

### The download itself

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/utils/DownloadUtils.ets`

```ts
let context = GlobalContext.get().getAppSettingContext() as common.UIAbilityContext;
let sourceFile: string = context.filesDir;

async function downloadNetworkImage(url: string): Promise<image.PixelMap | undefined> {
  // 下载完成之后需要保存图片数据到沙箱中
  let data = await http.createHttp().request(url,     // FIX (HW-19-0049): keep and destroy()
    {
      method: http.RequestMethod.GET,
      connectTimeout: 60000,
      readTimeout: 60000
    });
  let code = data.responseCode;
  if (code === http.ResponseCode.OK) {
    let buffer = data.result as ArrayBuffer;
    let downloadedPixelMap = PhotoAlbumsUtil.createPixelImage(buffer);
    // 保存图片至沙箱中，后续保存至相册时需要用到文件uri
    let fileName = sourceFile + '/test.jpeg';
    let file: fs.File | undefined = undefined;
    try {
      let fileUri1 = '';
      file = fs.openSync(fileName, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
      fs.writeSync(file.fd, buffer);
      fileUri1 = fileUri.getUriFromPath(fileName);
      Logger.info('load success' + fileUri1);
    } catch (e) {
      hilog.error(0x0000, 'testTag', `save image file error. Code: ${e.code}, message: ${e.message}`);
    } finally {
      if (file) {
        fs.closeSync(file);
      }
    }
    return downloadedPixelMap;
  } else {
    return undefined;
  }
}
```

### Folding the measurement into the persisted totals

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/pages/FunPage.ets`

```ts
aboutToAppear(): void {
  new DownloadImage3().instanceMethod(this.url).then((pixelMap: image.PixelMap | undefined) => {
    this.countTimes3 =
      PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
        LstOptionClass.countTimesInfo[3]) as number;
    this.delayTime3 =
      PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
        LstOptionClass.delayTimeInfo[3]) as number;
    this.downloadedPixelMap = pixelMap;
    if (this.countTimes3 === 0) {
      this.countTimes3 = count3;
    } else {
      this.countTimes3 = count3 + this.countTimes3;
    }
    this.delayTime3 = required3 + this.delayTime3;
    PreferencesClass.savePreferenceInfo(this.context, LstOptionClass.countTimesInfo[3],
      PreferencesClass.DEFAULT_STORE, this.countTimes3);
    PreferencesClass.savePreferenceInfo(this.context, LstOptionClass.delayTimeInfo[3],
      PreferencesClass.DEFAULT_STORE, this.delayTime3);
  });
}

aboutToDisappear(): void {
  this.countTimesArr[3] =
    PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
      LstOptionClass.countTimesInfo[3]) as number;
  this.delayTimeArr[3] =
    PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
      LstOptionClass.delayTimeInfo[3]) as number;
}
```

### The preferences wrapper

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/utils/PreferenceUtils.ets`

```ts
export class PreferencesClass {
  static DEFAULT_STORE: string = 'DEFAULT_STORE';
  static FIRST_OPEN_KEY: string = 'firstOpenKey';

  //创建仓库
  static getStore(content: Context, storeName: string) {
    return preferences.getPreferencesSync(content, { name: storeName });
  }

  // 2.给仓库中添加数据
  static savePreferenceInfo(content: Context, storeKey: string, storeName: string, value: number) {
    const STORE = PreferencesClass.getStore(content, storeName);
    STORE.putSync(storeKey, value);
    STORE.flushSync();
  }

  static getPreferenceInfo(content: Context, storeName: string, storeKey: string) {
    const STORE = PreferencesClass.getStore(content, storeName);
    const VAL = STORE.getSync(storeKey, 0);
    return VAL as number;
  }
}
```

### Reading the totals on the statistics screen

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/pages/SummaryPage.ets`

```ts
aboutToAppear(): void {
  for (let index = 0; index < 4; index++) {
    this.countTimesArr[index] =
      PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
        LstOptionClass.countTimesInfo[index]) as number;
    this.delayTimeArr[index] =
      PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
        LstOptionClass.delayTimeInfo[index]) as number;
  }
}
```

## Permissions & config

`ohos.permission.INTERNET` is genuinely required - `downloadNetworkImage` issues
an HTTP GET. The document's 权限说明 lists exactly that one permission and
nothing else, which matches the sample.

`CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]`, `"routerMap": "$profile:route_map"`,
the `EntryAbility` with the home skill, and an `EntryBackupAbility` backup
extension.

`Preferences` and `fileIo` need no permission - both stay inside the application
sandbox (`context.filesDir`).

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `util.Aspect.addBefore` /
  `addAfter` are API 11 APIs.
- **`Aspect` hooks class methods only**, identified by `(targetClass, methodName,
  isStatic)`; read-only methods are not supported.
- **The inserted function's return value wins**: "The final return value is the
  return value of the function inserted." Always return the original result from
  an `addAfter` hook.
- **`addAfter` sees the raw return value**, so for an `async` method it sees the
  Promise, not the resolved value.
- **One `HttpRequest` per request**, and each must be destroyed: "When the
  request is no longer needed, call destroy() to release resources. Otherwise,
  memory leaks may occur."
- **`GlobalContext` must be populated before `DownloadUtils` is first imported** -
  the module evaluates `GlobalContext.get().getAppSettingContext()` at load time
  and caches `filesDir` in a module-level variable.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **Timing an `async` method with `addAfter` is incorrect.** The hook fires when
  the Promise is created, so the reported 耗时 is not the download time. Wrap the
  returned Promise and stop the clock in its `then`. Typing the parameter
  `ret: number` is also wrong - it is a `Promise<image.PixelMap | undefined>`.
  (HW-19-0045)
- **The document's `addAfter` snippet omits `return ret`, which is incorrect.**
  Per the reference the inserted function's return value replaces the method's,
  so copying the snippet makes `instanceMethod` resolve to `undefined` and every
  `.then()` on it throws. All four sample pages do return it. (HW-19-0046)
- **`count = 1` instead of `count += 1` is incorrect**, and three of the four
  pages also drop the running total from the elapsed-time line. The document
  shows the accumulating form; the sample does not match it, and the four pages
  do not match each other. (HW-19-0047)
- **The image URL is empty in all four pages.** Nothing downloads, so the numbers
  on the statistics screen describe a request that never happened. The document
  does not mention that the URL is desensitised. (HW-19-0048)
- **`http.createHttp()` is called inline and never destroyed, which is
  incorrect** - the reference names this as a memory-leak cause. (HW-19-0049)
- **`savePreferenceInfo` and `getPreferenceInfo` take their key and store name in
  opposite orders.** Both are `(context, string, string, ...)`, so a swap
  compiles cleanly and silently reads or writes the wrong store.
- **`@State downloadedPixelMap?: image.PixelMap = undefined` is passed to
  `@Link downloadedPixelMap: image.PixelMap`** in `ReUsePage` - an optional bound
  to a non-optional link, which renders an undefined image until the download
  resolves.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` -
  `util.Aspect.addBefore` and `addAfter`: the parameter contract ("the second
  parameter is the return value of the original method"), the rule that "The
  final return value is the return value of the function inserted", and the
  official example.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-util#aspect11
- `documentation/harmonyos-references/03_system/js-apis-http.md` -
  `http.createHttp()`, the one-object-per-request rule and the `destroy()`
  requirement.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-http
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `getPreferencesSync`, `putSync`, `flushSync`, `getSync`, `deleteSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `openSync`, `writeSync`, `closeSync`, `OpenMode`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-application-aspect-programming-design -
  the aspect-programming design practice the document links.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
