---
id: MEDIA-17
title: Playback speed control - a speed menu plus hold-to-fast-forward on the right half of the video
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/17_speed_play.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/speed_play-0000002321158281
sample: huawei_industry_tree/13_media_entertainment/downloads/SpeedPlay.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [base, hilog, media, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0045, HW-13-0046, HW-13-0012]
status: verified-with-fixes
---

## When to use

Load this card when a video surface needs **two kinds of speed control at once**:
a sticky one the user picks from a menu (0.5x … 3x, remembered until changed)
and a transient one bound to a gesture (hold to fast-forward, release to go
back). Nearly every short-video and long-form player ships both, and they
interact: the transient one has to know what to restore to.

The pattern is a single `AVPlayer` driven through `setSpeed`, a state
whitelist that gates every command, and one number - `speedIndex` - that is
the source of truth for "the speed the user actually chose". The gesture
temporarily overrides the player but never touches that number, so the release
handler can restore from it.

It generalises past speed. Any transient override on a media surface - hold to
mute, hold to boost brightness, hold to scrub - wants exactly this shape:
never mutate the persisted preference inside the gesture, read it back on
release. **The trap is initialising the persisted value to something other
than the player's real starting state** (`HW-13-0045`); read that finding
before copying the sample.

## Feature checklist

- A landscape full-screen player over an `XComponent` surface, driven by
  `AVPlayer` from rawfile video assets.
- A 倍速 (multi-speed) button in the control bar opens a vertical menu of five
  presets: 0.5x, 1x, 1.5x, 2x, 3x.
- Picking a preset applies it immediately, updates the button label, closes the
  menu, and shows a 已切换N倍速播放 (switched to Nx playback) toast for two
  seconds.
- Long-pressing the right half of the video sets 2x and shows a
  倍速播放中 (playing at multi-speed) badge while the finger is down.
- Releasing the long press restores the speed the user last picked from the
  menu - **1x if they never opened it**.
- Tapping either half toggles play/pause; a tap while the media is still
  preparing queues the play until `prepared` arrives.
- A slider seeks, and current/total time strings track playback.

## Architecture

One `entry` module, one page. There is no view model and no data layer - the
page owns the player, the surface, the gesture and the control bar.

```
entry/src/main/ets
├── common/Constants.ets           layout literals only (percentages, margins, radii)
├── entryability/EntryAbility.ets  full-screen layout, avoid areas -> AppStorage, window into GlobalContext
├── pages/MainPage.ets             608 lines: player, surface, gestures, control bar, speed menu
└── util
    ├── GlobalContext.ets          a Map<string, Object> singleton, used only to pass the window
    └── Logger.ets                 hilog wrapper
```

The documented 工程目录 matches the zip exactly, including the absence of an
`entrybackupability` directory.

**The design decision worth copying** is the `OPERATE_STATE` whitelist.
`AVPlayer` is a state machine, and `setSpeed`, `seek` and `play` all throw if
called from the wrong state. Rather than checking inline at every call site,
the page declares the legal set once and every command routes through a guard:

```typescript
private readonly OPERATE_STATE: Array<string> = ['prepared', 'playing', 'paused', 'completed'];
```

`setSpeed`, `setSeek` and `play` each start with the same
`this.OPERATE_STATE.indexOf(this.avPlayer.state) === -1` early return. That is
what makes the long-press gesture safe to fire on a surface that has not
finished preparing: the gesture calls `setSpeed`, the guard drops it, nothing
throws. Copy the whitelist, not a scatter of `try`/`catch`.

The second structural choice worth noting is negative: **the speed the gesture
restores is stored in the page, not read back from the player.** That is the
right call (the player has no "user preference" concept) but it makes the
initial value load-bearing, which is exactly where the sample goes wrong.

Unlike the five samples in `HW-13-0005`, this one *does* tear down properly:
`aboutToDisappear` unregisters all four listeners, releases the player and
turns off the window listener. What it does not release are the rawfile
descriptors (`HW-13-0012`).

## Implementation steps

1. **Collect the rawfile videos once** with `getRawFileListSync('video')` and
   map each entry through `getRawFdSync` into an `AVFileDescriptor`. Keep the
   `RawFileDescriptor` objects so you can call `closeRawFd` in teardown
   (`HW-13-0012`).
2. **Create the player in `aboutToAppear`,** register the callbacks *before*
   assigning `fdSrc` - the `initialized` state fires as soon as the source is
   set, and it is the callback that calls `prepare()`.
3. **Declare the legal state set once** and gate `setSpeed`, `seek` and `play`
   behind it.
4. **Keep the chosen speed in one number** and initialise it to `1`, matching
   the `setSpeed(SPEED_FORWARD_1_00_X)` the `prepared` branch actually issues
   (`HW-13-0045`).
5. **Bind the long press to an overlay `Column`** covering the right half, with
   `zIndex` above the `XComponent` and `margin({ left: '50%' })` as the
   geometry.
6. **Set 2x in `onAction` and restore from the stored index in
   `onActionEnd`** - never write the stored index from inside the gesture.
7. **Show the transient badge from a boolean** flipped in the same two
   handlers, so the badge lifetime is exactly the gesture lifetime.
8. **When a play request arrives before `prepared`, poll a single interval and
   clear it on teardown and on failure** (`HW-13-0046`).

## Verified snippets

All snippets are from `SpeedPlay.zip`. Corrected forms are marked.

**The two-speed gesture — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-13-0045`)

```typescript
@State speedIndex: number = 1;          // FIX: shipped value is 2

// the right-half overlay, above the XComponent
Column()
  .width(Constants.FIFTY_PERCENT)
  .height(Constants.FULL_PERCENT)
  .margin({ left: Constants.FIFTY_PERCENT })
  .zIndex(2)
  .onClick(() => {
    this.playOrPause();
  })
  .gesture(
    LongPressGesture({ repeat: true })
      .onAction(() => {
        this.isDialog = true;                                       // 倍速播放中 badge on
        this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_2_00_X);
      })
      .onActionEnd(() => {
        this.isDialog = false;
        switch (this.speedIndex) {
          case 0.5:
            this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_0_50_X);
            break;
          case 1.0:
            this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_1_00_X);
            break;
          case 1.5:
            this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_1_50_X);
            break;
          case 2.0:
            this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_2_00_X);
            break;
          case 3.0:
            this.setSpeed(media.PlaybackSpeed.SPEED_FORWARD_3_00_X);
            break;
          default:
            hilog.error(0x0000, 'TAG', 'speed error');
            break;
        }
      })
  );
```

**Three things carry the design.** The gesture lives on a transparent `Column`
rather than on the `XComponent`, because a surface component should not carry
hit-test logic; the overlay is pure geometry (right half, `zIndex(2)`) and can
be moved without touching the player. `repeat: true` keeps `onAction` firing
while the finger is held, which matters if the player only reaches a legal
state part-way through the press - the guard drops the early calls and a later
repeat succeeds. And `onActionEnd` reads `speedIndex` rather than a captured
value, so a preset chosen *during* the press is honoured on release.

The `switch` is a lookup table written the long way: `speedIndex` is a plain
number (0.5 … 3.0) and `media.PlaybackSpeed` is an enum of opaque constants,
so some mapping has to exist. What must not exist is the shipped initial value
of `2`. The `prepared` handler explicitly issues
`setSpeed(SPEED_FORWARD_1_00_X)`, so playback genuinely starts at 1x while the
page believes it is at 2x. A user who long-presses before ever opening the
speed menu releases into permanent 2x - and the doc's own comment on this
handler is 松手恢复原速 (release restores the original speed).

**The sticky speed and its toast — same file** (as shipped)

```typescript
@Builder
SpeedControl() {
  Column() {
    this.SpeedControlView($r('app.string.0point5Speed'), media.PlaybackSpeed.SPEED_FORWARD_0_50_X, 0.5,
      $r('app.string.0point5Speed'));
    this.SpeedControlView($r('app.string.1Speed'), media.PlaybackSpeed.SPEED_FORWARD_1_00_X, 1.0,
      $r('app.string.multiSpeed'));
    this.SpeedControlView($r('app.string.1point5Speed'), media.PlaybackSpeed.SPEED_FORWARD_1_50_X, 1.5,
      $r('app.string.1point5Speed'));
    this.SpeedControlView($r('app.string.2Speed'), media.PlaybackSpeed.SPEED_FORWARD_2_00_X, 2.0,
      $r('app.string.2Speed'));
    this.SpeedControlView($r('app.string.3Speed'), media.PlaybackSpeed.SPEED_FORWARD_3_00_X, 3.0,
      $r('app.string.3Speed'));
  }
  .zIndex(3)
  .reverse(true)                                  // 0.5x nearest the button, 3x furthest
  .margin({ left: Constants.SEVENTY_ONE_PERCENT, bottom: Constants.SIX_PERCENT })
  .width(Constants.COLUMN_WIDTH_32)
  .height(Constants.COLUMN_HEIGHT_100);
}

speedControl(setSpeed: number, speedIndex: number, multiSpeed: Resource) {
  this.setSpeed(setSpeed);
  this.speedIndex = speedIndex;                   // the value the gesture restores to
  this.multiSpeed = multiSpeed;                   // the control-bar label
  this.isSpeed = false;                           // close the menu
  this.showSpeedDialog = true;
  clearTimeout(this.dialogTimeout);               // one timer, restarted per pick
  this.dialogTimeout = setTimeout(() => {
    this.showSpeedDialog = false;
    this.dialogTimeout = undefined;
  }, 2000);
}
```

Each preset carries **three** values, not one: the enum for the player, the
number for the restore path, and the resource for the button label. Passing
them together through one `@Builder` parameter list is what keeps the five
rows declarative - adding 4x is one more line.

Note that 1x maps its label to `$r('app.string.multiSpeed')` (the generic 倍速
caption) rather than a "1x" string, so the control bar shows the neutral label
at normal speed. And `clearTimeout` before every `setTimeout` is the detail
that makes rapid preset switching behave: without it the second pick's toast
would be cancelled by the first pick's expiring timer. This one *is* cleared in
`aboutToDisappear` - which makes the interval below the odd one out.

**Waiting for `prepared` before playing — same file** (corrected, see `HW-13-0046`)

```typescript
private prepareInterval?: number = undefined;     // FIX: shipped code uses a local

iconOnclick() {
  if (this.isPlay === true) {
    this.avPlayer?.pause();
    this.isPlay = false;
    this.isOpacity = false;
    return;
  }
  if (this.flag === true) {                       // flag is set true in the 'prepared' branch
    this.avPlayer?.play();
    this.isPlay = true;
    this.isOpacity = true;
  } else {
    if (this.prepareInterval !== undefined) {     // FIX: no guard in the sample - taps stack
      return;
    }
    // The scheduled task determines whether the video loading is complete.
    this.prepareInterval = setInterval(() => {
      if (this.flag === true) {
        this.avPlayer?.play();
        this.isPlay = true;
        this.isOpacity = true;
        clearInterval(this.prepareInterval);
        this.prepareInterval = undefined;
      }
    }, 100);
  }
}

aboutToDisappear(): void {
  clearTimeout(this.dialogTimeout);
  clearInterval(this.prepareInterval);            // FIX: absent in the sample
  if (this.avPlayer) {
    this.avPlayer.off('timeUpdate');
    this.avPlayer.off('seekDone');
    this.avPlayer.off('error');
    this.avPlayer.off('stateChange');
    this.avPlayer.release();
  }
}
```

**A 10 Hz poll with no owner is the defect.** In the shipped code
`intervalFlag` is a local `let`, so nothing outside the closure can reach it:
the only exit is `this.flag` turning true. If `prepare()` fails - a corrupt
asset, a surface that never arrives - the timer runs for the life of the page,
and every further tap on the play icon spawns another one, each of which will
later call `play()`. Promoting the id to a field costs two lines and gives you
both the re-entry guard and the teardown.

The `flag` boolean itself is worth keeping: it is set in the `prepared` branch
of `stateChange` and cleared by `reset()`, so it is a cheap "is the player
usable" signal that does not require reading `avPlayer.state` from the UI.

**Rawfile descriptors — same file** (corrected, see `HW-13-0012`)

```typescript
private rawFdPaths: string[] = [];                          // FIX: not kept in the sample

initFiles() {
  let fileList: string[] = this.context.resourceManager.getRawFileListSync('video');
  fileList.forEach((fileStr: string) => {
    // getRawFd returns {fd, offset, length}: fd is the HAP package fd,
    // offset the asset's offset inside it, length the playable length.
    let rawPath: string = `video/${fileStr}`;
    let fileDescriptor = this.context.resourceManager.getRawFdSync(rawPath);
    this.rawFdPaths.push(rawPath);                          // FIX
    let avFileDescriptor: media.AVFileDescriptor = {
      fd: fileDescriptor.fd,
      offset: fileDescriptor.offset,
      length: fileDescriptor.length
    };
    this.videoFiles.push(avFileDescriptor);
  });
  this.sourceFiles = this.videoFiles;
}

// in aboutToDisappear, after avPlayer.release():
this.rawFdPaths.forEach((rawPath: string) => {
  this.context.resourceManager.closeRawFdSync(rawPath);     // FIX: pair every getRawFd
});
```

A rawfile descriptor is a window into the HAP package, and the resource manager
hands out a real fd for it. `AVPlayer.fdSrc` copies the three numbers but does
not take ownership, so the descriptor stays open until the app closes it. This
sample opens one per video in the `video/` rawfile directory at page entry, so
the leak is bounded and small - but the same omission in `MEDIA-18`, which
re-opens a descriptor on every swipe, is unbounded. Close *after* the player is
released, never before: the player may still be reading.

## Permissions & config

**None declared.** The sample has no `requestPermissions` array at all - the
video assets are rawfiles inside the HAP, so nothing user-grant is needed.

Two `module.json5` entries do shape the feature:

```json5
"orientation": "landscape",
"backgroundModes": ["audioPlayback"]
```

`orientation: "landscape"` is why the geometry can assume a wide surface and
hardcode `surfaceWidth: 2280, surfaceHeight: 1080` in the `XComponent`'s
`onLoad`. Note that `backgroundModes: ["audioPlayback"]` is declared without
the matching `ohos.permission.KEEP_BACKGROUND_RUNNING` and without any
`backgroundTaskManager` call, so the declaration does nothing here - do not
copy it as evidence that background playback is configured.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` in the zip is
  `6.0.0(20)`, so unlike `MEDIA-18` the constraint text and the build profile
  agree here.
- The surface rect is fixed at 2280x1080 px. Any aspect ratio other than 19:9
  is letterboxed by the `XComponent` itself, not by `objectFit` - there is no
  `startRenderFrame`-driven resize as in `MEDIA-18`.
- The 上一个 / 下一个 (previous / next) icons in the control bar are decoration:
  they carry no `onClick`. `currentIndex` and `sourceFiles` exist for
  multi-video playback but nothing advances them.
- `stopPlay()` and `setSeek()` are defined and never called.
- The speed menu has no visual indication of which preset is active; the
  control-bar label is the only feedback after the toast expires.
- The window handle is passed from the ability to the page through the
  `GlobalContext` map singleton, read in a field initialiser
  (`GlobalContext.getContext().getObject('windowClass') as window.Window`). It
  works because `onWindowStageCreate` stores it before `loadContent`, but the
  cast hides a possible `undefined`, and `aboutToDisappear` guards the
  `off('windowSizeChange')` call with a `try`/`catch` for exactly that reason.

## Pitfalls

- **`HW-13-0045`** (B/medium, confirmed): `speedIndex` is initialised to `2`
  while the `prepared` handler actually starts playback at 1x, so the first
  long-press release leaves the video permanently at 2x - the opposite of the
  doc's 松手恢复原速. Same defect in the `VideoScreenshot` sample
  (`VideoPage.ets:56`), two instances in this industry. Fix: initialise
  `speedIndex = 1`.
- **`HW-13-0046`** (B/low, confirmed): the "wait for prepared" 100 ms
  `setInterval` is held in a local variable, cleared only when `this.flag`
  flips, and never touched by `aboutToDisappear`. A failed prepare leaves a
  10 Hz timer running for the life of the page and each further tap adds
  another, all of which eventually call `play()`. Same shape in
  `BufferedProgressBar` and `VideoScreenshot`/`Mirror`. Fix: keep the id in a
  field, guard re-entry, clear on teardown and on error.
- **`HW-13-0012`** (B/low, confirmed): `getRawFdSync` is called once per video
  in `initFiles` and no `closeRawFd` exists anywhere in the project. Bounded
  here (one fd per rawfile at page entry) but part of a seven-sample pattern
  across this industry, unbounded in the feed-style samples. Fix: keep the
  `RawFileDescriptor`s and close them after `avPlayer.release()`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `AVPlayer`, `setSpeed`, `fdSrc`, the state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-media-e.md` - the `PlaybackSpeed` enum values
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-e
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-single-gesture.md` - `LongPressGesture`, `repeat`, `onAction` vs `onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-single-gesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE`, `setXComponentSurfaceRect`, `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileListSync`, `getRawFdSync`, `closeRawFd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/02_media/video-playback.md` - the prescribed AVPlayer lifecycle, ending in `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `MEDIA-18` - the same rawfile-descriptor omission, unbounded in a scrolling feed
