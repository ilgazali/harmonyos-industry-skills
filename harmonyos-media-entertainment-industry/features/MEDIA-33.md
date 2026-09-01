---
id: MEDIA-33
title: Mirror video playback - flip the surface container with a Y-axis rotate instead of touching the decoder
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/33_video_mirroring.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_mirroring-0000002344967801
sample: huawei_industry_tree/13_media_entertainment/downloads/Mirror.zip
kits: ["@kit.AVSessionKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BackgroundTasksKit", "@kit.BasicServicesKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit"]
apis: [avSession, backgroundTaskManager, base, hilog, image, media, wantAgent, window]
permissions: [ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [entry]
findings: [HW-13-0012, HW-13-0046, HW-13-0072, HW-13-0099, HW-13-0103]
status: verified-with-fixes
---

## When to use

Load this card when a player needs a **mirror / flip toggle** - the control that
turns a horizontally reversed source (a front-camera recording, a shared
whiteboard feed, a dance tutorial the user wants to follow "as in a mirror")
back into the natural view. The pattern is one line: bind the surface
container's `rotate` to a state number and set it to 180 about the Y axis.

The transferable idea is that **the flip never reaches the media pipeline**.
The video keeps decoding into the same `XComponent` surface; only the render
node that composites that surface is transformed. Nothing is re-encoded, no
`AVPlayer` state changes, the toggle is free at any playback position, and the
same trick works on an `Image`, a `Canvas` or a whole page.

The rest of the sample is a landscape `AVPlayer` video player wired to an
`AVSession` and a background continuous task. That scaffolding is what makes
the mirror worth shipping - and it is also where all three defects sit. Read
`HW-13-0072` before you copy the session callbacks.

## Feature checklist

- A full-screen landscape player over an `XComponent` surface, sourced from
  rawfile videos enumerated at startup.
- A mirror button in the bottom control bar, white when off and blue when on.
- Tapping it flips the video horizontally in place, without a re-prepare and
  without losing the playback position.
- A small hint image fades in for two seconds when the mirror is switched on,
  and does not reappear when it is switched off.
- Tapping again restores the natural orientation.
- Progress slider, play/pause, previous/next, and elapsed/total time.
- An `AVSession` publishes metadata and playback state so the system media
  controls drive the same player.
- A continuous task in `audioPlayback` mode keeps playback alive in the
  background.

## Architecture

One `entry` module. There is no model layer - the file list, the player, the
session and the UI all live in the page.

```
entry/src/main/ets
├── common/Constants.ets            layout numbers + ROTATE_ANGLE (180) / DEFAULT_ANGLE (0)
├── entryability/EntryAbility.ets   forces light mode, full-screen layout, hides the nav indicator
├── pages/MainPage.ets              @Entry, 763 lines: player + session + task + controls + mirror
└── util/
    ├── GlobalContext.ets           a string-keyed Map used to hand the window object to the page
    └── Logger.ets                  hilog wrapper
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the mirror is a property of the
*container*, not of the player. `MainPage` wraps the `XComponent` (plus the
overlay play button and the right-half tap target) in a `Stack`, and that one
`Stack` carries `.rotate({ y: 2, angle: this.angle, perspective: 200 })`.
Because the transform is applied at composition time, the flip is instant, it
survives seeks and track changes, and it costs one state assignment. The
alternative implementations - re-configuring the surface transform, or asking
the codec for a flipped output - all require tearing the player down.

**The design decision worth avoiding** is the 763-line `@Entry` struct.
Playback state, session callbacks, background-task lifecycle, time formatting
and the whole control bar are in one file, which is why the same
`avPlayer`/`avSession` boilerplate has drifted apart across this industry's
samples and picked up the systematic defects below.

## Implementation steps

1. **Put the surface in a container you can transform.** An `XComponent` of
   `XComponentType.SURFACE` inside a `Stack`; call
   `setXComponentSurfaceRect` in `onLoad` and read `getXComponentSurfaceId()`
   for `avPlayer.surfaceId`.
2. **Keep the flip as one number.** `@State angle: number = 0`, set to 180 to
   mirror and back to 0 to restore. Do not store a matrix.
3. **Rotate about Y, with a perspective.** `rotate({ y: 2, angle,
   perspective: 200 })`. The `y` value is a direction vector component - any
   non-zero value selects the Y axis - and `perspective` is the camera
   distance that keeps the half-way frames from looking flat.
4. **Drive the button icon off a second boolean** (`isMirror`) so the control
   can show blue/white without reading the angle back.
5. **Show the hint with a `setTimeout` you own.** `clearTimeout` the previous
   handle before arming a new one, and clear it in `aboutToDisappear`.
6. **Wire `AVSession` last, and honour the units.** `on('seek')` delivers an
   absolute position in milliseconds; do not feed it to a relative-seconds
   helper (`HW-13-0072`).
7. **Close every raw file descriptor you open.** `getRawFdSync` hands out HAP
   descriptors that survive for the process lifetime unless you call
   `closeRawFd` (`HW-13-0012`).
8. **Track the "wait for prepared" interval in a field** and clear it on
   teardown and on error, instead of leaving a 10 Hz timer per tap
   (`HW-13-0046`).

## Verified snippets

All snippets are from `Mirror.zip`. Corrected forms are marked.

**The flip - `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@State angle: number = 0;
@State isMirror: boolean = false;
@State showMirrorDialog: boolean = false;
private dialogTimeout?: number = undefined;

@Builder
VideoPlayer() {
  Stack({ alignContent: Alignment.Bottom }) {
    Stack() {
      if (!this.isPlay) {
        Image($r('app.media.ic_public_play'))
          .rotate({ angle: this.angle })            // Z rotation, cancels the mirror for a symmetric glyph
          .zIndex(4)
          .onClick(() => { this.play(); });
      }
      Column() {
        XComponent({ id: '', type: XComponentType.SURFACE, libraryname: '',
                     controller: this.xComponentController })
          .onLoad(() => {
            this.xComponentController.setXComponentSurfaceRect({
              surfaceWidth: Constants.XCOMPONENT_SURFACE_WIDTH,
              surfaceHeight: Constants.XCOMPONENT_SURFACE_HEIGHT
            });
            this.surfaceID = this.xComponentController.getXComponentSurfaceId();
          })
      }
      .zIndex(1)
      .onClick(() => { this.playOrPause(); });
    }
    .height(Constants.FULL_PERCENT)
    .rotate({ y: 2, angle: this.angle, perspective: Constants.ROTATE_CAMERA_DISTANCE });

    this.PlayControl();
    if (this.showMirrorDialog) { this.MirrorDialog(); }
  }
  .backgroundColor(Color.Black);
}
```

**Three arguments carry the whole feature.** `y: 2` names the rotation axis -
the triple `{x, y, z}` is a direction vector, so any non-zero `y` with `x` and
`z` unset rotates about the vertical axis and gives a horizontal mirror at 180.
`angle` is the only state. `perspective: 200` sets the camera distance; without
it the intermediate frames of the flip project orthographically and the video
reads as a squash rather than a turn.

Note what happens to the overlay play button. It sits *inside* the rotated
`Stack`, so it is mirrored too, and it carries its own `.rotate({ angle })`
with no axis - which ArkUI reads as a Z rotation. A 180° Z turn on top of a
Y mirror is a vertical flip, and the play triangle is vertically symmetric, so
it comes out pointing the right way. **That cancellation only works for
symmetric artwork**; a text label or a logo in the same position would come out
upside down.

**The toggle - same file** (as shipped)

```typescript
Image(this.isMirror ? $r('app.media.mirror_blue') : $r('app.media.mirror_white'))
  .height(Constants.IMAGE_18)
  .width(Constants.IMAGE_18)
  .onClick(() => {
    if (!this.isMirror) {
      this.isMirror = true;
      this.angle = Constants.ROTATE_ANGLE;      // 180
      this.dialogDuration();
    } else {
      this.isMirror = false;
      this.angle = Constants.DEFAULT_ANGLE;     // 0
    }
  });

dialogDuration() {
  this.showMirrorDialog = true;
  clearTimeout(this.dialogTimeout);             // never stack two hint timers
  this.dialogTimeout = setTimeout(() => {
    this.showMirrorDialog = false;
    this.dialogTimeout = undefined;
  }, 2000);
}
```

`ROTATE_ANGLE` and `DEFAULT_ANGLE` are constants rather than literals, which is
what lets the same handler drive a 90° rotate or a vertical flip by changing
one file. The hint is a plain `if` in the `Stack` with
`TransitionEffect.OPACITY.animation({ duration: 500, curve: Curve.Ease })` -
appearance and disappearance both animate because the branch is inside a
container that re-lays out. The `clearTimeout` before `setTimeout` is the
correct shape: a fast double toggle must not leave two timers racing to hide a
hint that is already gone.

**The AVSession seek - same file** (corrected, see `HW-13-0072`)

```typescript
curAvSession.on('seek', (time: number) => {
  logger.info(`AVPlayerVideo AVSession on seek entry time : ${time}`);
  this.setSeekAbsolute(time);              // FIX: sample calls setSeek(time), a RELATIVE-seconds helper
});

// the shipped helper, unchanged: `addSecond` is a delta in SECONDS
setSeek(addSecond: number, curTime?: number) {
  if (!this.avPlayer || this.OPERATE_STATE.indexOf(this.avPlayer.state) === -1) {
    return;
  }
  if (!curTime) {
    curTime = this.avPlayer.currentTime;
  }
  let curMillSeconds: number = curTime + addSecond * 1000;
  curMillSeconds = Math.min(Math.max(curMillSeconds, 0), this.avPlayer.duration);
  this.currentTime = curMillSeconds * this.durationTime / this.avPlayer.duration;
  this.currentStringTime = this.secondToTime(Math.floor(curMillSeconds / 1000));
  this.avPlayer.seek(curMillSeconds, media.SeekMode.SEEK_PREV_SYNC);
}

// FIX: the absolute path the session command actually needs
setSeekAbsolute(positionMs: number) {
  if (!this.avPlayer || this.OPERATE_STATE.indexOf(this.avPlayer.state) === -1) {
    return;
  }
  const clamped = Math.min(Math.max(positionMs, 0), this.avPlayer.duration);
  this.currentTime = clamped * this.durationTime / this.avPlayer.duration;
  this.currentStringTime = this.secondToTime(Math.floor(clamped / 1000));
  this.avPlayer.seek(clamped, media.SeekMode.SEEK_PREV_SYNC);
}
```

**`AVSession`'s `seek` event is an absolute position in milliseconds.** The
sample routes it into `setSeek`, whose parameter is a delta in *seconds* and
which adds `addSecond * 1000` to the current position. Dragging the system
media controller to 30 s therefore asks for `currentTime + 30000` ms, which
`Math.min` clamps to the duration: every seek from outside the app jumps to the
end. `VideoScreenshot` has the identical bug, which is why this is filed as a
systematic - Mirror is the second instance.

**Raw descriptors - same file** (corrected, see `HW-13-0012`)

```typescript
private openedRawPaths: string[] = [];       // FIX: remember what we opened

initFiles() {
  let fileList: string[] = this.context.resourceManager.getRawFileListSync('video');
  fileList.forEach((fileStr: string) => {
    const rawPath = `video/${fileStr}`;
    // fd is the HAP package descriptor, offset/length delimit the entry inside it
    let fileDescriptor = this.context.resourceManager.getRawFdSync(rawPath);
    this.openedRawPaths.push(rawPath);
    this.videoFiles.push({
      fd: fileDescriptor.fd, offset: fileDescriptor.offset, length: fileDescriptor.length
    });
  });
  // FIX: the sample also opens every rawfile under 'audio/' into `audioFiles`,
  //      which nothing ever reads. Drop that loop instead of leaking it.
  this.sourceFiles = this.videoFiles;
}

aboutToDisappear(): void {
  // ... player and session teardown, then: ...
  this.openedRawPaths.forEach((rawPath: string) => {
    this.context.resourceManager.closeRawFd(rawPath);        // FIX: absent in the sample
  });
  this.openedRawPaths = [];
}
```

`getRawFdSync` returns a descriptor onto the HAP itself, shared by every
rawfile in the package; the `offset`/`length` pair is what selects the entry.
Those descriptors are **not** released by `avPlayer.release()` - the pairing is
`getRawFd` / `closeRawFd` on the same `resourceManager`, and it has to happen
after the player that used them is released. The sample opens every video *and*
every audio rawfile at startup and closes none; `audioFiles` is pushed to and
never read at all.

## Permissions & config

```json5
"abilities": [
  {
    "name": "EntryAbility",
    "orientation": "landscape",
    "backgroundModes": ["audioPlayback"]
  }
],
"requestPermissions": [
  { "name": "ohos.permission.KEEP_BACKGROUND_RUNNING" }
]
```

- `KEEP_BACKGROUND_RUNNING` is `system_grant`, so no `reason` or `usedScene` is
  required and no dialog is shown.
- The declared `backgroundModes` must contain the mode passed to
  `startBackgroundRunning`; here both say `audioPlayback`, which is correct for
  a video player whose audio keeps running when the screen is off.
- `orientation: "landscape"` is set on the ability, not by the page, so the
  surface geometry is fixed and the hardcoded 1920x1080 surface rect is safe.
- `EntryAbility` pins `COLOR_MODE_LIGHT` and hides the navigation indicator; the
  window object is stashed in `GlobalContext` for the page to read.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `deviceTypes` is `phone`, `tablet`, `2in1`, but the surface rect is a fixed
  1920x1080 and the control bar uses a hardcoded 135 vp side padding - the
  layout is tuned for one landscape phone width.
- The full-screen button (`app.media.all_screen`) has an empty `onClick`. It is
  decoration.
- The mirror is not persisted: it resets to 0 on every page entry and is not
  reapplied after `goToNextOrPre`.
- The Z-rotation trick that keeps the overlay play icon upright only works
  because the glyph is vertically symmetric. Any asymmetric overlay inside the
  mirrored `Stack` needs its own Y-axis counter-rotation.
- `windowClass` is fetched from `GlobalContext` at field-initialiser time, so
  the page cannot be loaded before `onWindowStageCreate` has run.

## Pitfalls

- **`HW-13-0072`** (B/medium, confirmed): the `AVSession` `seek` command
  delivers an absolute millisecond position but is passed to `setSeek`, a
  relative-seconds helper that computes `currentTime + time * 1000` and clamps
  to the duration - so every seek from the system media controller jumps to the
  end of the video. `VideoScreenshot` has the same code; Mirror is the second
  instance. Fix: give the session command its own absolute-position path.
- **`HW-13-0012`** (B/low, confirmed): `resourceManager.getRawFdSync` is called
  for every rawfile video *and* every rawfile audio in `initFiles`, and
  `closeRawFd` is never called - the `audioFiles` array is not even read. Raw
  HAP descriptors then accumulate for the process lifetime. Same defect across
  seven media samples. Fix: pair every `getRawFd` with `closeRawFd` after the
  player is released, and delete the unused audio loop.
- **`HW-13-0046`** (B/low, confirmed): `iconOnclick` starts a bare
  `setInterval(..., 100)` to wait for `flag` to become true, keeps the handle in
  a local, and clears it only when the flag flips. A prepare that never
  succeeds leaves a 10 Hz timer running for the life of the page, and every tap
  spawns another one, each of which will later call `play()`. Same pattern in
  `SpeedPlay` and `BufferedProgressBar`. Fix: keep the id in a field, guard
  against a second instance, clear it in `aboutToDisappear` and in the error
  callback.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `rotate`, the `{x, y, z}` axis vector, `perspective`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `surfaceId`, `fdSrc`, the state machine, `seek` and `SeekMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/js-apis-avsession.md` - `createAVSession`, `setAVMetadata`, `setAVPlaybackState`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-avsession
- `documentation/harmonyos-references/04_media/arkts-apis-avsession-i.md` - `AVPlaybackState` and the `seek` callback's units
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-avsession-i
- `documentation/harmonyos-references/02_application-framework/js-apis-resourceschedule-backgroundtaskmanager.md` - `startBackgroundRunning` and `BackgroundMode.AUDIO_PLAYBACK`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resourceschedule-backgroundtaskmanager
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-wantagent.md` - the `WantAgent` the continuous task needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-wantagent
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - declaring `backgroundModes` to match the requested mode
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `KEEP_BACKGROUND_RUNNING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `MEDIA-32` - the same player scaffolding, and the first instance of `HW-13-0072`
- `MEDIA-17` - the same 100 ms "wait for prepared" polling defect
