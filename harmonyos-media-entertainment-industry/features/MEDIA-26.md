---
id: MEDIA-26
title: Seamless list-to-detail video - move one AVPlayer's surfaceId between two XComponents
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/26_video_transition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_transition-0000002349370885
sample: huawei_industry_tree/13_media_entertainment/downloads/XComponentTransition.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [base, hilog, media, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0060, HW-13-0005, HW-13-0053]
status: verified-with-fixes
---

## When to use

Load this card when **a video is already playing in a feed and tapping it must
open a detail page without the picture stopping, restarting or flashing black**.
The pattern: keep exactly one `AVPlayer` for the whole app, give it a new
`surfaceId` when the destination's `XComponent` is ready, and let the frames
follow. There is no second player, no handoff of a playback position, nothing
to re-prepare.

It generalises to any "the media outlives the page" navigation: a feed card
expanding to full screen, a mini player docking and undocking, a picture-in-
picture window returning to the page it came from. The invariant is the same -
the decoder is app state, the surface is page state, and only the surface
changes.

**The technique it teaches is `avPlayer.surfaceId` being writable after
`initialized`.** Most implementations assume the surface is fixed at prepare
time and rebuild the player instead. This sample shows the cheaper route, and
also shows the sharp edge: it rebinds the surface on every `timeUpdate` tick
rather than once, because there is no "surface changed" event to hook.

## Feature checklist

- The big card on the recommend tab autoplays a muted video as soon as its
  `XComponent` loads.
- A speaker icon toggles mute without interrupting playback.
- Scrolling the card out of view pauses; scrolling it back in resumes - but
  only while the home page is on top of the stack.
- Tapping the card pushes the detail page; the video keeps playing across the
  transition, at the same position, unmuted.
- The detail page has its own controls: play/pause, a seek slider, elapsed and
  total time, and a full-screen button that rotates the window to landscape.
- Tapping the video toggles a control overlay that auto-hides after 2 s.
- Back (gesture or the on-screen arrow) returns to the feed with the video
  still running in the card, and leaves landscape first if it was full screen.

## Architecture

One `entry` module. One `@Entry` page hosting a `Navigation`, one
`NavDestination` registered by name, and a single shared player class.

```
entry/src/main/ets
├── components
│  ├── Home.ets                 the home tab: search bar + five content tabs
│  ├── Introduce.ets            static "related videos" list on the detail page
│  ├── Recommend.ets            the recommend tab: big card + a 2-lane list of little cards
│  └── VideoComponent.ets       the feed card's XComponent + mute toggle + setWindowDirection()
├── constants/Constants.ets     tab labels, AppStorage key names, timings
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage, `new AVPlayer()` -> AppStorage
├── model
│  ├── BottomTabModel.ets       the four bottom tabs
│  ├── CardModel.ets            the little-card list data
│  ├── VideoDataModel.ets       VideoParams (the push payload) + videoList
│  └── VideoSourceModel.ets     @Observed video source
├── pages
│  ├── DetailPage.ets           the NavDestination: second XComponent, controls, back handling
│  └── HomePage.ets             @Entry: Navigation + the four bottom tabs
└── utils
   ├── AppRouter.ets            a static wrapper over one NavPathStack
   └── AVPlayer.ets             the shared player: static media.AVPlayer + surfaceID field
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `AVPlayer.ets` keeps the
`media.AVPlayer` in a **static** field and the target `surfaceID` in an
**instance** field. The decoder is global; the destination is a property that
any page may overwrite. `EntryAbility` puts one instance in `AppStorage` under
`'player'`, and both `VideoComponent` and `DetailPage` read the same object.
When the detail page's `XComponent` loads it calls `player.setSurfaceID(...)`
and does nothing else - no prepare, no seek, no play. The frames arrive at the
new surface on the next `timeUpdate`.

`AppRouter` is a second, smaller decision: a static facade over one
`NavPathStack`, so `Recommend` can ask `AppRouter.getPageIndex() === -1`
("are we still on the feed?") without holding a stack reference. That is what
stops the visibility-driven pause/resume in the feed from fighting the detail
page for control of the player.

## Implementation steps

1. **Create one player instance in the ability** and publish it:
   `AppStorage.setOrCreate('player', new AVPlayer())`, inside the
   `loadContent` callback.
2. **Read it lazily in consumers**, not in a field initializer that runs at
   component construction - the write happens in an async callback
   (`HW-13-0053`).
3. **Set the source from the feed card's `onLoad`,** never from
   `aboutToAppear`: `getXComponentSurfaceId()` only returns a real id once the
   surface exists, so the source assignment (which triggers `initialized`) must
   follow it.
4. **In `initialized`, assign `avPlayer.surfaceId` and call `prepare()`;** in
   `prepared`, call `play()` and apply the stored mute mode.
5. **Re-assign `surfaceId` on every `timeUpdate`** so a surface swap made while
   playing takes effect at the next tick without any explicit "rebind" call.
6. **On the detail page's `XComponent.onLoad`, call `setSurfaceID` only.**
   The state machine is already past `initialized`; touching it again would
   restart playback.
7. **Stash the feed card's surface id in `AppStorage`** before pushing, so the
   back handler can point the player at it again.
8. **On back, restore the surface and only resume if it was playing**
   (`HW-13-0060`); leave landscape first if the page is full screen.
9. **Release the player** when the app tears down - the sample never does
   (`HW-13-0005`).

## Verified snippets

All snippets are from `XComponentTransition.zip`. Corrected forms are marked.

**The feed card — `entry/src/main/ets/components/VideoComponent.ets`** (as shipped)

```typescript
@Component
export struct VideoComponent {
  @ObjectLink item: VideoSource;
  @State surfaceId: string = '';
  @StorageLink('muteMode') muteMode: boolean = false;
  player: AVPlayer = AppStorage.get(Constants.AVPLAYER_NAME) as AVPlayer;
  mXComponentController: XComponentController = new XComponentController();

  build() {
    Stack({ alignContent: Alignment.BottomStart }) {
      XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
        .onLoad(() => {
          this.surfaceId = this.mXComponentController.getXComponentSurfaceId();
          this.player.setSurfaceID(this.surfaceId);
          this.player.avPlayerFdSrc(this.getUIContext().getHostContext()!);   // sets fdSrc -> 'initialized'
        })
        .onClick(() => {
          AppStorage.setOrCreate(Constants.SURFACE_ID_KEY, this.surfaceId);   // remember where to come back to
          this.player.cancelMuteMode();
          this.muteMode = false;
          AppRouter.pushByName(Constants.DETAIL_PAGE_NAME,
            new VideoParams(this.item, this.currentTime, 10), () => {
            });
        });
      // ... shadow, view counts, and the mute toggle image
    }
    .width('100%')
    .height(210);
  }
}
```

**The order inside `onLoad` is the whole point.** `getXComponentSurfaceId()`
returns an empty string until the surface exists, so the id must be captured
first, handed to the player second, and only then may the source be set -
because assigning `fdSrc` is what drives the state machine into `initialized`,
and `initialized` is where `surfaceId` gets applied. Do it in `aboutToAppear`
instead and the player binds an empty surface: audio, no picture. (`MEDIA-28`
ships exactly that race.)

`AppStorage.setOrCreate(SURFACE_ID_KEY, this.surfaceId)` in `onClick` is the
return ticket. The card is about to be scrolled off or covered; storing its
surface id now is what lets the back handler restore it later without a
component reference.

**The player — `entry/src/main/ets/utils/AVPlayer.ets`** (as shipped)

```typescript
export class AVPlayer {
  private surfaceID: string = '';
  private static avPlayer: media.AVPlayer | null = null;
  private status: string = '';
  public currentTime: number = 0;
  public duration: number = 0;

  setAVPlayerCallback(avPlayer: media.AVPlayer) {
    avPlayer.on('timeUpdate', (time: number) => {
      this.currentTime = time;
      avPlayer.surfaceId = this.surfaceID;      // re-bind every tick: a swap takes effect here
    });
    avPlayer.on('durationUpdate', (time: number) => {
      this.duration = time;
    });
    avPlayer.on('stateChange', async (state: string) => {
      switch (state) {
        case 'initialized':
          avPlayer.surfaceId = this.surfaceID;  // first bind
          avPlayer.prepare();
          break;
        case 'prepared':
          avPlayer.play();
          if (AppStorage.get('muteMode') === true) {
            avPlayer.setMediaMuted(media.MediaType.MEDIA_TYPE_AUD, true);
          }
          break;
        case 'playing':
          this.status = state;
          AppStorage.set('isPlaying', true);
          break;
        case 'paused':
          this.status = state;
          AppStorage.set('isPlaying', false);
          break;
        case 'completed':
          avPlayer.stop();
          break;
        case 'stopped':
          avPlayer.prepare();                    // loop: re-prepare and play again
          break;
      }
    });
  }
}
```

**Two writes to `surfaceId`, doing different jobs.** The one under
`'initialized'` is the mandatory first bind, without which there is no picture
at all. The one inside `timeUpdate` is the transition mechanism: there is no
"surface changed" event, so the sample simply re-asserts the field on every
progress tick. Whoever called `setSurfaceID` most recently wins, within one
tick. It is blunt - a redundant assignment several times a second - but it
means neither page has to know when the other's surface became valid.

The `status` field mirrored into `AppStorage('isPlaying')` is what lets
`DetailPage` render the right play/pause glyph with a `@StorageProp`, and it is
also the value the back handler *should* be consulting (see below).

`completed -> stop -> prepare` is the sample's loop: the video restarts
forever. Note there is no `release()` anywhere in the project (`HW-13-0005`).

**The detail page — `entry/src/main/ets/pages/DetailPage.ets`** (corrected, see `HW-13-0060`)

```typescript
@StorageProp('isPlaying') isPlaying: boolean = true;
player: AVPlayer = AppStorage.get(Constants.AVPLAYER_NAME) as AVPlayer;
xComponentController: XComponentController = new XComponentController();

XComponent({ type: XComponentType.SURFACE, controller: this.xComponentController })
  .onLoad(() => {
    this.surfaceId = this.xComponentController.getXComponentSurfaceId();
    this.player.setSurfaceID(this.surfaceId);     // the entire handoff: one line, no prepare
  })

// 自定义返回逻辑，确保视频无缝转场 (custom back logic, to keep the transition seamless)
handleBackAction() {
  if (this.isLayoutFullScreen) {
    this.isLayoutFullScreen = false;
    setWindowDirection(window.Orientation.PORTRAIT, this.getUIContext().getHostContext()!);
  }
  this.player.setSurfaceID(AppStorage.get('surfaceID') as string);
  if (this.isPlaying) {                          // FIX: sample calls play() unconditionally
    this.player.play();
  }
  AppRouter.pop();
}
```

**`onLoad` here is a single `setSurfaceID` and nothing else** - that is what
"seamless" means in practice. The player is already `playing`; it does not know
a navigation happened. Compare with the feed card's `onLoad`, which also sets
the source: only the *first* surface to appear may touch the state machine.

`handleBackAction` reverses the same two facts: restore the feed card's
surface id from `AppStorage`, then resume. The shipped code calls
`this.player.play()` unconditionally, so a user who paused on the detail page
finds the feed card playing again the moment they go back - the pause cannot
survive the transition. The component already holds the answer in
`@StorageProp('isPlaying')`, fed by the player's own `playing`/`paused`
callbacks, so the guard costs one line.

Also note the order: leave landscape *before* popping, otherwise the feed
renders one frame rotated.

**The shared instance — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped, see `HW-13-0053`, `HW-13-0005`)

```typescript
windowStage.loadContent('pages/HomePage', (err) => {
  if (err.code) {
    hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
    return;
  }
  AppStorage.setOrCreate('player', new AVPlayer());     // written inside an async callback
  this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
  let windowClass: window.Window = windowStage.getMainWindowSync();
  windowClass.setWindowLayoutFullScreen(true).then(() => { /* ... */ });
  let avoidArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);
  avoidArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);
});
```

**This is the sample's structural hazard.** The write happens inside the
`loadContent` callback, while every consumer reads it in a field initializer -
`player: AVPlayer = AppStorage.get(Constants.AVPLAYER_NAME) as AVPlayer` in
`VideoComponent`, `Recommend` and `DetailPage`. Whether that `as` cast lands on
an object or on `undefined` depends on callback ordering against component
construction, which is not something the code controls. Fetch it lazily in
`aboutToAppear`, or set `AppStorage` before `loadContent`.

The same ability never releases the player, and neither does any page:
a project-wide grep finds zero `release()` calls (`HW-13-0005`). `AVPlayer`
holds native decoder and audio resources; the official playback guide ends
every lifecycle with `release()`.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - the video is
`product.mp4` inside the HAP, read through
`ctx.resourceManager.getRawFd('product.mp4')` and played via `fdSrc`. A real
feed would stream over the network and need `ohos.permission.INTERNET`.

Routing config: `"pages": "$profile:main_pages"` lists only
`pages/HomePage`, and `"routerMap": "$profile:route_map"` registers a single
destination:

```json
{ "name": "DetailPage",
  "pageSourceFile": "src/main/ets/pages/DetailPage.ets",
  "buildFunction": "detailPageBuilder" }
```

`deviceTypes` is `phone`, `tablet`, `2in1`. There is no `orientation` entry in
the ability, so the landscape switch is done at runtime through
`window.setPreferredOrientation`, which is the right call for a video that is
only sometimes full screen.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- One player means one video. The feed shows a single playable big card; the
  little cards are static images. Extending this to several simultaneously
  visible videos needs a player per surface, and the whole
  "move the surfaceId" trick stops applying.
- Rebinding on `timeUpdate` means the swap is only as fast as the progress
  callback. It is imperceptible at normal rates, but a paused player emits no
  `timeUpdate`, so a surface swap made while paused does not take effect until
  playback resumes.
- `Recommend` gates its visibility-driven pause/resume on
  `AppRouter.getPageIndex() === -1`. Any other page pushed onto the same stack
  silently disables that behaviour.
- Three of the four bottom tabs and four of the five content tabs are empty
  `TabContent()` blocks; the discussion tab on the detail page is empty too.
  This is a transition demo, not an app shell.
- `formatTime` derives hours, minutes and seconds from milliseconds with three
  `Math.floor` calls and no hour suppression, so a 12-second clip reads
  `00:00:12`.

## Pitfalls

- **`HW-13-0060`** (B/low, confirmed): `handleBackAction` calls
  `this.player.play()` unconditionally, so a pause made on the detail page
  never survives the back transition and the feed card always resumes. Fix:
  guard on the `@StorageProp('isPlaying')` the component already holds.
- **`HW-13-0005`** (B/medium, confirmed): the project contains no
  `release()` call at all - `AVPlayer.ets` creates the player and nothing ever
  frees it. It is one of five media samples in this industry with the same
  omission. Fix: `avPlayer.release()` in `onWindowStageDestroy` / the owning
  page's `aboutToDisappear`.
- **`HW-13-0053`** (B/medium, probable): the shared player is written to
  `AppStorage` inside the `loadContent` callback but read in component field
  initializers, so whether consumers get an object or `undefined` depends on
  evaluation order - a first-run initialization hazard. Fix: read it lazily, or
  publish it before `loadContent`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the state machine, `surfaceId`, `on('timeUpdate')`, `on('stateChange')`, `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE`, `onLoad`, `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowLayoutFullScreen`, `setPreferredOrientation`, `getWindowAvoidArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - the prescribed create → set source → prepare → play → release lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, `NavDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `MEDIA-28` - the same surface/source ordering, got wrong
