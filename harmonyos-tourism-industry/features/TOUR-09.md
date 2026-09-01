---
id: TOUR-09
title: Attraction audio guide - AVPlayer over rawfile commentary tracks
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
sample: huawei_industry_tree/09_tourism/downloads/景点语音讲解示例代码.zip
kits: ["@kit.MediaKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkTS"]
apis: ["media.createAVPlayer", "media.AVPlayer", "media.AVFileDescriptor", "avPlayer.fdSrc", "on('stateChange')", "on('timeUpdate')", "on('durationUpdate')", "on('seekDone')", "on('error')", prepare, play, pause, stop, reset, release, "resourceManager.getRawFdSync", "@StorageLink", "@StorageProp", "@Watch", NavPathStack, pushPath, setInterval, clearInterval]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0052, HW-09-0053, HW-09-0054, HW-09-0055, HW-09-0056, HW-09-0057, HW-09-0058, HW-09-0059, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for **short spoken audio bundled with the app** - an attraction
commentary, a museum guide, a pronunciation clip, a meditation track - where
the files ship in `rawfile` rather than streaming, and one shared player
serves a list screen and a detail screen at the same time.

The two pieces worth taking are the **`AVPlayer` state machine driven entirely
from `on('stateChange')`** - which is how HarmonyOS playback is meant to be
written, since `prepare()` and `play()` are only legal from specific states -
and **`getRawFdSync` into `fdSrc`**, the way to play a packaged asset without
copying it into the sandbox.

**Read the Pitfalls before copying `MediaPlayer.ets`.** The state machine is
sound; the object around it is not. Eight findings, and the two that matter
most are in the setup path.

## Feature checklist

- A main page with feature entries, recommended attractions and exhibitions.
- An album page listing the commentary tracks of one attraction, each row with
  an index, title, duration and play count.
- Tapping a row's play icon starts, pauses or switches tracks against a single
  shared player.
- The currently playing row is tinted blue.
- Tapping the row body opens a detail page with the cover image and a large
  transport control.
- The detail page shows elapsed time updating as the audio plays.
- Leaving the detail page stops playback and resets the player.
- Destroying the ability releases the player.

## Architecture

One `entry` module. The player is a singleton utility; everything else is
presentation over static model data.

```
entry/src/main/ets
├── component
│   ├── ExhibitionRecommendations.ets
│   ├── FeatureItemComponent.ets
│   ├── RecentExhibition.ets
│   ├── RecommendedAttractions.ets
│   ├── ScenicSpotSeriesCard.ets
│   └── VoicePlayerComponent.ets      one commentary row: the transport button
├── entryability/EntryAbility.ets     creates and releases the player
├── entrybackupability/
├── model
│   ├── AttractionsAlbumParam.ets     SingleAudioCommentary { id, title, cover, duration, audioResource }
│   ├── FeatureItems.ets
│   ├── RecommendedAttractions.ets
│   └── SelfGuidedData.ets
├── pages
│   ├── MainPage.ets
│   ├── AttractionsAlbumPage.ets      the track list
│   └── ScenicSpotDetailsPage.ets     the single-track player screen
└── utils
    ├── Logger.ets
    └── MediaPlayer.ets               the singleton AVPlayer wrapper
entry/src/main/resources/rawfile/
└── example1.mp3 … example9.mp3
```

The documented tree matches the zip.

**One player, many screens.** `MediaPlayer` is a singleton whose *state* lives
in `public static` fields - `state`, `isPlaying`, `time`, `duration`,
`curAudioId`, `audioUrl`, `playList`, `playIndex`, `playMode`. Every screen
reads those statics directly. That is what makes a track started from the list
still playing when the detail page opens, and it is also the source of three
of the findings: statics are not observable (hence the polling timer,
`HW-09-0055`), and any component can write them (hence a list row resetting
app-wide playback, `HW-09-0059`).

**The state machine is the design.** Almost nothing happens in the click
handlers; they set `audioUrl` and `curAudioId` and then push the player toward
`idle`, and the `stateChange` handler carries it the rest of the way:

```
                    (audioUrl set)          prepare()          play() if not on detail page
  reset() ──> idle ──────────────> initialized ──────> prepared ──────────────> playing
     ^                                                                             │
     └──────────── completed / stopped / error <───────────────── pause() ──> paused
```

`isDetailsPage` is the one flag that changes the machine's behaviour: on the
list, reaching `prepared` auto-plays; on the detail page it does not, because
the user is expected to press the big button.

## Implementation steps

1. **Create the player once, in the ability**, and release it in `onDestroy`.
   Call `getAVPlayer` **or** `initialize`, not both (`HW-09-0053`).
2. **Register the callbacks on the player, once**: `error`, `stateChange`,
   `seekDone`, `durationUpdate`, `timeUpdate` - and `audioInterrupt`
   (`HW-09-0056`).
3. **Log the error and stop the retry cycle** in the `error` handler
   (`HW-09-0054`).
4. **Drive everything from `stateChange`**: assign `fdSrc` on `idle`,
   `prepare()` on `initialized`, `play()` on `prepared`, `reset()` on
   `completed`.
5. **Open a `rawfile` asset with `getRawFdSync`** and hand the three fields to
   `fdSrc`.
6. **Switch tracks by resetting**, not by assigning a new source over a
   playing one: set the new `audioUrl` and `curAudioId`, call `reset()`, and
   let the `idle` branch pick the new source up.
7. **Publish position and state observably** rather than polling
   (`HW-09-0055`).
8. **Let the screen, not the row, own teardown** (`HW-09-0059`).

## Verified snippets

All snippets are from `景点语音讲解示例代码.zip`. Corrected forms are marked.

**Playing a packaged asset — `entry/src/main/ets/utils/MediaPlayer.ets`** (as shipped)

```typescript
import { media } from '@kit.MediaKit';

public async setAudioResourceFromRowFile(url: string, context: common.UIAbilityContext) {
  if (url === undefined) {
    return;
  }
  if (!this.avPlayer) {
    this.avPlayer = await this.getAVPlayer(context);
  }
  const fileDescriptor = context.resourceManager.getRawFdSync(url);
  const avFileDescriptor: media.AVFileDescriptor = {
    fd: fileDescriptor.fd,
    offset: fileDescriptor.offset,      // rawfiles live inside the HAP: offset and length matter
    length: fileDescriptor.length
  };
  this.avPlayer.fdSrc = avFileDescriptor;   // assigning fdSrc moves the player to `initialized`
}
```

**The `offset` and `length` are not optional.** A `rawfile` is a slice of the
HAP, not a standalone file, so the descriptor names a region: pass only `fd`
and the player reads the whole package. This is the mechanism to reach for
whenever audio ships with the app - no sandbox copy, no `fs.open`, no
permission.

Assigning `fdSrc` is itself the state transition. Nothing else is called here.

**The state machine — same file** (corrected, see `HW-09-0054`)

```typescript
private setupCallbacks(context: common.UIAbilityContext, avPlayer: media.AVPlayer): void {
  avPlayer.on('error', (err: BusinessError) => {
    Logger.error(`AVPlayer error ${err.code}: ${err.message}`);   // FIX: the sample discards err
    MediaPlayer.audioUrl = '';                                    // FIX: stops the retry loop
    MediaPlayer.instance?.reset();
  });

  avPlayer.on('stateChange', (state) => {
    MediaPlayer.state = state;
    switch (state) {
      case 'idle':
        MediaPlayer.isPlaying = false;
        if (MediaPlayer.audioUrl !== '') {           // re-arm whatever track is now selected
          const fd = context.resourceManager.getRawFdSync(MediaPlayer.audioUrl);
          avPlayer.fdSrc = { fd: fd.fd, offset: fd.offset, length: fd.length };
        }
        break;
      case 'initialized':
        avPlayer.prepare();                          // only legal from `initialized`
        break;
      case 'prepared':
        if (!MediaPlayer.isDetailsPage) {            // list auto-plays, detail page waits
          avPlayer.play();
        }
        break;
      case 'playing':
        MediaPlayer.isPlaying = true;
        break;
      case 'paused':
        MediaPlayer.isPlaying = false;
        break;
      case 'completed':
        MediaPlayer.isPlaying = false;
        MediaPlayer.time = 0;
        avPlayer.reset();                            // back to idle, ready to re-arm
        break;
      case 'stopped':
      case 'error':
        MediaPlayer.isPlaying = false;
        avPlayer.reset();
        break;
      case 'released':
        MediaPlayer.time = 0;
        MediaPlayer.isPlaying = false;
        break;
    }
  });

  avPlayer.on('seekDone',       (t: number) => { MediaPlayer.time = t; });
  avPlayer.on('durationUpdate', (d: number) => { MediaPlayer.duration = d; });
  avPlayer.on('timeUpdate',     (t: number) => { MediaPlayer.time = t; });
  // FIX: absent in the sample - see HW-09-0056
  avPlayer.on('audioInterrupt', (info: audio.InterruptEvent) => { /* pause / resume */ });
}
```

**This is the AVPlayer idiom worth internalising.** `prepare()` is legal only
from `initialized` and `play()` only from `prepared`, so a caller cannot
sequence them itself without racing; instead the caller only ever *sets a
source* or *calls reset*, and the handler advances the machine. `idle` doubles
as the re-arm point, which is what makes track switching a one-liner.

**Switching tracks — `entry/src/main/ets/component/VoicePlayerComponent.ets`** (as shipped)

```typescript
.onClick(async () => {
  const instance: MediaPlayer = await MediaPlayer.getInstance();
  await instance.getAVPlayer(this.context);

  if (MediaPlayer.state === 'playing' && MediaPlayer.curAudioId === this.singleAudioCommentary.id) {
    await instance.pause();                                    // same track, playing -> pause
  } else if (MediaPlayer.state === 'playing') {
    MediaPlayer.audioUrl = this.singleAudioCommentary.audioResource;
    MediaPlayer.curAudioId = this.singleAudioCommentary.id;
    await instance.reset();                                    // other track -> reset, idle re-arms
  } else if (MediaPlayer.state === 'paused' && MediaPlayer.curAudioId === this.singleAudioCommentary.id) {
    await instance.play();                                     // same track, paused -> resume
  } else if (MediaPlayer.state === 'paused') {
    MediaPlayer.audioUrl = this.singleAudioCommentary.audioResource;
    MediaPlayer.curAudioId = this.singleAudioCommentary.id;
    await instance.reset();
  } else {
    MediaPlayer.audioUrl = this.singleAudioCommentary.audioResource;
    MediaPlayer.curAudioId = this.singleAudioCommentary.id;
    instance.setAudioResourceFromRowFile(this.singleAudioCommentary.audioResource, this.context);
  }
  AppStorage.setOrCreate('curAudioId', MediaPlayer.curAudioId);
})
```

**Set the target first, then `reset()`.** Every switch branch writes
`audioUrl` and `curAudioId` *before* resetting, so by the time the player
reaches `idle` the new source is already selected and the handler picks it up.
That is why there is no explicit load call on the switch paths.

The row's highlight is driven by `AppStorage`:
`@StorageLink('curAudioId') curAudioId` on every row, compared against the
row's own id.

**Creating and releasing the player — `entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-09-0053`)

```typescript
async onWindowStageCreate(windowStage: window.WindowStage): Promise<void> {
  const instance: MediaPlayer = await MediaPlayer.getInstance();
  await instance.getAVPlayer(this.context);      // FIX: the sample also passes this into initialize()
  // ...
}

async onDestroy(): Promise<void> {
  const instance: MediaPlayer = await MediaPlayer.getInstance();
  await instance.release();                      // FIX: the sample does not await
}
```

The AVPlayer guide is blunt about why the release matters: "When the AVPlayer
is in the **prepared**, **playing**, **paused**, or **completed** state, the
playback engine is working and a large amount of RAM is occupied. If your
application does not need to use the AVPlayer, call **reset()** or
**release()** to release the instance."

**Formatting a duration — `VoicePlayerComponent.ets`** (as shipped)

```typescript
private formatTime(ms: number): string {
  if (ms < 0) {
    return '00:00';
  }
  const totalSeconds = Math.floor(ms / 1000);
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;
  if (hours > 0) {
    return util.format('%s:%s:%s',
      String(hours).padStart(2, '0'), String(minutes).padStart(2, '0'), String(seconds).padStart(2, '0'));
  }
  // ... mm:ss below an hour
}
```

`AVPlayer` reports **milliseconds**, and the hour segment is dropped below an
hour - both are the conventions a media UI needs.

## Permissions & config

**None.** The sample declares no `requestPermissions`, which is correct: the
audio is bundled in `rawfile` and read through the resource manager.

Add `ohos.permission.INTERNET` only if the commentary is streamed - the
AVPlayer guide requires it for a network playback path.

Audio files: `entry/src/main/resources/rawfile/example1.mp3` …
`example9.mp3`. `SingleAudioCommentary.audioResource` holds the bare filename.

Routing is by `routerMap` (`$profile:route_map`).

Resource directories: `base`, `dark`, `rawfile`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **One player for the whole app.** Two tracks cannot play at once, and any
  screen can reset the one player - that is the design, not a limitation to
  work around by creating a second `AVPlayer`.
- **Playback stops when the app backgrounds.** `EntryAbility` toggles an
  `isForeGround` flag that the row watches and uses to clear playback state.
  For commentary that should continue with the screen off, the AVPlayer guide
  points at AVSession plus a continuous task - neither is used here.
- The formats are limited to what the system supports; the sample ships MP3.
- `MediaPlayer.playList`, `playIndex` and `playMode` exist with getters and
  setters but nothing sets a play list, so continuous album playback is
  scaffolding only.
- Track durations shown in the list come from the static model, not from the
  player's `durationUpdate`.

## Pitfalls

- **`HW-09-0052` — `initialize()`'s fallback branch assigns the created player
  to its own parameter,** so `this.avPlayer` stays null and every later call
  hits its null guard. Masked only because the ability always passes a player
  in.
- **`HW-09-0053` — the ability wires the callbacks twice** by passing
  `getAVPlayer()`, which already wired them, into `initialize()`, which wires
  them again. If `on` appends, `prepare()` and `reset()` are each called twice
  per transition. *(probable — the AVPlayer reference page in this corpus is
  empty on repeated `on`.)*
- **`HW-09-0054` — the `error` handler binds the `BusinessError` and discards
  it,** then `reset()`s, which re-arms the same failing source through the
  `idle` branch - an invisible infinite retry.
- **`HW-09-0055` — the detail page polls the statics every 500 ms** although
  `timeUpdate`, `durationUpdate` and `stateChange` already push those values.
  The document teaches the polling.
- **`HW-09-0056` — no `audioInterrupt` handler and no `audioRendererInfo`,**
  in a player whose whole use case is running while the user walks around.
- **`HW-09-0057` — `timer: number | undefined = -1` cleared under
  `if (this.timer)`,** which is false for a legal id of 0, and nothing stops
  an existing interval before a new one is created.
- **`HW-09-0058` — `onPageShow` and `onPageHide` on a non-`@Entry`
  component** never run. Dead code that reads as working intent.
- **`HW-09-0059` — each list row resets the shared player** in its own
  `aboutToAppear` and `aboutToDisappear`, so rebuilding the list stops audio
  app-wide, N times over.

## References

- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - the AVPlayer flow, the callbacks to register, `audioRendererInfo`, when to `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-references/04_media/arkts-apis-media-t.md` - `AVPlayerState`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-t
- `documentation/harmonyos-references/04_media/arkts-apis-media-i.md` - `AVFileDescriptor`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-i
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFd` and `getRawFdSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/ts-custom-component-lifecycle.md` - why `onPageShow` needs `@Entry`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-custom-component-lifecycle
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageLink` and `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `TOUR-01` - the park app this commentary would sit inside
