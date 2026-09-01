---
id: MEDIA-31
title: Drum pad - low-latency SoundPool samples fired together with a haptic pattern
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/31_drum.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/drum-0000002332280266
sample: huawei_industry_tree/13_media_entertainment/downloads/demo_ElecDrum.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["media.createSoundPool", "SoundPool.load", "SoundPool.play", "SoundPool.unload", "SoundPool.release", "vibrator.startVibration", "VibrateFromFile", "fs.open", "fs.openSync", "fs.closeSync", RelativeContainer, clickEffect, "@Styles", abilityAccessCtrl, audio, bundleManager, media, vibrator, window]
permissions: [ohos.permission.VIBRATE]
min_api: 20
modules: [entry]
findings: [HW-13-0070, HW-13-0071, HW-13-0015, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a tap has to produce **sound and vibration that feel like
one event**: a drum pad, a piano or launchpad app, a game controller overlay, a
keyboard with tactile keys, a slot-machine reel. The requirement that shapes the
code is latency - a drum that answers 80 ms late is not a drum - and that rules
out `AVPlayer`, which has to prepare a source per play.

The pattern is `SoundPool` for audio plus `vibrator.startVibration` with a
`file` source for haptics, fired **in parallel**. The sample's own comment
records why it is not using the obvious API:

```
// 短语audioHaptic 有问题，会出现音振不同步现象，可以采用sound+vibrate的方式
// (AudioHaptic has a problem - sound and vibration go out of sync - so use sound + vibrate instead)
```

`SoundPool` is the transferable half: it decodes a short clip once into a
managed pool and gives back a `soundId` that can be played repeatedly and
concurrently, with a priority so a burst of taps degrades gracefully instead of
queueing. The haptics half generalises to any app that wants a designed
vibration rather than the stock click - the JSON pattern format described in the
document is authored per pad, exactly like a sound effect.

**Two defects to know before adopting.** The permission helper returns the
opposite of the truth (`HW-13-0070`), and the app plays all three drums and
fires three haptic patterns at every cold start because "preload" was
implemented by calling the tap handler (`HW-13-0071`).

## Feature checklist

- A landscape, full-screen drum kit over a wooden-board background: one big
  drum, three small drums, four cymbals of two sizes.
- Every pad responds to a tap with its own sample and its own vibration
  pattern, together.
- Pads share three sound/haptic pairs: the big drum, the small drums, the
  cymbals.
- Tapping shows a press effect (`clickEffect` with a per-pad scale), so the pad
  visibly gives under the finger.
- Rapid, overlapping taps play overlapping sounds rather than cutting each other
  off.
- `ohos.permission.VIBRATE` is requested at startup if it has not been granted.
- The layout keeps clear of the status bar and navigation indicator through
  avoid-area padding.

## Architecture

One `entry` module, one page, four utility classes. No model layer - the pads
are literal `Image` components positioned by percentage.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   landscape, full screen, no system bars, avoid areas -> AppStorage
├── entrybackupability/
├── pages/ElecDrumIndex.ets         @Entry: the eight pads in a RelativeContainer + tapButton
└── utils/
    ├── HapticUtils.ets             vibrator.startVibration from a haptic JSON fd
    ├── Logger.ets                  hilog wrapper
    ├── PermissionsUtil.ets         check + request VIBRATE (a default-exported singleton)
    └── SoundPoolUtil.ets           the SoundPool singleton: init / load / play / stop / release
```

The documented tree matches the zip.

**The design decision worth copying** is firing the two effects as one
`Promise.all` rather than awaiting them in sequence:

```typescript
Promise.all([
  SoundPoolUtil.playSound(audioUri, 1.0, this.soundPoolUtil),
  HapticUtils.vibrateFile(hapticUri)
])
```

Sound and vibration are independent subsystems with independent latencies;
awaiting the sound before starting the motor adds the audio path's latency to
the haptic one, which is precisely the desync the sample's comment complains
about in `AudioHaptic`. Starting both in the same tick makes the perceived
offset the *difference* of the two latencies instead of their sum.

The second structural choice is that both assets are read from
`context.resourceDir`, not from `rawfile`. `SoundPool.load` wants an `fd://`
URI and `vibrator.startVibration` wants a `hapticFd`; `resourceDir` gives real
sandbox paths that `fs.open` accepts directly, whereas `rawfile` entries need
`resourceManager.getRawFd` and its offset/length triple. For a pad kit with a
handful of small assets, `resourceDir` is the shorter road.

**Worth avoiding**: `SoundPoolUtil` is a singleton whose `release()` is never
called - `aboutToDisappear` is an empty body. The pool, its five streams and its
three listeners outlive the page.

## Implementation steps

1. **Author one JSON haptic pattern per pad.** A pattern is a list of `Event`s
   with `Type` (`transient` for a hit), `StartTime`, and `Parameters.Frequency`
   / `Parameters.Intensity` - a drum wants a low frequency and high intensity, a
   cymbal the opposite.
2. **Put the `.wav` and `.json` files in `entry/src/main/resources/resfile`** so
   they are reachable as `context.resourceDir + '/' + name`.
3. **Create the pool once**: `media.createSoundPool(5, audioRendererInfo)` with
   `usage: audio.StreamUsage.STREAM_USAGE_MUSIC`. The first argument is the
   maximum number of simultaneous streams - it is what lets a drum roll overlap.
4. **Cache the `initPromise`** so concurrent callers of `init()` await the same
   creation instead of building a second pool.
5. **Register `loadComplete`, `playFinished` and `error` once**, right after
   creation.
6. **Preload without playing** (`HW-13-0071`). Call `loadSound` for each asset
   in `aboutToAppear`; never call the tap handler to "warm up".
7. **Close the sound fd from the `loadComplete` callback**, not from the
   `finally` after `load()` resolves (`HW-13-0015`) - the pool is still reading
   the descriptor.
8. **Close the haptic fd from the `startVibration` callback** for the same
   reason (`HW-13-0015`).
9. **Check the permission correctly**: `abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED`
   is `0`, so `authResults[i] === 0` means *granted*. Accumulate over all
   entries and return after the loop (`HW-13-0070`).
10. **Release the pool in `aboutToDisappear`**: stop the live streams, `unload`
    each `soundId`, `off` the three listeners, then `release()`.

## Verified snippets

All snippets are from `demo_ElecDrum.zip`. Corrected forms are marked.

**Startup and the pad handler — `entry/src/main/ets/pages/ElecDrumIndex.ets`** (corrected, see `HW-13-0071`)

```typescript
aboutToAppear(): void {
  this.soundPoolUtil.init().then(() => {
    // FIX: the sample calls this.tapButton(0/1/2) here, which plays all three
    // drums and fires three haptic patterns on every cold start.
    ['key_press_click.wav', 'key_press_delete.wav', 'key_press_modifier.wav']
      .forEach((name: string) => {
        this.soundPoolUtil.loadSound(`${this.context?.resourceDir}/${name}`);
      });
  }).catch(() => {
    Logger.error(`初始化失败`);
  });
  // 检查权限
  const permissions: Array<Permissions> = ['ohos.permission.VIBRATE'];
  PermissionsUtil.checkPermissions(permissions, this.context);
}

tapButton(index: number) {
  let audio = 'key_press_click.wav';
  let haptic = 'little_drum.json';
  switch (index) {
    case 0:
      audio = 'key_press_click.wav';
      haptic = 'big_drum.json';        // the big drum: low frequency, high intensity
      break;
    case 1:
      audio = 'key_press_delete.wav';
      haptic = 'backspace_key.json';   // the small drums
      break;
    case 2:
      audio = 'key_press_modifier.wav';
      haptic = 'space_key.json';       // the cymbals
      break;
  }
  const audioUri = this.context?.resourceDir + '/' + audio;
  const hapticUri = this.context?.resourceDir + '/' + haptic;
  // 短语audioHaptic 有问题，会出现音振不同步现象，可以采用sound+vibrate的方式
  Promise.all([
    SoundPoolUtil.playSound(audioUri, 1.0, this.soundPoolUtil),
    HapticUtils.vibrateFile(hapticUri)
  ]).then(results => {
    Logger.info('音频和振动播放成功');
  }).catch(() => {
    Logger.error(`播放失败`);
  });
}
```

**Preloading matters, and the sample was right to want it** - the first
`SoundPool.load` of a clip costs a file open and a decode, so an unwarmed first
tap is audibly late. It just implemented it by calling the tap handler, which
also plays the sound and vibrates. The corrected form keeps the warm-up and
drops the side effects: `loadSound` populates `soundMap` and is exactly the
expensive half.

The `switch` is the whole instrument mapping: eight pads, three
sound-plus-pattern pairs, chosen by an index the `onClick` passes in. Copying
that shape keeps a bigger kit declarative - the pads stay dumb `Image`s with a
`clickEffect` and one call.

**The haptic file — `entry/src/main/ets/utils/HapticUtils.ets`** (corrected, see `HW-13-0015`)

```typescript
public static vibrateFile(fileName: string): void {
  let file = fs.openSync(fileName, fs.OpenMode.READ_ONLY);
  try {
    vibrator.startVibration({
      type: 'file',
      hapticFd: { fd: file.fd }
    }, {
      id: 0,
      usage: 'touch'
    }, (error: BusinessError) => {
      fs.closeSync(file.fd);              // FIX: was in a finally, i.e. before this callback ran
      if (error) {
        Logger.error(`Failed to start vibration. Code: ${error.code}, message: ${error.message}`);
        return;
      }
      Logger.info('Succeed in starting vibration');
    });
  } catch (err) {
    fs.closeSync(file.fd);                // FIX: the synchronous-throw path still needs the close
    let e: BusinessError = err as BusinessError;
    Logger.error(`An unexpected error occurred. Code: ${e.code}, message: ${e.message}`);
  }
}
```

**`type: 'file'` is what makes the vibration designed rather than generic.** The
alternative, `type: 'time'`, is a plain duration; the file form hands the motor
a JSON pattern of `transient` and `continuous` events with per-event frequency
and intensity, which is how a big drum and a cymbal feel different through the
same actuator. `usage: 'touch'` classifies it so the system's haptic settings
treat it as feedback rather than as an alarm.

**The `finally` in the shipped code is a race, not cleanup.** `startVibration`
with a callback returns immediately and reads the descriptor afterwards, so
`fs.closeSync(file.fd)` in a `finally` closes the fd while the vibrator service
is still using it - the vibration fails intermittently, depending on timing.
Closing from the completion callback is the fix. `HW-13-0015` records the same
shape across five samples in this industry, `demo_ElecDrum` being an instance
twice over: here and in `SoundPoolUtil.loadSound`.

**The pool — `entry/src/main/ets/utils/SoundPoolUtil.ets`** (corrected, see `HW-13-0015`)

```typescript
private soundPool: media.SoundPool | null = null;
private soundMap: Map<string, number> = new Map();
private streamMap: Map<number, number> = new Map();
private fdMap: Map<number, number> = new Map();      // FIX: soundId -> fd, closed on loadComplete
private initPromise: Promise<void> | null = null;

init(): Promise<void> {
  if (this.initPromise) {
    return this.initPromise;                         // one pool, however many callers
  }
  this.initPromise = new Promise(async (resolve, reject) => {
    const audioRendererInfo: audio.AudioRendererInfo = {
      usage: audio.StreamUsage.STREAM_USAGE_MUSIC,
      rendererFlags: 0
    };
    try {
      this.soundPool = await media.createSoundPool(5, audioRendererInfo);
      this.registerListeners();
      resolve();
    } catch (error) {
      reject(error);
    }
  });
  return this.initPromise;
}

private registerListeners() {
  if (!this.soundPool) {
    return;
  }
  this.soundPool.on('loadComplete', (soundId: number) => {
    const fd = this.fdMap.get(soundId);              // FIX: close here, not in load()'s finally
    if (fd !== undefined) {
      fs.closeSync(fd);
      this.fdMap.delete(soundId);
    }
    Logger.info('音频加载完成, soundId: ' + soundId);
  });
  this.soundPool.on('playFinished', () => {
    Logger.info('音频播放完成');
  });
  this.soundPool.on('error', (error) => {
    Logger.error('发生错误: ' + error.message);
  });
}

public async loadSound(filePath: string): Promise<number> {
  if (!this.soundPool || this.soundMap.get(filePath)) {
    return -1;
  }
  const file = await fs.open(filePath, fs.OpenMode.READ_ONLY);
  const soundId = await this.soundPool.load('fd://' + (file.fd).toString());
  this.fdMap.set(soundId, file.fd);                  // FIX: shipped code closes in a finally here
  this.soundMap.set(filePath, soundId);
  return soundId;
}
```

**`createSoundPool(5, ...)` is the concurrency budget.** Five streams means five
overlapping hits; a sixth tap evicts by the `priority` field passed to `play`.
For a drum kit that is the right knob - the sample plays everything at
`priority: 1`, so the oldest stream loses, which is what a real kit does when a
new hit lands on a still-ringing pad.

**`load()` resolving is not the same as the clip being decoded.** The promise
gives back a `soundId`; the pool signals readiness by emitting `loadComplete`
for that id, and it is still reading the descriptor in between. The shipped
`finally` closes the fd immediately after `load()` resolves, so the first play
of a clip can fail depending on how the decode is scheduled. Keeping a
`soundId -> fd` map and closing on `loadComplete` is the smallest correct fix.

Note also that `loadSound` returns `-1` both for "already loaded" and for
"failed", and `play()` treats a falsy `soundId` as missing - a real `soundId` of
`0` would be dropped. Use `soundMap.has(...)` rather than truthiness if the pool
ever hands out `0`.

**The permission check — `entry/src/main/ets/utils/PermissionsUtil.ets`** (corrected, see `HW-13-0070`)

```typescript
public reqPermissionsFromUser(permissions: Array<Permissions>, context: Context | undefined) {
  let atManager = abilityAccessCtrl.createAtManager();
  // requestPermissionsFromUser会判断权限的授权状态来决定是否唤起弹窗
  return atManager.requestPermissionsFromUser(context, permissions).then((data) => {
    let grantStatus: Array<number> = data.authResults;
    let length: number = grantStatus.length;
    let allGranted: boolean = true;                  // FIX: accumulate instead of returning at i=0
    for (let i = 0; i < length; i++) {
      if (grantStatus[i] !== 0) {                    // FIX: 0 is PERMISSION_GRANTED, not denied
        allGranted = false;
        break;
      }
    }
    return allGranted;
  });
}
```

**The shipped body is `if (grantStatus[i] === 0) { return false; } else { return true; }`
inside the loop** - two errors stacked. `authResults[i] === 0` is
`PERMISSION_GRANTED`, so the helper answers `false` for a granted permission and
`true` for a refused one; and because both branches return, the loop never
reaches index 1 regardless. The only reason vibration still works in the sample
is that no caller looks at the return value: `checkPermissions` calls
`reqPermissionsFromUser` and discards the promise.

`checkPermissions` has the same accumulate-versus-assign shape as `TOUR-03`'s
`HW-09-0017` - it overwrites `applyResult` on every iteration, so the last
permission decides. With a single-element array that is harmless today and a
trap the moment a second permission is added. Accumulate there too.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.VIBRATE",
  }
]
```

- `VIBRATE` is `system_grant`, so the declaration alone is enough and no dialog
  will ever appear - which is why `HW-13-0070`'s inverted result stays invisible
  in this sample. `reason` and `usedScene` are not required for a
  `system_grant` permission and are correctly omitted.
- The ability is pinned to `"orientation": 'landscape'` - a drum kit is a
  landscape instrument, and the percentage-positioned pads assume that aspect
  ratio.
- `EntryAbility` calls `setWindowSystemBarEnable([])` for a bar-free stage,
  sets `setWindowLayoutFullScreen(true)`, and publishes the avoid-area heights
  into `AppStorage` **already converted with `px2vp`** - so the page's
  `padding({ top: this.topRectHeight, ... })` is in the right unit. Compare
  `MEDIA-29`, which stores raw px.
- The audio and haptic assets live in `resourceDir`, reached through
  `context.resourceDir`; nothing here uses `rawfile`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Vibration requires a device with a motor. On a device without one,
  `startVibration` fails and only the audio half of the pair is heard - the
  callback logs it, and nothing in the UI reflects it.
- The three bundled samples are keyboard sounds (`key_press_click.wav`,
  `key_press_delete.wav`, `key_press_modifier.wav`), not drum recordings, and
  two of the haptic patterns are keyboard patterns (`backspace_key.json`,
  `space_key.json`). Only `big_drum.json` is instrument-specific.
- `aboutToDisappear` is empty: the `SoundPool` singleton, its streams and its
  three listeners are never released. `release()` exists and is correct - it is
  simply not called. Note that `release()` also deliberately keeps
  `this.soundPool` non-null, so a second page entry would call `load` on a
  released pool; set it to `null` there if you wire up the teardown.
- Eight pads are positioned with percentage `position()` values inside a
  `RelativeContainer` without any relative constraints, so the layout is tuned
  to one aspect ratio rather than genuinely relative.

## Pitfalls

- **`HW-13-0070`** (B/medium, confirmed): `reqPermissionsFromUser` returns
  `false` when the permission is granted and `true` when it is refused, and
  returns unconditionally on the first array element so later permissions are
  never inspected. Harmless only because callers ignore the value. Fix:
  accumulate across the loop and compare `!== 0` for denied.
- **`HW-13-0071`** (B/low, confirmed): `aboutToAppear` "preloads" by calling
  `tapButton(0)`, `tapButton(1)` and `tapButton(2)`, so every cold start plays
  three drum sounds and fires three haptic patterns with no user interaction.
  Fix: call `loadSound` for each asset instead.
- **`HW-13-0015`** (B/medium, probable, systematic across five samples): file
  descriptors are closed in a `finally` while an asynchronous media consumer is
  still reading them. This sample has two instances -
  `HapticUtils.ets:25-46` (the haptic fd closed while `startVibration` reads it)
  and `SoundPoolUtil.ets:83-98` (the sound fd closed as soon as `load()`
  resolves, before `loadComplete`). Fix: close from the completion callback.
- Not filed, but adjacent: `checkPermissions` assigns `applyResult` per
  iteration instead of accumulating, and `loadSound`'s `-1` return conflates
  "already loaded" with "failed".

## References

- `huawei_industry_tree/13_media_entertainment/docs/31_drum.md` - the source page, including the haptic JSON format
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drum-0000002332280266
- `documentation/harmonyos-references/04_media/js-apis-inner-multimedia-soundpool.md` - `createSoundPool`, `load`, `play`, `PlayParameters`, the `loadComplete` event
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-multimedia-soundpool
- `documentation/harmonyos-guides/05_media/using-soundpool-for-playback.md` - the pool lifecycle and stream budget
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-soundpool-for-playback
- `documentation/harmonyos-references/03_system/js-apis-vibrator.md` - `startVibration`, `VibrateFromFile`, `hapticFd`, `usage`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-vibrator
- `documentation/harmonyos-guides/04_system/vibrator-guidelines.md` - authoring and playing custom haptic patterns
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/vibrator-guidelines
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.VIBRATE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `MEDIA-04` - the systematic fd-lifetime finding (`HW-13-0015`) this sample instantiates twice
