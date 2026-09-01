---
id: MEDIA-01
title: Music app architecture - one static AVPlayerManager driving AppStorage, AVSession and a continuous task
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
sample: huawei_industry_tree/13_media_entertainment/downloads/HarmonyMusic.zip
kits: ["@kit.MediaKit", "@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BackgroundTasksKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.FormKit", "@kit.PerformanceAnalysisKit"]
apis: ["media.createAVPlayer", "AVPlayer.on('stateChange')", "AVPlayer.on('durationUpdate')", "AVPlayer.on('timeUpdate')", "AVPlayer.fdSrc", "avSession.createAVSession", setAVMetadata, setAVPlaybackState, "backgroundTaskManager.startBackgroundRunning", "wantAgent.getWantAgent", "resourceManager.getRawFdSync", "preferences.getPreferencesSync", AppStorage, "@StorageLink", "UIContext.createAnimator", swipeAction, Tabs, "UIAbility.callee", formBindingData]
permissions: ["ohos.permission.KEEP_BACKGROUND_RUNNING", "ohos.permission.MICROPHONE"]
min_api: 20
modules: [entry]
findings: [HW-13-0001, HW-13-0007, HW-13-0008, HW-13-0009, HW-13-0010, HW-13-0011, HW-13-0012]
status: verified-with-fixes
---

## When to use

Load this card when you are laying out **a whole audio app**, not a single
playback screen: several browse tabs, a mini player that follows the user
across them, a full-screen player page, and playback that must survive the
user leaving the app. It is the industry's reference architecture page, and
the zip behind it is the only sample in `13_media_entertainment` that wires
AVPlayer, AVSession, a continuous task, `preferences` persistence and a form
extension together in one project.

The transferable idea is a **single static manager holding the player, and
one AppStorage key holding the state**. No page owns the player; every page
reads the same `@StorageLink(SONG_KEY)` object and calls static methods on
`AVPlayerManager`. That is what makes the mini player on the home tabs and
the full player page stay in sync for free, and it is the shape to copy for
podcast, audiobook and radio apps too.

**Read `HW-13-0007` before adopting the background half.** The sample's
continuous task is guarded by a condition that is always true by the time any
song plays, so background playback - the headline claim of the document
(支持后台、锁屏播放, "supports background and lock-screen playback") - never
actually starts. Everything else in the architecture is sound; that one
`return` is not.

## Feature checklist

- Five bottom tabs (推荐 recommend, 发现 find, 喜欢 like, 动态 moment,
  我的 mine), each a separate page component.
- A mini player bar docked above the tab bar, visible only when the play list
  is non-empty and the current tab is not 我的.
- Tapping the bar routes to a full-screen player with blurred cover art, a
  rotating disc animation, a seek slider and prev/play/next.
- The play list opens as a bottom panel; a row can be swiped to reveal a
  删除 (delete) button that removes the song, including the playing one.
- Playback state (cover, title, artist, position, duration, index, mode) is
  one object in `AppStorage`, so every screen shows the same thing.
- The current state is written to `preferences` on every track change and
  reloaded at launch, so the app reopens on the song it was playing.
- An AVSession publishes metadata and playback state to the system control
  centre and accepts play/pause/next/previous/seek from it.
- The ability exposes `prev` and `next` over `callee`, so a widget or another
  ability can drive playback.
- Playback continues when the app goes to the background (only after the fix
  in `HW-13-0007`).

## Architecture

One `entry` HAP, `deviceTypes: ["phone"]` only, 21 ArkTS files, ~2900 lines.
The document says the production shape would split the feature modules into
HARs; the shipped zip is the flattened single-HAP version of that design.

```
entry/src/main/ets
├── components/PlayerNav.ets        the mini player bar; reads AppStorage, calls AVPlayerManager
├── constants/
│   ├── EventConstants.ets          EmitEventType / PublishEventType enums (widget events)
│   ├── Index.ets                   barrel + SONG_KEY, the single AppStorage key
│   └── MusicConstants.ets          336 lines of static playlists, rankings, comments
├── entryability/EntryAbility.ets   window setup, manager bootstrap, callee handlers
├── entryformability/EntryFormAbility.ets  widget provider (see Constraints)
├── models/
│   ├── Index.ets                   barrel + TabClass
│   ├── Music.ets                   SongItemType and the per-tab list shapes
│   └── PlayState.ets               PlayStateType - the one object in AppStorage
├── pages/
│   ├── Index.ets                   @Entry, the five Tabs + the docked PlayerNav
│   ├── recommend/Recommend.ets     daily picks
│   ├── find/Find.ets               rankings (HW-13-0010 lives here)
│   ├── like/Like.ets               favourites list
│   ├── moment/Moment.ets           comments feed
│   ├── mine/Mine.ets               profile, receives $playState by reference
│   └── play/Play.ets               @Entry, the full player + swipe-delete list
└── utils/
    ├── AVPlayerManager.ets         the player, the play list, the continuous task
    ├── AVSessionManager.ets        the system session and its remote controls
    ├── ImageSave.ets               downloads cover art into tempDir via request
    └── SongManager.ets             preferences-backed persistence of PlayStateType
```

The documented tree matches the zip file for file - unusual in this industry,
where `HW-13-0003` records three other media docs whose trees disagree with
their zips in case or extension. Only the trailing comments differ.

**The design decision worth copying** is that `AVPlayerManager` is a class of
static members, not an instance, and its only channel to the UI is
`AppStorage.setOrCreate<PlayStateType>(SONG_KEY, ...)`. Pages never hold a
player reference and never pass playback state to each other. `PlayerNav` and
`Play` both declare `@StorageLink(SONG_KEY) playState: PlayStateType`, so the
mini bar and the full player are two views of one object; routing between
them costs nothing and cannot desync. The ability injects the two things a
static class cannot obtain for itself - the `UIContext` and the ability
`Context` - once, in `onWindowStageCreate`.

The cost of that decision is that the manager is a singleton with no
lifecycle: it is created before any page exists and released only when
`pages/Index` disappears. If you copy this shape, keep the release path
(`stop` then `release` in `aboutToDisappear`) that this sample does get right -
`HW-13-0005` records five media samples that create players and never release
them at all.

## Implementation steps

1. **Define one state class** (`PlayStateType`) covering cover, title, artist,
   source, index, position, duration, `isPlay`, play mode and the play list,
   and one constant key (`SONG_KEY`) for it.
2. **Make the manager static.** `player`, `currentSong` and `uiContext` are
   static fields; every page calls `AVPlayerManager.xxx()` directly.
3. **Bootstrap from the ability**, inside `loadContent().then()`: hand the
   manager the `UIContext`, hand `SongManager` the ability `Context`, then
   `await SongManager.getSong()` and push the restored state into AppStorage.
4. **Register the three AVPlayer callbacks** (`stateChange`, `durationUpdate`,
   `timeUpdate`) once in `init()`, and drive `prepare()` / `play()` from the
   state machine rather than from the click handlers.
5. **Also register `on('audioInterrupt')`** and map its hints onto `isPlay`
   and the AVSession state; without it a phone call desyncs everything
   (`HW-13-0009`).
6. **Create the AVSession at launch and start the continuous task when
   playback starts.** Both are required for background audio - do not treat
   the session as a substitute for the task (`HW-13-0007`).
7. **Set `backgroundModes: ["audioPlayback"]` on the ability** and declare
   `ohos.permission.KEEP_BACKGROUND_RUNNING`; declare nothing else you do not
   use - the sample also asks for `MICROPHONE` and never records
   (`HW-13-0001`).
8. **Close every raw file descriptor** you open for the player, after the
   player is done with it (`HW-13-0012`).
9. **Mutate the play list with `splice`, never `slice`** (`HW-13-0008`), and
   recompute `playIndex` modulo the *new* length.
10. **Format times with a `>= 60` boundary** and keep any random-mode helper
    genuinely random (`HW-13-0011`).
11. **Pass the right column to the right argument** in list builders; the
    rankings pass the title where the artist belongs (`HW-13-0010`).

## Verified snippets

All snippets are from `HarmonyMusic.zip`. Corrected forms are marked.

**Bootstrapping the singletons - `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.loadContent('pages/Index')
    .then(async () => {
      let win = windowStage.getMainWindowSync();
      let uiContext = win.getUIContext();
      win.setWindowLayoutFullScreen(true);
      AppStorage.setOrCreate(
        'topHeight',
        uiContext.px2vp(win.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height)
      );
      AppStorage.setOrCreate(
        'bottomHeight',
        uiContext.px2vp(win.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height)
      );
      AVPlayerManager.uiContext = uiContext;
      AVPlayerManager.init();
      AVSessionManager.init(this.context);
      SongManager.context = this.context;
      AVPlayerManager.currentSong = await SongManager.getSong(); // 获取缓存的音乐信息
      AppStorage.setOrCreate(SONG_KEY, AVPlayerManager.currentSong); // 更新到全局状态
    });
}
```

**Three injections carry the design.** `AVPlayerManager.uiContext` comes from
the *window*, `win.getUIContext()` - this is the correct source, and worth
noting because `HW-13-0032` records four other media samples that write
`new UIContext()` instead and end up with a detached instance whose
`getHostContext()` has no host. `SongManager.context` is the ability context,
which `preferences` needs. And the restored `PlayStateType` is pushed into
AppStorage *after* the await, so the first frame of `pages/Index` already sees
the previous session's song.

`callee.on('prev' | 'next')` in `onCreate` is the other half of the same idea:
the widget does not talk to the player, it talks to the ability, which calls
the same static methods the UI calls.

**The state pipe - `entry/src/main/ets/utils/AVPlayerManager.ets`** (as shipped)

```typescript
class AVPlayerManager {
  static player: media.AVPlayer | null = null;
  static currentSong: PlayStateType = new PlayStateType();
  static uiContext: UIContext | undefined = undefined;

  static async init() {
    if (!AVPlayerManager.player) {
      AVPlayerManager.player = await media.createAVPlayer();
    }
    AVPlayerManager.player.on('stateChange', (state) => {
      switch (state) {
        case 'initialized':
          AVPlayerManager.player!.prepare();
          break;
        case 'prepared':
          AVPlayerManager.player!.play();
          AVPlayerManager.currentSong.isPlay = true;
          break;
        case 'completed':
          AVPlayerManager.nextPlay(AVPlayerManager.currentSong.playMode === 'repeat');
          break;
      }
    });
    AVPlayerManager.player.on('durationUpdate', (duration) => {
      AVPlayerManager.currentSong.duration = duration;
      AVSessionManager.setAVMetaData(AVPlayerManager.currentSong.playList[AVPlayerManager.currentSong.playIndex]);
    });
    AVPlayerManager.player.on('timeUpdate', async (time) => {
      AVPlayerManager.currentSong.time = time;
      AVSessionManager.setAVPlaybackState();                             // system control centre
      AppStorage.setOrCreate<PlayStateType>(SONG_KEY, AVPlayerManager.currentSong); // every page
    });
  }
```

**`stateChange` is a state machine, not a set of callbacks.** Setting `fdSrc`
moves the player to `initialized`; the handler calls `prepare()`; that moves
it to `prepared`; the handler calls `play()`. So `changePlay()` only has to
`reset()` and assign a new source - the rest happens by itself, and the same
path serves a first play, a track switch and an auto-advance.

The `timeUpdate` handler is the single writer of UI state. One event feeds
both consumers: `setAVPlaybackState()` for the lock screen and
`AppStorage.setOrCreate` for the slider, the mini bar and the disc animation.
Adding a fourth consumer costs one line here and no changes anywhere else.

What is missing from this list is `on('audioInterrupt')` and `on('error')`.
Focus loss pauses the stream without any of these three firing, so `isPlay`
stays true, the pause icon keeps showing "playing", and the next tap calls
`pause()` on an already-paused player (`HW-13-0009`).

**The continuous task - same file** (corrected, see `HW-13-0007`)

```typescript
static async startBackgroundTask() {
  // FIX: shipped code begins with
  //   if (AVSessionManager.session.sessionId) { return; }
  // The session is created in onWindowStageCreate, so this is always true and
  // the continuous task is never requested.
  try {
    let wantAgentInfo: wantAgent.WantAgentInfo = {
      wants: [
        {
          bundleName: 'com.example.xiaohuamusic',
          abilityName: 'com.example.xiaohuamusic.EntryAbility'
        }
      ],
      actionType: wantAgent.OperationType.START_ABILITY,
      requestCode: 0,
      wantAgentFlags: [wantAgent.WantAgentFlags.UPDATE_PRESENT_FLAG]
    };
    let want = await wantAgent.getWantAgent(wantAgentInfo);
    await backgroundTaskManager.startBackgroundRunning(
      AVPlayerManager.uiContext?.getHostContext(),
      backgroundTaskManager.BackgroundMode.AUDIO_PLAYBACK,
      want
    );
  } catch (e) {
    AVPlayerManager.uiContext?.showAlertDialog({ message: e.message });
  }
}
```

**The guard inverts the platform requirement.** The continuous task guide is
explicit: an application that needs to play media in the background "must
access the AVSession service **and** request a continuous task of the
AUDIO_PLAYBACK type". The session is not an alternative to the task; it is a
precondition for it. The sample creates the session at launch, then refuses
the task because the session exists, so pressing Home suspends the process and
the audio stops - while `module.json5` still declares
`backgroundModes: ["audioPlayback"]` and `KEEP_BACKGROUND_RUNNING`, which is
what makes the defect look like it works in review.

The `WantAgent` is what the notification in the panel opens, which is why it
names this app's own `EntryAbility`. Note that nothing calls
`stopBackgroundRunning`: a full implementation cancels the task when playback
stops, otherwise the system will suspend the app anyway once it detects a
continuous task of `AUDIO_PLAYBACK` type with no audio playing.

**Loading a track and its descriptor - same file** (corrected, see `HW-13-0012`)

```typescript
static async changePlay() {
  await AVPlayerManager.player!.reset();
  AVPlayerManager.currentSong.duration = 0;
  AVPlayerManager.currentSong.time = 0;

  let ctx = AVPlayerManager.uiContext?.getHostContext();
  if (!ctx) {
    return;
  }
  // 此处为特例代码来访问本地音乐文件，具体播放资源可自定义
  let fileDescriptor = ctx.resourceManager.getRawFdSync(`Delacey - Dream It Possible.flac`);
  let avFileDescriptor: media.AVFileDescriptor = {
    fd: fileDescriptor.fd,
    offset: fileDescriptor.offset,
    length: fileDescriptor.length
  };
  AVPlayerManager.player!.fdSrc = avFileDescriptor;
  // FIX: absent in the sample - release the previous track's raw descriptor.
  // Keep the rawfile name alongside the fd and call, after the reset above:
  //   ctx.resourceManager.closeRawFd(previousRawFileName);

  AVPlayerManager.currentSong.img = AVPlayerManager.currentSong.playList[AVPlayerManager.currentSong.playIndex].img;
  AVPlayerManager.currentSong.name = AVPlayerManager.currentSong.playList[AVPlayerManager.currentSong.playIndex].name;
  SongManager.updateSong(JSON.stringify(AVPlayerManager.currentSong));
}
```

**`reset()` before assigning a new source, and a raw fd per track.** The
`await` on `reset()` matters: assigning `fdSrc` while the player is still
`playing` throws. Assigning it afterwards re-enters the `initialized` branch
of the `stateChange` handler, which is how a track switch reaches `play()`
without any imperative sequence here.

`getRawFdSync` hands out a descriptor into the HAP's own resources, and every
track change takes a new one. `closeRawFd` is never called anywhere in the
project, so the descriptors accumulate for the process lifetime
(`HW-13-0012`, systematic across seven media samples). The correct order is
release-then-close: the player must be reset or released before the
descriptor backing it is closed.

The final line is the persistence story - `SongManager.updateSong` serialises
`PlayStateType` into `preferences` on every track change, and `getSong()` at
launch parses it back with `new PlayStateType()` as the default, so a first
run needs no special case.

**Removing the playing song - same file** (corrected, see `HW-13-0008`)

```typescript
static async remove(index: number) {
  if (AVPlayerManager.currentSong.playIndex === index) {
    if (AVPlayerManager.currentSong.playList.length > 1) {
      AVPlayerManager.currentSong.playList.splice(index, 1);   // FIX: shipped code calls slice(index, 1)
      AVPlayerManager.currentSong.playIndex =
        (AVPlayerManager.currentSong.playIndex + AVPlayerManager.currentSong.playList.length) %
        AVPlayerManager.currentSong.playList.length;
      AVPlayerManager.singlePlay(AVPlayerManager.currentSong.playList[AVPlayerManager.currentSong.playIndex]);
    } else {
      await AVPlayerManager.player?.reset();
      AVPlayerManager.currentSong = new PlayStateType();
      AVPlayerManager.uiContext?.getRouter().back();
    }
  } else {
    if (AVPlayerManager.currentSong.playIndex > index) {
      AVPlayerManager.currentSong.playIndex--;
    }
    AVPlayerManager.currentSong.playList.splice(index, 1);
  }
  AppStorage.setOrCreate<PlayStateType>(SONG_KEY, AVPlayerManager.currentSong);
}
```

**Two branches, two different index repairs - and one of them was written
with the wrong method.** `slice(index, 1)` returns a copy and mutates nothing
(and its second argument is an *end*, not a count), so in the shipped code the
playing song stays in the list; the modulo then reproduces the same index and
`singlePlay` restarts the same song. The sibling branch four lines below uses
`splice` correctly, which is what makes this a typo rather than a design
error - and what makes it easy to miss in review.

The non-current branch decrements `playIndex` *before* splicing when the
removed row sits above the playing one - after any list mutation the index
into it must be repaired in the same operation, because `PlayStateType`
carries the index, not a reference to the song.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.KEEP_BACKGROUND_RUNNING" },
  {
    "name": 'ohos.permission.MICROPHONE',
    "reason": "$string:MICROPHONE_REASON",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
],
"abilities": [
  {
    "name": "EntryAbility",
    "backgroundModes": ["audioPlayback"]
  }
]
```

- `KEEP_BACKGROUND_RUNNING` is a normal permission granted at install, and is
  exactly what `startBackgroundRunning` needs; `backgroundModes: ["audioPlayback"]`
  on the ability is its counterpart. Both are correct - they are simply never
  exercised until `HW-13-0007` is fixed.
- `MICROPHONE` is `user_grant` and nothing in the project records: a
  full-source search finds no `AVRecorder` and no `AudioCapturer`. An audio
  player asking for the microphone is a privacy red flag a store review will
  raise (`HW-13-0001`). Delete it - and note `when: "always"` compounds it,
  since even a sample that did record from the foreground would say `"inuse"`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  matching the document - unlike the three docs in `HW-13-0004`.
- `deviceTypes` is `["phone"]` only. The player page uses `px` literals
  (`'1100px'`) for the cover art and header width, so the layout is designed
  for one screen width and will not adapt to a tablet or 2in1 window.
- **Every song plays the same file.** `changePlay` hardcodes
  `getRawFdSync('Delacey - Dream It Possible.flac')`; `SongItemType.src` is
  overwritten with a stringified descriptor and never used as a source. This
  is a framework sample, not a working library - the list, the metadata and
  the transport are real, the media resolution is not.
- `EntryFormAbility` exists and implements the full `FormExtensionAbility`
  surface, but `module.json5` declares no `extensionAbilities` entry for it,
  so the widget cannot be added from the home screen as shipped. The
  `callee` handlers it would use (`prev`, `next`, `getFormId`) are wired.
- Random play mode is unreachable: no UI writes `playMode`, and the helper
  behind it is broken anyway (`HW-13-0011`).
- The 喜欢 / 动态 / 我的 tabs render static data from `MusicConstants.ets`;
  there is no network layer and no database, although the document's
  播放历史、收藏技术方案 section describes an encrypted relational store.

## Pitfalls

- **`HW-13-0007` (B/high, probable) - background playback is unreachable.**
  `startBackgroundTask` returns early whenever `AVSessionManager.session.sessionId`
  is set, and the session is created at launch, so the continuous task is never
  requested and audio stops when the app leaves the foreground. Fix: delete the
  early return - AVSession and the continuous task are both required, not
  alternatives.
- **`HW-13-0008` (B/medium, confirmed) - deleting the playing song uses
  `Array.slice(index, 1)`.** `slice` is non-mutating and takes an end index,
  so the song stays in the list and simply replays. Fix: `splice(index, 1)`,
  as the sibling branch already does.
- **`HW-13-0009` (B/medium, probable) - no `on('audioInterrupt')` handler.**
  A phone call pauses the stream while `isPlay` stays true, so the UI and the
  AVSession both claim to be playing and the next tap pauses an already-paused
  player. Fix: register the handler and map its hints onto the state.
- **`HW-13-0010` (B/low, confirmed) - the rankings pass the title as the
  artist.** `this.Song('1', item.song1, item.song1, true)` against a
  `Song(id, song, author, firstRow)` signature; `PaihangListType.author1..3`
  are defined and never read. Fix: pass `item.authorN`.
- **`HW-13-0011` (B/low, confirmed) - `number2time` uses `second > 60`,** so
  exactly one minute renders as `00:60`; and `doRand()` re-seeds a DRBG with
  the constant `[1,2,3]` on every call, so the `do { } while (index === playIndex)`
  loops around it never terminate if random mode is ever wired up. Fix:
  `>= 60`, and drop the fixed seed.
- **`HW-13-0012` (B/low, confirmed) - `getRawFdSync` descriptors are never
  closed.** One leaks per track change here, and the same omission is present
  in six other media samples. Fix: pair every `getRawFd`/`getRawFdSync` with
  `closeRawFd` after the player has released the source.
- **`HW-13-0001` (D/low, confirmed) - `MICROPHONE` is declared and never
  used.** Fix: remove it; keep `KEEP_BACKGROUND_RUNNING`, which the
  continuous task genuinely needs.
- Related systematics that this sample avoids and its siblings do not:
  `HW-13-0005` (players never released - this one releases in
  `aboutToDisappear`), `HW-13-0032` (`new UIContext()` - this one takes the
  context from the window), `HW-13-0003` (documented trees that disagree with
  the zip - this one matches).

## References

- `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - AUDIO_PLAYBACK, `backgroundModes`, and the rule that background media needs AVSession *and* a continuous task
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-guides/05_media/avsession-overview.md`, `.../local-avsession.md` - creating and activating a local session
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avsession-overview
- `documentation/harmonyos-guides/05_media/avsession-background-scene.md` - the background/lock-screen scenario
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avsession-background-scene
- `documentation/harmonyos-guides/05_media/audio-playback-overview.md`, `.../audio-playback-stream-management.md` - AVPlayer state machine and focus/interrupt handling
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-playback-overview
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `fdSrc`, `reset`, `release`, the event list
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-avsession.md` - `setAVMetadata`, `setAVPlaybackState`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-avsession
- `MEDIA-02` - the index of the 40 key-scenario samples that fill in this architecture
- `MEDIA-13` - importing local music files, the piece this sample stubs out
