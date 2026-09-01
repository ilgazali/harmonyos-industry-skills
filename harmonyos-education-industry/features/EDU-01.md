---
id: EDU-01
title: Education app framework - layered HAR skeleton with AVPlayer course video and a TextTimer exam
industry: 04_education
doc: huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
sample: huawei_industry_tree/04_education/downloads/Education_Framework_Code_V1.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.MediaKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkWeb", "@kit.CoreFileKit", "@kit.ArkData"]
apis: ["media.createAVPlayer", "AVPlayer.on('stateChange')", "AVPlayer.on('timeUpdate')", "AVPlayer.setSpeed", "AVPlayer.seek", XComponent, XComponentController, setXComponentSurfaceRect, getXComponentSurfaceId, TextTimer, TextTimerController, "AppStorage.setOrCreate", "@StorageProp", Navigation, NavPathStack, NavDestination, routerMap, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", setWindowPrivacyMode, "dataPreferences.getPreferences", "fs.listFile"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.PRIVACY_WINDOW"]
min_api: 20
modules: [entry, common/basic, features/home, features/login, features/mine, features/online, features/train]
findings: [HW-04-0001, HW-04-0002, HW-04-0003, HW-04-0004, HW-04-0005, HW-04-0006, HW-04-0007, HW-04-0008, HW-04-0009, HW-04-0010, HW-04-0011, HW-04-0012, HW-04-0013, HW-04-0014]
status: verified-with-fixes
---

## When to use

Load this card when you are starting an **education or training application** and
need the whole skeleton at once: a four-tab shell, a course catalogue, a video
course player, an online exam with a countdown, and a personal area with
courses, exams and a wrong-answer book.

It is the industry's root architecture document, and the two industry-specific
techniques it settles are the ones every education app needs and no generic
template gives you:

- **Course video** - AVPlayer bound to an XComponent surface, with speed
  control, seek, progress and a custom control bar.
- **Online exam** - a `TextTimer` countdown that auto-submits at zero and
  survives the user leaving the page mid-exam by parking the remaining time in
  `AppStorage`.

Every other card in this industry (`EDU-03` … `EDU-20`) is a single feature that
plugs into this shell. Start here for module boundaries and routing, then take
the feature card for the specific screen.

## Feature checklist

- One `entry` HAP for phone; every business area is a HAR, so the app size stays
  controllable and nothing is loaded dynamically.
- Bottom navigation with four tabs: 首页 (home), 培训 (training), 消息
  (messages), 我的 (mine).
- Course detail plays an online video: play/pause, previous/next, seek by
  slider, playback speed, elapsed and total time.
- The exam page counts down, shows one question at a time, allows going back to
  a previous question and changing the answer, and auto-submits at zero.
- Leaving the exam mid-way stores the remaining time; re-entering resumes from
  it, not from the full duration.
- The login page is capture-proof (`setWindowPrivacyMode`).
- The layout is full-screen and pads itself by the real avoid areas rather than
  by hard-coded status-bar heights.
- Cross-HAR navigation goes through per-module `route_map.json`, so `entry`
  never imports a feature page directly.

## Architecture

Three layers, exactly as the document's 软件视图 (software view) prescribes.

```
Education_Framework_Code_V1
├── entry                      product layer - the only HAP, phone only
│   └── src/main/ets
│       ├── entryability/EntryAbility.ets   full-screen + avoid areas -> AppStorage
│       ├── pages/Index.ets                 @Entry: Navigation + Tabs + CustomTabBar
│       ├── pages/Tab.ets                   the custom bottom bar
│       └── viewmodel/ConstantsInterface.ets
├── features                   feature layer - one HAR per business area
│   ├── home                   banner, entries, notifications
│   ├── login                  login, register, password, cancellation, privacy
│   ├── mine                   my courses / exams / wrong answers / settings
│   ├── online                 message list
│   └── train                  training list, training detail, course detail,
│                              AVPlayerDemo, exam detail
└── common/basic               common layer - constants, Logger, breakpoints,
                               preferences, MyGlobalContext, ResManager
```

**The data flow that holds it together is `AppStorage`, not props.**
`EntryAbility` writes `topRectHeight` / `bottomRectHeight` there;
`entry/pages/Index.ets` writes the single `NavPathStack` there under the key
`pathStack`. Every page in every HAR then reads them:

```typescript
@StorageProp('topRectHeight') topRectHeight: number = 0;
pathStack: NavPathStack = AppStorage.get('pathStack') as NavPathStack;
```

That is what lets a HAR page navigate without importing `entry`, and it is why
the HARs have no dependency on each other. The exam feature reuses the same
channel for its own state (`MyCount`, `Score`, `Time`) - convenient, and also
the source of the score-clobbering defect below (`HW-04-0012`).

**Cross-HAR routing is by name.** Each feature HAR declares
`"routerMap": "$profile:route_map"` in its `module.json5` and lists its pages in
`src/main/resources/base/profile/route_map.json`, each entry naming a
`@Builder` function exported from the page file. `entry` declares no router map
at all, and `pushPathByName('MineStartTestPage', null)` still resolves, because
the route maps of the dependent HARs are merged at build time.

The documented tree matches the zip with two exceptions: the source-path
comments in the code sections drop the `src/main` segment (`HW-04-0003`), and
`ResManager.ets` is really named `ResManager .ets` (`HW-04-0004`).

## Implementation steps

1. **Create one `entry` HAP and six HARs.** `deviceTypes: ["phone"]` on the
   entry, `"type": "har"` on the rest, and wire them into
   `entry/oh-package.json5` as `file:` dependencies under a single scope
   (`@safe/home`, `@safe/train`, …). Add every module to the root
   `build-profile.json5`.
2. **Give each feature HAR a route map** and export a `@Builder` per page:
   ```typescript
   @Builder
   export function MineStartTestPageBuilder() { MineStartTestPage(); }
   ```
   Never import a feature page from `entry`; import only the tab-level component
   from the HAR's `Index.ets`.
3. **In `EntryAbility.onWindowStageCreate`,** set full-screen layout, read both
   avoid areas, publish them to `AppStorage`, and subscribe to
   `avoidAreaChange`. **Keep the window handle and call `off` in
   `onWindowStageDestroy`** - the sample does not (`HW-04-0008`).
4. **In `entry/pages/Index.ets`,** create the one `NavPathStack`, publish it as
   `pathStack`, and wrap the `Tabs` in a `Navigation`. Drive the tab bar with a
   `@Link` index and `barHeight(0)` so the real bar is your own component.
5. **For the course video,** create the AVPlayer in `aboutToAppear`, register
   the callbacks, and set `url`. Obtain the surface id in the XComponent's
   `onLoad`, assign it in the `initialized` branch of `stateChange`, then
   `prepare()`. Use the **`XComponentOptions` constructor and
   `setXComponentSurfaceRect`** - the document's snippet uses the deprecated
   forms (`HW-04-0001`, `HW-04-0002`).
6. **Release the player symmetrically** in `aboutToDisappear`: `off` for *every*
   event you registered - the sample forgets `speedDone` (`HW-04-0009`) - then
   `release()`. Clear any pending `setInterval` there too (`HW-04-0014`).
7. **Bound the error path.** Do not let `on('error')` call `reset()`
   unconditionally; `reset()` returns to `idle`, which re-assigns the URL and
   fails again forever (`HW-04-0010`).
8. **For the exam,** drive the countdown from `TextTimer({ isCountDown: true,
   count, controller })`, start it in `onAppear`, and in `onTimer` compute the
   remaining time and auto-submit when it reaches zero.
9. **Persist the remaining time in `onDisAppear` only when the exam was not
   submitted.** Do **not** add an else branch that writes `Score` - it races the
   submit path and zeroes the score just computed (`HW-04-0012`).
10. **Store answers in a map keyed by question**, not an append-only array; the
    user can go back and change an answer (`HW-04-0011`).
11. **Declare only the permissions you request.** `INTERNET` for the video,
    `PRIVACY_WINDOW` for the capture-proof login page. Drop the four unused
    `user_grant` permissions the sample carries (`HW-04-0006`, `HW-04-0007`).

## Verified snippets

All snippets are from `Education_Framework_Code_V1.zip`. Corrected forms are marked.

**Avoid areas and full screen — `entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-04-0008`)

```typescript
import { UIAbility } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';
import { BusinessError } from '@kit.BasicServicesKit';

const DOMAIN = 0x0000;

export default class EntryAbility extends UIAbility {
  private windowClass?: window.Window;                 // FIX: sample keeps only a local

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
        return;
      }
    });
    const windowClass: window.Window = windowStage.getMainWindowSync();
    this.windowClass = windowClass;

    windowClass.setWindowLayoutFullScreen(true)
      .catch((err: BusinessError) => {
        hilog.error(DOMAIN, 'testTag', 'Failed to set full-screen. Cause:' + JSON.stringify(err));
      });

    let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
    AppStorage.setOrCreate('bottomRectHeight', windowClass.getWindowAvoidArea(type).bottomRect.height);
    type = window.AvoidAreaType.TYPE_SYSTEM;
    AppStorage.setOrCreate('topRectHeight', windowClass.getWindowAvoidArea(type).topRect.height);

    windowClass.on('avoidAreaChange', (data) => {
      if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
        AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
      } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
        AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
      }
    });
  }

  onWindowStageDestroy(): void {
    this.windowClass?.off('avoidAreaChange');          // FIX: absent in the sample
    this.windowClass = undefined;
  }
}
```

**Two avoid areas, two types.** `TYPE_SYSTEM` gives the status bar at the top,
`TYPE_NAVIGATION_INDICATOR` the gesture bar at the bottom. Both are read once
for the first frame *and* subscribed to, because a fold, a rotation or a
gesture-mode change moves them. The values are **px**; every consumer converts
with `px2vp` at the point of use.

**The shell — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
@Entry
@Component
struct Index {
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @State currentTabIndex: number = 0;
  pathStack: NavPathStack = new NavPathStack();
  private controller: TabsController = new TabsController();

  aboutToAppear() {
    AppStorage.setOrCreate('pathStack', this.pathStack);
  }

  build() {
    Navigation(this.pathStack) {
      Column() {
        Tabs({ index: this.currentTabIndex, controller: this.controller }) {
          TabContent() { Column() { HomeView(); } }
          TabContent() { TrainPage(); }
          TabContent() { OnlineIndexView(); }
          TabContent() { MinePage(); }
        }
        .clip(false)
        .layoutWeight(1)
        .barHeight(0)                       // the built-in bar is suppressed...
        .scrollable(false)
        .onChange((index: number) => { this.currentTabIndex = index; })

        CustomTabBar({ currentTabIndex: $currentTabIndex })   // ...this one replaces it
          .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
      }
      .backgroundColor('#F1F3F5')
    }
    .hideTitleBar(true)
    .hideToolBar(true)
  }
}
```

**`Navigation` wraps `Tabs`, not the other way round.** That single decision is
what makes the whole app work: pushing a `NavDestination` from any tab covers
the tab bar, and popping returns to the tab the user was on, because the tabs
never left the tree. The `NavPathStack` is created here and published to
`AppStorage` so five HARs can navigate without importing `entry`.

`barHeight(0)` plus `scrollable(false)` disables the built-in bar and the swipe;
`CustomTabBar` is the real bar and only writes `currentTabIndex` back through
`@Link`.

**Course video — `features/train/src/main/ets/view/AVPlayerDemo.ets`** (corrected, see `HW-04-0001`, `HW-04-0002`, `HW-04-0009`, `HW-04-0010`)

```typescript
import media from '@ohos.multimedia.media';
import { BusinessError } from '@ohos.base';

@Component
export struct AVPlayerDemo {
  @State isPlay: boolean = false;
  @State flag: boolean = false;
  @State durationTime: number = 0;
  @State currentTime: number = 0;
  private avPlayer: media.AVPlayer | undefined = undefined;
  private xComponentController = new XComponentController();
  private surfaceID: string = '';
  private retries: number = 0;                          // FIX: sample has no bound

  aboutToAppear(): void {
    this.createAVPlayer();
    this.reset();
  }

  aboutToDisappear(): void {
    if (this.avPlayer) {
      this.avPlayer.off('timeUpdate');
      this.avPlayer.off('seekDone');
      this.avPlayer.off('speedDone');                   // FIX: missing in the sample
      this.avPlayer.off('error');
      this.avPlayer.off('stateChange');
      this.avPlayer.release();
    }
  }

  @Builder
  VideoPlayer() {
    XComponent({                                        // FIX: XComponentOptions form
      type: XComponentType.SURFACE,
      controller: this.xComponentController
    })
      .onLoad(() => {
        this.xComponentController.setXComponentSurfaceRect({   // FIX: ...SurfaceSize is deprecated
          surfaceWidth: 1920,
          surfaceHeight: 1080
        });
        this.surfaceID = this.xComponentController.getXComponentSurfaceId();
      })
      .width('100%')
      .height('100%')
  }

  setAVPlayerCallback(avPlayer: media.AVPlayer) {
    avPlayer.on('timeUpdate', (time: number) => {
      this.currentTime = Math.floor(time * this.durationTime / avPlayer.duration);
      this.currentStringTime = this.secondToTime(Math.floor(time / 1000));
    });

    avPlayer.on('error', (err: BusinessError) => {      // FIX: sample takes no argument
      Logger.error(TAG, `AVPlayer error ${err.code}: ${err.message}`);
      if (this.retries++ < 2) {                        // FIX: sample resets unconditionally
        avPlayer.reset();
      }
    });

    avPlayer.on('stateChange', async (state: string) => {
      switch (state) {
        case 'idle':
          avPlayer.url = VIDEO_URL;
          break;
        case 'initialized':
          this.reset();
          avPlayer.surfaceId = this.surfaceID;          // the id captured in onLoad
          avPlayer.prepare();
          break;
        case 'prepared':
          this.flag = true;
          this.durationTime = Math.floor(avPlayer.duration / 1000);
          avPlayer.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_1_00_X);
          avPlayer.videoScaleType = media.VideoScaleType.VIDEO_SCALE_TYPE_FIT;
          avPlayer.play();
          break;
        case 'playing': this.isPlay = true; break;
        case 'paused':
        case 'stopped':
        case 'completed': this.isPlay = false; break;
      }
    });
  }
}
```

**The state machine is the API.** You never call `prepare` after `createAVPlayer`
directly; you assign `url`, which moves the player to `initialized`, and only
*there* is the surface id valid to assign and `prepare()` legal. Getting this
order wrong is the single most common AVPlayer failure, and it is why the
surface id must already exist - `XComponent.onLoad` fires before the user can
reach the play button, but if it did not, `surfaceID` would be the empty string
and `prepare()` would produce audio with no picture.

`OPERATE_STATE = ['prepared', 'playing', 'paused', 'completed']` guards every
control entry point (`setSpeed`, `play`, `seek`), because calling them in
`idle`/`initialized` throws.

**Exam countdown — `features/mine/src/main/ets/views/MineStartTestPage.ets`** (corrected, see `HW-04-0011`, `HW-04-0012`)

```typescript
const EXAM_DURATION = 30000;                        // ms

@Component
export struct MineStartTestPage {
  @StorageProp('MyCount') count: number = 0;
  @StorageProp('Score') score: number = 0;
  @State isSubmitted: boolean = false;
  textTimerController: TextTimerController = new TextTimerController();
  elapsedTime: number = 0;
  private answers: Map<string, number> = new Map();     // FIX: sample uses a push-only array

  aboutToAppear(): void {
    const myCount: number | undefined = AppStorage.get('MyCount');
    this.count = (myCount !== undefined && myCount !== 0) ? myCount : EXAM_DURATION;
    this.currentModel = ERROR_MODELS[this.currentIndex];
  }

  build() {
    // ...
    TextTimer({ isCountDown: true, count: this.count, controller: this.textTimerController })
      .format(this.format)
      .onTimer((utc: number, elapsedTime: number) => {
        this.elapsedTime = this.count - elapsedTime * 1000;   // remaining, in ms
        if (elapsedTime * 1000 === this.count) {
          this.submit(true);                                  // time is up
        }
      })
      .onAppear(() => { this.textTimerController.start(); })
      .onDisAppear(() => {
        if (!this.isSubmitted) {
          AppStorage.setOrCreate('MyCount', this.elapsedTime);  // park the remaining time
        }
        // FIX: the sample's else branch writes AppStorage 'Score' = 0 here and
        // clobbers the score submit() has just computed.
      })
  }
}
```

**`count` is milliseconds and `onTimer`'s `elapsedTime` is seconds.** The mixed
units are the reason the sample multiplies by 1000 on both lines; keep the two
straight or the auto-submit either never fires or fires immediately.

**The resume trick is one key and two branches**: `onDisAppear` writes the
remaining time into `AppStorage` under `MyCount`, `aboutToAppear` reads it back
and falls through to the full duration when it is absent or zero. `TextTimer`
itself is stateless across a page change - the countdown restarts from whatever
`count` says - so this is the whole mechanism.

**Capture-proof login — `features/login/src/main/ets/views/LoginView.ets`** (as shipped)

```typescript
private context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;

async aboutToAppear() {
  const myWindow = this.context.windowStage.getMainWindowSync();
  myWindow.setWindowPrivacyMode(true);       // needs ohos.permission.PRIVACY_WINDOW
}

aboutToDisappear(): void {
  const myWindow = this.context.windowStage.getMainWindowSync();
  myWindow.setWindowPrivacyMode(false);      // symmetric - the flag is on the window, not the page
}
```

The mode is a property of the **window**, not of the page, so the `false` in
`aboutToDisappear` is mandatory: without it every later screen in the app is
also unrecordable. The permission is `system_grant`, so no runtime request is
needed - only the `module.json5` line.

## Permissions & config

`entry/src/main/module.json5` — the corrected list (see `HW-04-0006`, `HW-04-0007`):

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },          // system_grant - online course video
  { "name": "ohos.permission.PRIVACY_WINDOW" }     // system_grant - capture-proof login page
]
```

- Both are `system_grant`, so neither needs `reason`/`usedScene` and neither
  needs a runtime request.
- The sample additionally ships `READ_MEDIA`, `WRITE_MEDIA`, `CAMERA` and
  `MEDIA_LOCATION`. All four are `user_grant`, nothing in the project ever calls
  `requestPermissionsFromUser`, and `READ_MEDIA` / `WRITE_MEDIA` are deprecated
  from API 22 in favour of the Files permission group. Delete them.
- `ohos.permission.ACCESS_BIOMETRIC` is also declared and also unused; keep it
  only if you add the biometric login the framework leaves unimplemented.
- In a multi-HAP application, permissions declared in `entry` apply to the whole
  application - do not repeat them in the feature HARs.

Each feature HAR needs only:

```json5
{
  "module": {
    "name": "train",
    "type": "har",
    "routerMap": "$profile:route_map",
    "description": "$string:shared_desc",
    "deviceTypes": ["phone", "tablet", "2in1"]
  }
}
```

## Constraints

- DevEco Studio 6.0.0 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  Huawei phone running HarmonyOS 6.0.0 Release or later.
- `compatibleSdkVersion` and `targetSdkVersion` are both `6.0.0(20)`.
- The `entry` HAP is `deviceTypes: ["phone"]` only - the document states
  「应用设备形态只有手机端」 (the application form factor is phone only).
- This is **framework code, not a complete application**. The document says so:
  「本篇代码非应用全量代码，只包括应用部分框架代码」. All data is local mock
  data under each HAR's `viewmodel/*LocalData.ets`; there is no backend.
- **Login performs no validation.** The document is explicit:
  「框架代码中登录验证模块，只是UI能力，手机号位数满足条件，任意密码可登录」
  (the login module is UI only; any password is accepted). `LoginView.onClick`
  is a bare `pathStack.pop()`. Do not ship this.
- The exam is 30 seconds long (`EXAM_DURATION = 30000`) with five identical
  placeholder questions - a demonstration of the timer, not of an exam engine.
- The course video URL is a hard-coded Huawei marketing clip, flagged in the
  code as 当前链接为参考链接 (this link is for reference only).
- The service-card exam reminder in 行业创新设计 is a **design proposal only** -
  no `FormExtensionAbility` exists in the zip.

## Pitfalls

- **`HW-04-0001` — the document's video snippet calls
  `setXComponentSurfaceSize`,** deprecated since API 12. Use
  `setXComponentSurfaceRect`; the shipped code already does, so only the
  document is wrong.
- **`HW-04-0002` — both the document and the code use the
  `{id, type, libraryname, controller}` XComponent constructor,** deprecated
  since API 12 in favour of `XComponentOptions`. `libraryname` is worse than
  useless here: the reference states `onLoad` fires *only* when it is not set,
  and this component depends on `onLoad` for the surface id.
- **`HW-04-0003` — every path comment in the code sections says
  `features/train/src/ets/...`,** dropping `main`. `src/ets` is not a directory
  hvigor compiles, and the document's own tree contradicts it.
- **`HW-04-0004` — the tree says `ResManager.ets`; the files are
  `ResManager .ets`** with a space, and four imports carry the space in the
  specifier. Renaming to the documented name breaks the build.
- **`HW-04-0005` — the module list claims to show 全部HAR模块 and omits
  `login`,** which the tree, `build-profile.json5` and `entry/oh-package.json5`
  all contain - and which is the one module the document tells you to finish
  yourself.
- **`HW-04-0006` — four `user_grant` permissions are declared and never
  requested.** `READ_MEDIA`, `WRITE_MEDIA`, `CAMERA`, `MEDIA_LOCATION`: no
  `requestPermissionsFromUser` call exists anywhere in the project. The document
  says only `INTERNET` is needed.
- **`HW-04-0007` — those permissions use `reason: "$string:app_name"` and
  `when: "always"`.** The app label is not a reason, and this app has no
  background task.
- **`HW-04-0008` — `avoidAreaChange` is subscribed and never released.** The
  handle is a local in `onWindowStageCreate` and `onWindowStageDestroy` only
  logs.
- **`HW-04-0009` — five AVPlayer events registered, four unregistered.**
  `speedDone` survives `release()`.
- **`HW-04-0010` — `on('error')` calls `reset()` unconditionally,** which
  returns to `idle`, which re-assigns the same URL. A dead link spins the media
  pipeline forever, and the handler takes no argument so nothing is logged.
- **`HW-04-0011` — answers are pushed to an array and read back with `find`,**
  which returns the *first* match. Go back with 上一题, change your answer, and
  the original is still what gets scored.
- **`HW-04-0012` — `TextTimer.onDisAppear` writes `Score = 0` whenever
  `isSubmitted` is true,** which is exactly the state `submit(true)` leaves
  behind after computing the score. A timed-out exam reports zero.
- **`HW-04-0013` — `clearCache` builds `cacheDir + filename` with no
  separator,** so every `statSync` throws into a swallowing `catch` and 清除缓存
  silently does nothing.
- **`HW-04-0014` — the play-when-ready `setInterval` handle is a local,** so
  leaving the course page while the video is still preparing leaves a 100 ms
  timer running for the life of the process.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - the XComponent overloads, `setXComponentSurfaceRect`, and the `libraryname`/`onLoad` interaction
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimer`, `isCountDown`, `count`, `onTimer` units
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-guides/02_media/video-playback.md` - the AVPlayer state machine and surface binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `documentation/harmonyos-guides/02_media/media-kit-intro.md` - Media Kit overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/media-kit-intro
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `AppStorage`, `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `Navigation`, `NavPathStack`, `route_map.json`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on('avoidAreaChange')`, `setWindowPrivacyMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene.when` specifications
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `READ_MEDIA` / `WRITE_MEDIA` deprecation and the `user_grant` list
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `EDU-02` and `EDU-21` - the two architecture-only revisions of this same document
