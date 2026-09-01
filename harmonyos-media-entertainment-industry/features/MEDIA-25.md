---
id: MEDIA-25
title: Navigation splash page - the brand screen is the navigation bar, not a route
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/25_splash_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page-0000002303412054
sample: huawei_industry_tree/13_media_entertainment/downloads/Navigation实现闪屏页示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, curves, dataSharePredicates, fileIo, fileUri, hilog, image, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0032, HW-13-0050, HW-13-0051, HW-13-0058, HW-13-0097, HW-13-0098, HW-13-0100, HW-13-0101, HW-13-0102]
status: verified-with-fixes
---

## When to use

Load this card when the app needs **a branded first screen that is gone after a
couple of seconds and must never come back**, and you are already using
`Navigation` for routing. The pattern: put the splash artwork in the
`Navigation` container's own bar area, push the real home page onto the path
stack, and hide the bar. No router entry for the splash, no `replaceUrl`, no
transition to fight with.

The trick generalises to any "shell that exists once": an onboarding carousel,
a licence agreement gate, a first-run permission explainer. Anything whose only
job is to be shown before the app proper, and whose disappearance should not
leave a back-stack entry behind.

The sample bolts this two-file idea onto a full video-merge editor
(`MainPage` + `VideoList`), which is where all four of its defects live. Take
the splash mechanism; treat the editor as a second, weaker sample that happens
to share the zip - it is the same code as `MergeVideo` (`MEDIA-20`), copied,
with the same bugs.

## Feature checklist

- On launch, a full-bleed splash image fills the window, edge to edge.
- After exactly 2000 ms the app shows the main page, without a visible route
  transition on the splash itself.
- The splash never reappears while the app is alive.
- The timer is cancelled if the page is destroyed before it fires.
- The main page renders inside the system avoid areas (status bar, navigation
  indicator) using values published by the ability.
- The main page hosts the actual demo: pick up to 9 videos, see them as a
  drag-reorderable horizontal thumbnail strip with durations, preview one,
  merge and save to the gallery.

## Architecture

One `entry` module. Two `@Entry`-decorated pages, both registered in the
router map, plus three utility singletons.

```
entry/src/main/ets
├── common/Constants.ets              layout literals (FULL = '100%', sizes, colors)
├── entryability/EntryAbility.ets     full-screen window, avoid areas -> AppStorage, uiContext -> AppStorage
├── entrybackupability/
├── pages
│  ├── SplashPage.ets                 @Entry: Navigation host + the splash image + the 2 s timer
│  ├── MainPage.ets                   the merge editor, reached as a NavDestination
│  └── VideoList.ets                  horizontal thumbnail strip with long-press drag reorder
└── utils
   ├── CacheUtils.ets                 work dir lifecycle + showAssetsCreationDialog save
   ├── Logger.ets                     hilog wrapper
   ├── PhotoUtils.ets                 getAssets -> getThumbnail
   └── VideoUtils.ets                 ffmpeg transcode + concat list + duration formatting
```

The documented tree matches the zip exactly, including the file comments.

**The design decision worth copying** is that the splash is the `Navigation`
component's *content*, not a destination on its stack. `SplashPage` builds a
`Navigation(this.pageInfos)` whose children are the splash image; `MainPage` is
pushed by name onto `pageInfos`, so it renders as a `NavDestination` on top;
and `hideNavBar(this.isHideNavBar)` - flipped in the same callback that pushes -
removes the splash from the tree. One boolean and one push, no route
replacement, no reverse animation to suppress. Both pages are still listed in
`main_pages.json` and `route_map.json`, so `MainPage` keeps a name that any
other push could target later.

The cost of that choice is worth knowing before you copy it: because the splash
is the bar and not a stack entry, a `pop()` from `MainPage` lands on a nav bar
that `isHideNavBar` still keeps hidden - an empty window rather than the splash.
The sample never pops, so it never shows; an app with a real back gesture on the
home page must reset `isHideNavBar` in the pop handler or leave `hideNavBar`
permanently true.

## Implementation steps

1. **Register both pages in `route_map.json`** with a `buildFunction` each, and
   export a `@Builder` function per page (`SplashBuilder`, `MainBuilder`)
   alongside the struct. Point `module.json5`'s `"routerMap"` at the profile.
2. **Make the splash the `Navigation` host.** `pageInfos: NavPathStack = new NavPathStack()`
   is a plain field, not `@State` - the stack notifies on its own.
3. **Start the timer in `aboutToAppear`,** keep the id in a field, and clear it
   in `aboutToDisappear` so a fast destroy cannot push onto a dead stack.
4. **In the timeout, set `isHideNavBar = true` and push by name** in that
   order. `pushPathByName('MainPage', null, false)` - the third argument is
   `animated`, and `false` is deliberate: the destination should appear without
   sliding over the brand image.
5. **Publish the avoid areas from `EntryAbility`** into `AppStorage` and read
   them with `@StorageProp` on every page, converting with `px2vp` at the point
   of use.
6. **Get the UIContext from the component, never from `new UIContext()`**
   (`HW-13-0032`). The sandbox paths, the media library helper and the toasts
   in `MainPage` all hang off that context.
7. **Declare and request `ohos.permission.READ_IMAGE_VIDEO` before calling
   `getAssets`** - or drop `getAssets` and read metadata off the picked fd
   (`HW-13-0058`). Picker URIs do not grant query rights.
8. **Push one record object per picked video,** not three parallel arrays fed
   from three different callbacks (`HW-13-0051`).
9. **Open reused work files with `OpenMode.TRUNC`** and clear the work
   directory per run, not only after a successful save (`HW-13-0050`).

## Verified snippets

All snippets are from `Navigation实现闪屏页示例代码.zip`. Corrected forms are marked.

**The whole splash — `entry/src/main/ets/pages/SplashPage.ets`** (as shipped)

```typescript
import { Constants } from '../common/Constants';

@Builder
export function SplashBuilder(name: string, param: Object) {
  SplashPage();
}

@Entry
@Component
struct SplashPage {
  pageInfos: NavPathStack = new NavPathStack();
  @State isHideNavBar: boolean = false;
  private timeOutId: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageProp('topRectHeight') topRectHeight: number = 0;

  aboutToAppear() {
    this.jumpToMainPage();
  }

  jumpToMainPage() {
    this.timeOutId = setTimeout(() => {
      this.isHideNavBar = true;                                  // the splash leaves the tree
      this.pageInfos.pushPathByName('MainPage', null, false);    // false = no push animation
    }, 2000);
  }

  aboutToDisappear() {
    clearTimeout(this.timeOutId);
  }

  build() {
    Navigation(this.pageInfos) {
      Flex({ direction: FlexDirection.Column, alignItems: ItemAlign.Center }) {
        Column() {
          Image($r('app.media.splash_bg'))
            .height(Constants.FULL)
            .width(Constants.FULL);
        }
        .flexGrow(1);
      }
      .height(Constants.FULL)
      .width(Constants.FULL);
    }
    .hideToolBar(true)
    .hideNavBar(this.isHideNavBar);
  }
}
```

**Three lines carry the whole feature.** `hideNavBar(this.isHideNavBar)` is what
makes the splash disappear - the children of `Navigation` *are* the nav bar, so
hiding the bar unmounts the image. `pushPathByName(..., false)` puts `MainPage`
on the stack without an animation, so the two events (bar vanishes, destination
appears) read as one cut rather than a slide over the brand. And
`clearTimeout` in `aboutToDisappear` is not boilerplate here: without it a
launch that is killed inside the two seconds still fires a push against a stack
whose component is gone.

Note what is *absent*: no `@State` on `pageInfos`, no `router.replaceUrl`, no
`NavDestination` in this file at all. The splash is deliberately not a
destination.

**The router map — `entry/src/main/resources/base/profile/route_map.json`** (as shipped)

```json
{
  "routerMap": [
    {
      "name": "SplashPage",
      "pageSourceFile": "src/main/ets/pages/SplashPage.ets",
      "buildFunction": "SplashBuilder",
      "data": { "description": "this is Splash Page" }
    },
    {
      "name": "MainPage",
      "pageSourceFile": "src/main/ets/pages/MainPage.ets",
      "buildFunction": "MainBuilder",
      "data": { "description": "this is Main Page" }
    }
  ]
}
```

The names in this file are the only contract between `pushPathByName` and the
page; `module.json5` wires it up with `"routerMap": "$profile:route_map"` and
`"pages": "$profile:main_pages"`, and `EntryAbility` loads
`'pages/SplashPage'` as the window content. Registering `SplashPage` here as
well as in `main_pages.json` is redundant for this flow - nothing ever pushes
it - but harmless, and it keeps the two pages symmetric.

**The host context — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-13-0032`)

```typescript
@Entry
@Component
struct MainPage {
  private context = this.getUIContext().getHostContext() as common.UIAbilityContext;  // FIX
  getLocalDirPath = this.context.cacheDir;
  videoDir = this.getLocalDirPath + '/videoDir';
  picDir = this.getLocalDirPath + '/picDir';
  private photoUtils: PhotoUtils = new PhotoUtils();

  aboutToAppear(): void {
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;  // FIX
    CacheUtils.setContext(context);
    VideoUtils.setContext(context);
    CacheUtils.initializeFolder();
    if (fs.accessSync(this.picDir)) {
      fs.rmdirSync(this.picDir);
    }
    fs.mkdirSync(this.picDir);
  }
```

The shipped file writes `const uiContext = new UIContext();` at module scope
and `let uiContext = new UIContext();` again inside `aboutToAppear`. A bare
`new UIContext()` is not attached to any window, so `getHostContext()` has no
host: `cacheDir` is read off nothing, and every sandbox path, the media library
helper and the toasts derive from it. The same file already gets it right at
the bottom of `build()` -
`top: this.getUIContext().px2vp(this.topRectHeight)` - which is the form to use
everywhere. `PhotoUtils.ets` has the sibling shape,
`AppStorage.get('uiContext')` evaluated at module import time, before
`EntryAbility` writes the key inside its `loadContent` callback; fetch it
lazily inside the method instead.

**One record per video — same file, the picker callback** (corrected, see `HW-13-0051`, `HW-13-0058`)

```typescript
interface VideoRecord { uri: string; thumb: string; time: string; }
@State records: VideoRecord[] = [];                    // FIX: replaces list/listUri/timeList

photoViewPicker.select(photoSelectOptions).then(async (photoSelectResult) => {
  let i = 0;
  for (let result of photoSelectResult.photoUris) {
    i++;
    const thumbnailView = await this.photoUtils.acquireThumbnailByUrl(result);
    if (thumbnailView === undefined) {
      Logger.error('the pixelMap acquires is undefined!');
      continue;
    }
    const uri = fileUri.getUriFromPath(this.picDir + '/pic' + i.toString() + '.jpg');
    const file = await fileIo.open(uri,
      fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC);  // FIX
    const imagePackerApi = image.createImagePacker();
    const packOpts: image.PackingOption = { format: 'image/jpeg', quality: 98 };
    await imagePackerApi.packToFile(thumbnailView, file.fd, packOpts);               // FIX: awaited
    await imagePackerApi.release();
    fileIo.close(file.fd);

    // FIX: getAssets needs ohos.permission.READ_IMAGE_VIDEO declared AND requested;
    // without it this throws and the duration is silently lost (HW-13-0058).
    const videoTime = VideoUtils.formatDuration(Number(await this.durationOf(result)));
    this.records.push({ uri: result, thumb: uri, time: videoTime });                 // FIX: one push
  }
});
```

**The shipped loop pushes into three arrays from three different places.**
`listUri` is pushed synchronously, `list` (thumbnails) only inside the
`packToFile` *callback* - so it is skipped when packing fails and arrives in
completion order, not selection order - and `timeList` only when `getAssets`
succeeds. `VideoList` then indexes all three by the thumbnail's position and
`handleDrag` splices all three by it. One failed thumbnail and taps play the
wrong clip, durations sit under the wrong video, and the merge order stops
matching what the user arranged. A single record object per video removes the
whole class of bug; awaiting `packToFile` removes the ordering half of it.

`getAssets` is the second half of the same failure. `module.json5` in this
sample declares **no `requestPermissions` at all**, and a picker URI does not
carry query rights, so the call throws, the catch only logs, and no duration is
ever appended - which is exactly what puts `timeList` out of step with the
other two arrays.

**Truncating the concat list — `entry/src/main/ets/utils/VideoUtils.ets`** (corrected, see `HW-13-0050`)

```typescript
const file = await fsStream.open(this.videoTxt,
  fsStream.OpenMode.CREATE | fsStream.OpenMode.READ_WRITE | fsStream.OpenMode.TRUNC);  // FIX
let videoNum = 0;
for (let uri of videoList) {
  videoNum++;
  const videoName = 'video' + videoNum.toString() + '.ts';
  const fileContent = 'file \'' + videoName + '\'\n';
  await fsStream.write(file.fd, fileContent);
  // ... last iteration runs MP4Parser.videoMultMerge(this.videoTxt, out)
}
await fsStream.close(file.fd);
```

`video.txt` is the ffmpeg concat manifest, and it is reopened on every merge.
Without `TRUNC` a shorter second selection leaves the previous run's
`file 'video4.ts'` lines beyond the new content, and the leftover `.ts`
segments are still on disk because `CacheUtils.initializeFolder()` only runs
after a *successful* save. The output then contains clips the user removed. The
cheap fix is the flag; the complete fix is to clear the work directory at the
start of every run.

## Permissions & config

**None declared.** `module.json5` has no `requestPermissions` block at all.

That is the bug in `HW-13-0058`: `PhotoUtils.acquireThumbnailByUrl` and
`MainPage`'s duration lookup both call `phAccessHelper.getAssets`, which needs
`ohos.permission.READ_IMAGE_VIDEO` declared and granted. `PhotoViewPicker`
itself needs nothing - that is the point of the picker - but the URIs it hands
back are scoped to reading the file, not to querying the media library. Either
add the permission with `reason` and `usedScene`, or drop `getAssets` and read
size and duration from the picked fd with `AVMetadataExtractor`.

The splash mechanism itself needs no permission: it is one `Image` and one
`setTimeout`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Popping `MainPage` returns to a nav bar that `isHideNavBar` still hides -
  an empty window. Reset the flag in the pop handler if the home page has a
  real back gesture.
- The 2000 ms delay is a bare literal in `jumpToMainPage`, unrelated to how
  long the app actually takes to be ready. A splash that is meant to cover
  initialisation should race the timer against the readiness signal.
- The merge half of the sample depends on `@ohos/mp4parser` (ffmpeg) and writes
  to `cacheDir`, which the system may reclaim; the manifest and segments are
  not durable across a low-storage event mid-merge.
- `EntryAbility` registers `avoidAreaChange` and never releases it, and drops
  the promise from `setWindowLayoutFullScreen` - the same boilerplate defect
  that recurs across this industry's samples.

## Pitfalls

- **`HW-13-0032`** (B/medium, probable): `MainPage` builds its UIContext with
  `new UIContext()` - twice - and reads `getHostContext()` off the detached
  instance, so `cacheDir`, the media library helper and the toasts all hang off
  a context with no host. Fix: use `this.getUIContext()`, as the same file
  already does at line 289. `PhotoUtils.ets` has the import-time
  `AppStorage.get('uiContext')` variant of the same problem.
- **`HW-13-0051`** (B/medium, probable): thumbnail, uri and duration go into
  three parallel arrays filled from a sync path, an async `packToFile` callback
  and a `getAssets` try block. One failure or one reordering and taps, drags
  and durations all address the wrong clip. Fix: push a single
  `{uri, thumb, time}` record after awaiting the pack.
- **`HW-13-0058`** (D/high, probable): `getAssets` is called with no
  `requestPermissions` in `module.json5`; picker URIs do not grant query
  rights, so thumbnails and durations silently never load - which is also what
  triggers the array desync above. Fix: declare and request
  `READ_IMAGE_VIDEO`, or read metadata via `AVMetadataExtractor` on the picked
  fd.
- **`HW-13-0050`** (B/medium, probable): `video.txt` is reopened without
  `OpenMode.TRUNC` and the work directory is only cleaned after a successful
  save, so a shorter second selection concatenates leftover segments. Fix: add
  `TRUNC` and clear the work dir at the start of each run.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `Navigation`, `hideNavBar`, `NavPathStack.pushPathByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - the routerMap / buildFunction registration flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - the splash image
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions` and what the returned URIs do and do not allow
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getAssets` and its permission requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `READ_IMAGE_VIDEO`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `MEDIA-20` - the same merge editor without the splash, carrying the same three data-layer defects
