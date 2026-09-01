# Pitfalls

> Generated from `features/*.md`. Source industry: `13_media_entertainment`, 43 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (20)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-13-0058` - getAssets used with READ_IMAGEVIDEO neither declared nor requested — thumbnails/durations silently never load (two samples)

- Category D, severity high, confidence probable
- Features: MEDIA-23, MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/23_audio_extractor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_extractor-0000002298797484
- Why: The getAssets calls throw at runtime (only logged): AudioExtractor never shows its preview thumbnail; SplashPage never shows thumbnails or durations — which also triggers its parallel-array desync.
- Fix: Declare/request READ_IMAGEVIDEO or read size/duration via AVMetadataExtractor on the picked fd.

### `HW-13-0002` - getAssets is called with READ_IMAGEVIDEO declared but never requested at runtime (also in 20_merge_video)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-03, MEDIA-19, MEDIA-20
- Document: `huawei_industry_tree/13_media_entertainment/docs/03_get_video_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_video_info-0000002273254857
- Why: A declared user-grant permission is not granted without a runtime request, so the getAssets query fails and the info/merge pipeline that consumes the fetched asset breaks; the picker-URI file APIs would need no permission at all.
- Fix: Drop the getAssets lookup in both samples and operate on the picker-returned URI directly.

### `HW-13-0005` - Systematic: five media samples create AVPlayer/SoundPool instances and never call release() anywhere in the project

- Category B, severity medium, confidence confirmed
- Features: MEDIA-18, MEDIA-24, MEDIA-26, MEDIA-38, MEDIA-41, MEDIA-43
- Document: `huawei_industry_tree/13_media_entertainment/docs/18_video_playlist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_playlist-0000002286827110
- Why: AVPlayer/SoundPool hold native decoder and audio resources; the official player guides end every lifecycle with release(). Templates that never release teach a leak — especially egregious in the play-resume demo whose entire subject is player lifecycle management.
- Fix: Add avPlayer.release() (and soundPool.release()) in aboutToDisappear/onDestroy of the five samples.

### `HW-13-0015` - Systematic: fds closed while an async media consumer is still using them (five samples)

- Category B, severity medium, confidence probable
- Features: MEDIA-04, MEDIA-20, MEDIA-31, MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/04_video_compress_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292
- Why: Thumbnails/haptics/first-plays/decodes fail intermittently depending on timing — the async consumer holds an fd the app already closed.
- Fix: Move the close into the completion path.

### `HW-13-0032` - Systematic: UIContext constructed with `new UIContext()` — a detached instance whose getHostContext() has no host (four media samples)

- Category B, severity medium, confidence probable
- Features: MEDIA-11, MEDIA-19, MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/11_avplayer_audio.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avplayer_audio-0000002293630521
- Why: UIContext must come from getUIContext(); a bare-constructed instance is not attached to any window, so host-context-dependent flows (resource loading, media library, file paths, toasts) fail at the core of each sample.
- Fix: Pass this.getUIContext() (or the ability context) in.

### `HW-13-0033` - Backgrounding only flips the UI flag while the comment claims audio stops; completion is never propagated either (three samples)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-11, MEDIA-14
- Document: `huawei_industry_tree/13_media_entertainment/docs/11_avplayer_audio.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avplayer_audio-0000002293630521
- Why: UI and player state permanently desync on routine background/complete events; in AVPlayerAudio one toggle after backgrounding destroys the player.
- Fix: Pause in the background hook; propagate completed to the UI.

### `HW-13-0039` - Systematic: finally-block closeSync on possibly-undefined files turns dialog refusal/cancel into secondary crashes (five samples)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-13, MEDIA-14, MEDIA-20, MEDIA-27, MEDIA-30
- Document: `huawei_industry_tree/13_media_entertainment/docs/14_video_demo_videocreategif.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_demo_videocreategif-0000002312993113
- Why: User refusal and open failures — routine paths — are converted into unhandled TypeErrors from cleanup code that masks the original error and leaks the successfully opened fds.
- Fix: if (f) closeSync(f) per handle.

### `HW-13-0045` - Systematic: speedIndex initialized to 2 — the first long-press fast-forward 'restores' 2x instead of the actual 1x (two samples)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-17, MEDIA-32
- Document: `huawei_industry_tree/13_media_entertainment/docs/17_speed_play.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/speed_play-0000002321158281
- Why: Before the user ever opens the speed menu, releasing the long press permanently leaves playback at 2x.
- Fix: Initialize speedIndex = 1.

### `HW-13-0050` - Systematic: reused work files opened without TRUNC — stale bytes survive into later runs (three samples)

- Category B, severity medium, confidence probable
- Features: MEDIA-20, MEDIA-25, MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
- Why: Cancel a save, merge fewer videos → the concat list references leftover segments and the output contains clips the user removed; the saved image carries old-image garbage after EOI.
- Fix: Add OpenMode.TRUNC; clear the work dir per run.

### `HW-13-0051` - Systematic: thumbnail/uri/duration parallel arrays desync via async packToFile callbacks (two samples)

- Category B, severity medium, confidence probable
- Features: MEDIA-20, MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
- Why: One failed/out-of-order thumbnail makes taps play the wrong video, reorders the wrong entries, and shows durations against the wrong clip — the merged order differs from what the user arranged.
- Fix: Push a single {uri, thumb, time} object after awaiting packToFile.

### `HW-13-0053` - Module-level singletons dereference AppStorage 'uiContext' at import time — set only inside the loadContent callback

- Category B, severity medium, confidence probable
- Features: MEDIA-20, MEDIA-26
- Document: `huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
- Why: Whether the app crashes at launch depends on module-evaluation order vs the callback — a first-run initialization hazard.
- Fix: Fetch the context lazily (or set AppStorage before loadContent).

### `HW-13-0061` - Waveform dB math omits the square root — 20*log10(meanSquare) doubles every level (three files, two samples)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-27, MEDIA-35
- Document: `huawei_industry_tree/13_media_entertainment/docs/27_audio_waveform_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_waveform_animation-0000002349412093
- Why: A true -30 dB signal computes as -60 dB; bars are systematically far too short and quiet audio clamps to the floor early — the samples' core visual is wrong.
- Fix: Fix the formula.

### `HW-13-0072` - Systematic: AVSession 'seek' delivers an absolute ms position but is fed into a relative-seconds helper (two samples)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-32, MEDIA-33
- Document: `huawei_industry_tree/13_media_entertainment/docs/32_video_screenshot.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_screenshot-0000002368729597
- Why: Any seek from the system playback controller jumps to the end of the video (clamped).
- Fix: setSeek(absolute) path for the session command.

### `HW-13-0099` - Systematic: 6 sample projects declare permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: MEDIA-09, MEDIA-13, MEDIA-28, MEDIA-32, MEDIA-33, MEDIA-34
- Document: `huawei_industry_tree/13_media_entertainment/docs/28_buffered_progress_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/buffered_progress_bar-0000002351008293
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-13-0003` - Systematic: media project trees use wrong letter case or extensions for real files (Entryability, logger, index.d.ets)

- Category E, severity low, confidence confirmed
- Features: MEDIA-05, MEDIA-13, MEDIA-24, MEDIA-30, MEDIA-32
- Document: `huawei_industry_tree/13_media_entertainment/docs/13_music_player.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_player-0000002296726005
- Why: The sample projects enable caseSensitiveCheck, so these names do not resolve; the .d.ets vs .d.ts confusion also misstates the NAPI typing file format.
- Fix: Correct the three entries.

### `HW-13-0012` - Systematic: resourceManager.getRawFd/getRawFdSync descriptors are never closed with closeRawFd in seven media samples

- Category B, severity low, confidence confirmed
- Features: MEDIA-01, MEDIA-11, MEDIA-17, MEDIA-18, MEDIA-24, MEDIA-32, MEDIA-33, MEDIA-34, MEDIA-37, MEDIA-42, MEDIA-43
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Raw HAP-resource descriptors accumulate for the process lifetime — unbounded in feed-style samples (VideoPlayList) and per-track in players.
- Fix: Pair every getRawFd with closeRawFd (after player release).

### `HW-13-0046` - Systematic: 100 ms 'wait for prepared' polling intervals are never cleared on failure and stack on repeated taps (three samples)

- Category B, severity low, confidence confirmed
- Features: MEDIA-17, MEDIA-28, MEDIA-32, MEDIA-33
- Document: `huawei_industry_tree/13_media_entertainment/docs/17_speed_play.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/speed_play-0000002321158281
- Why: A failed prepare leaves 10 Hz timers running forever; taps multiply them, and each later calls play().
- Fix: Track the id in a field, clear in aboutToDisappear and on error.

### `HW-13-0062` - Systematic: final partial audio chunk leaves stale bytes in the render buffer (four samples)

- Category B, severity low, confidence confirmed
- Features: MEDIA-27, MEDIA-30, MEDIA-35, MEDIA-36, MEDIA-42, MEDIA-43
- Document: `huawei_industry_tree/13_media_entertainment/docs/27_audio_waveform_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_waveform_animation-0000002349412093
- Why: The last frame of every playback renders garbage/echo from the previous buffer contents.
- Fix: memset the remainder / use partial-write APIs.

### `HW-13-0097` - 9 sample projects depend on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: MEDIA-04, MEDIA-14, MEDIA-15, MEDIA-20, MEDIA-23, MEDIA-25, MEDIA-30, MEDIA-41, MEDIA-42
- Document: `huawei_industry_tree/13_media_entertainment/docs/15_bullet_comment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bullet_comment-0000002284943480
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-13-0098` - Systematic: 31 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: MEDIA-03, MEDIA-06, MEDIA-08, MEDIA-10, MEDIA-11, MEDIA-12, MEDIA-13, MEDIA-14, MEDIA-15, MEDIA-16, MEDIA-18, MEDIA-19, MEDIA-20, MEDIA-21, MEDIA-22, MEDIA-23, MEDIA-25, MEDIA-27, MEDIA-28, MEDIA-29, MEDIA-30, MEDIA-31, MEDIA-34, MEDIA-35, MEDIA-36, MEDIA-37, MEDIA-38, MEDIA-39, MEDIA-40, MEDIA-41, MEDIA-42
- Document: `huawei_industry_tree/13_media_entertainment/docs/27_audio_waveform_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_waveform_animation-0000002349412093
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (87)

### `HW-13-0007` - Background playback is unreachable: startBackgroundTask early-returns whenever an AVSession exists, and the session is created at startup

- Category B, severity high, confidence probable
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Play a song, press Home: with no long-term task the process suspends and audio stops — contradicting the doc's 支持后台、锁屏播放 claim; HarmonyOS background audio needs AVSession PLUS the continuous task, the guard inverts that requirement.
- Fix: Remove the early return (or invert it: start the task because a session exists).

### `HW-13-0036` - playNewSong closes the fd in finally BEFORE assigning avPlayer.url = 'fd://N' — the player is handed a dead descriptor (plus an invalid-state play())

- Category B, severity high, confidence confirmed
- Features: MEDIA-13
- Document: `huawei_industry_tree/13_media_entertainment/docs/13_music_player.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_player-0000002296726005
- Why: AVPlayer needs the fd valid for the whole playback; closing before url assignment breaks prepare/playback (at best a race with the async close).
- Fix: Close after release/next-track; drop the premature play().

### `HW-13-0063` - bufferingUpdate CACHED_DURATION accumulated with += and never reset — the buffered bar shows garbage (doc ships the same code)

- Category B, severity high, confidence confirmed
- Features: MEDIA-28
- Document: `huawei_industry_tree/13_media_entertainment/docs/28_buffered_progress_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/buffered_progress_bar-0000002351008293
- Why: After a few reports the buffer slider pegs at 100% and the pause-on-stall logic runs on garbage — the sample's core feature.
- Fix: Assign instead of accumulate; add to reset().

### `HW-13-0078` - cutFile off-by-one: every delete writes srcFileSize - lastPos + 1 bytes — the PCM file grows 1 byte per delete, breaking 16-bit alignment

- Category B, severity high, confidence confirmed
- Features: MEDIA-35
- Document: `huawei_industry_tree/13_media_entertainment/docs/35_pcm_audio_edit.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_audio_edit-0000002476817872
- Why: S16LE mono becomes odd-sized: duration math is off and any recording appended at offset=fileSize is byte-misaligned — all further recorded audio is noise; errors accumulate per delete.
- Fix: Drop the +1.

### `HW-13-0080` - StopEncode waits on a condition variable with no predicate — a fast encode's notify is lost and the transcode hangs forever

- Category D, severity high, confidence confirmed
- Features: MEDIA-36
- Document: `huawei_industry_tree/13_media_entertainment/docs/36_pcm_transcode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_transcode-0000002513212618
- Why: The napi worker hangs, the JS callback never fires, TranscodePage stays disabled on 'transcoding' forever.
- Fix: wait(lock, [&]{return done;}) with the flag set at EOS/error.

### `HW-13-0083` - Player::Release detaches the decoder threads then frees the demuxer/decoder/context they may still be using — use-after-free on stop/exit

- Category D, severity high, confidence confirmed
- Features: MEDIA-37
- Document: `huawei_industry_tree/13_media_entertainment/docs/37_opengl_offscreen_render.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/opengl_offscreen_render-0000002553789479
- Why: Stopping playback or leaving the page mid-play frees objects under live threads — undefined behavior/native crash.
- Fix: Signal, notify, join; then free.

### `HW-13-0090` - Subtitle recognition starts before (and races) the async ffmpeg audio extraction — early len==0 is treated as EOF

- Category D, severity high, confidence confirmed
- Features: MEDIA-41
- Document: `huawei_industry_tree/13_media_entertainment/docs/41_video_subtitle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_subtitle-0000002568904821
- Why: On every launch output.pcm may not exist yet (openSync throws in the listener) or the reader catches the writer mid-file and finishes early — subtitles truncated or absent.
- Fix: Gate on the ffmpeg callback; distinguish EOF from catch-up.

### `HW-13-0104` - Every early return in the native compositor skips the EGL teardown, so each failed composition permanently leaks a display, a Pbuffer surface and a GL context

- Category B, severity high, confidence confirmed
- Features: MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/40_audio-v1_2-ts_64.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_64-0000002444273313
- Why: eglInitialize, eglCreatePbufferSurface and eglCreateContext allocate driver-side resources that are released only by the matching destroy calls; returning to ArkTS does not reclaim them, and the native module stays loaded for the life of the process. The three late returns at 223, 229 and 237 are the realistic ones: they fire on large images, which is exactly when a user retries. Each retry adds another leaked context and Pbuffer, so repeated failures exhaust GPU memory and eventually make even the succeeding path fail.
- Fix: Wrap the body in a scope guard, or route every failure through a single cleanup label that destroys whichever of display, surface and context are non-null. While there, add the missing glDeleteProgram for the program created at line 172; the other GL objects are deleted explicitly at 212-216 but the program is not.

### `HW-13-0008` - Deleting the currently playing song uses Array.slice(index, 1) — non-mutating and wrong arguments — so the song is never removed

- Category B, severity medium, confidence confirmed
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Swipe-delete the playing row: the song stays in the list and simply replays.
- Fix: Replace slice with splice.

### `HW-13-0009` - Music player registers no on('audioInterrupt') (or 'error') handler — focus interruptions desync UI and AVSession state

- Category B, severity medium, confidence probable
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: A phone call pauses the stream but currentSong.isPlay stays true: pause icon and disc animation keep showing 'playing', AVSession stays PLAYBACK_STATE_PLAY, and the next tap calls pause() on an already-paused player.
- Fix: Register on('audioInterrupt') and map hint events to state.

### `HW-13-0014` - selectVideo shadows `originFile` with an inner `let`, making the finally-close dead — one fd leaked per video selection

- Category B, severity medium, confidence confirmed
- Features: MEDIA-04
- Document: `huawei_industry_tree/13_media_entertainment/docs/04_video_compress_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292
- Why: Every selection (the demo's first step) leaks an fd; the cleanup is provably dead code.
- Fix: Drop the inner `let`.

### `HW-13-0018` - LazyForEach data source clear() starts at i = length — bogus notifyDataDelete(length) on every filter click

- Category B, severity medium, confidence confirmed
- Features: MEDIA-06
- Document: `huawei_industry_tree/13_media_entertainment/docs/06_hiararchicle_filtering.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hiararchicle_filtering-0000002287162465
- Why: Every filter interaction sends LazyForEach an out-of-range delete notification that can corrupt its index bookkeeping.
- Fix: Start at this.dataArr.length - 1.

### `HW-13-0022` - Floating-window drag adds the cumulative gesture offset every frame (and in vp against px state) — the window overshoots the finger

- Category B, severity medium, confidence confirmed
- Features: MEDIA-07
- Document: `huawei_industry_tree/13_media_entertainment/docs/07_sub_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sub_window-0000002254941572
- Why: Movement is roughly the sum of all intermediate offsets — the window flies past the finger; on density-3 screens the units are also 3x off.
- Fix: Capture the start position in onActionStart and add the converted cumulative offset once.

### `HW-13-0023` - Double-tap creates two AVPlayers: switchPlayOrPause has no in-flight guard while avPlayerFdSrc is async

- Category B, severity medium, confidence confirmed
- Features: MEDIA-07
- Document: `huawei_industry_tree/13_media_entertainment/docs/07_sub_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sub_window-0000002254941572
- Why: Routine double tap on the floating window doubles the audio and leaks a player.
- Fix: Guard with a creating flag / promise cache.

### `HW-13-0024` - Selecting a second song never changes the audio, and the subwindow is never destroyed

- Category B, severity medium, confidence probable
- Features: MEDIA-07
- Document: `huawei_industry_tree/13_media_entertainment/docs/07_sub_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sub_window-0000002254941572
- Why: First song works; every later selection only updates the title text while play/pause keeps toggling the first track.
- Fix: Add a reset/re-fdSrc path when 'resource' changes.

### `HW-13-0027` - Pause-on-cellular flow assumes netLost precedes netAvailable — seamless WiFi→cellular handovers are missed

- Category B, severity medium, confidence probable
- Features: MEDIA-09
- Document: `huawei_industry_tree/13_media_entertainment/docs/09_data_network_pause_playback.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_network_pause_playback-0000002291777477
- Why: The exact advertised scenario (video on WiFi, switch to data) is missed in the common handover order.
- Fix: Check the new handle's bearer directly in netAvailable.

### `HW-13-0030` - Doc advertises 歌词滚动 (lyric scrolling) but the sample never scrolls — scroller and 滚动索引 state are dead

- Category D, severity medium, confidence probable
- Features: MEDIA-10
- Document: `huawei_industry_tree/13_media_entertainment/docs/10_lyric_dynamic_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/lyric_dynamic_effect-0000002291790569
- Why: The advertised scrolling half of the effect is absent.
- Fix: scrollToIndex on line change.

### `HW-13-0034` - ListScroller never bound to any List, yet @Watch calls scrollTo on every video switch (throws 100004)

- Category B, severity medium, confidence confirmed
- Features: MEDIA-12
- Document: `huawei_industry_tree/13_media_entertainment/docs/12_video_switching_association.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_switching_association-0000002296452929
- Why: Every prev/next/row tap throws 'controller not bound to component' in the watch callback; the scroll-follow behavior can never work.
- Fix: Render the rows in a List bound to the scroller.

### `HW-13-0037` - Import loop bounded by the deduplicated array but indexing the raw one — re-importing an existing song skips new unique files

- Category B, severity medium, confidence confirmed
- Features: MEDIA-13
- Document: `huawei_industry_tree/13_media_entertainment/docs/13_music_player.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_player-0000002296726005
- Why: Whenever a selection overlaps existing songs, duplicates at the head are processed again while unique uris at higher indexes are never visited — new songs silently missing.
- Fix: Use filePath[i].

### `HW-13-0040` - A single failed thumbnail decode permanently traps the user: canNext gates BOTH back and next and only counts successes

- Category B, severity medium, confidence probable
- Features: MEDIA-14
- Document: `huawei_industry_tree/13_media_entertainment/docs/14_video_demo_videocreategif.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_demo_videocreategif-0000002312993113
- Why: One decode failure leaves the '视频处理中，请等候' state forever — back and next both refuse.
- Fix: catch → count++ (and never gate back).

### `HW-13-0047` - onReachEnd has no re-entry guard — one over-scroll schedules multiple page loads

- Category B, severity medium, confidence probable
- Features: MEDIA-18
- Document: `huawei_industry_tree/13_media_entertainment/docs/18_video_playlist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_playlist-0000002286827110
- Why: Duplicate pages appended to the feed (do-it-twice boundary); the flag exists but only drives the spinner.
- Fix: if (this.isLoading) return;

### `HW-13-0048` - isChosen set at the START of the last loop iteration — the Swiper renders before the sixth moving photo exists

- Category B, severity medium, confidence confirmed
- Features: MEDIA-19
- Document: `huawei_industry_tree/13_media_entertainment/docs/19_dynamic_photo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_photo-0000002321583145
- Why: MovingPhotoView gets an undefined movingPhoto on first render and srcList order depends on callback timing.
- Fix: Set isChosen after the loop (and await preparation).

### `HW-13-0049` - FFmpeg transpose=3 used as a 180° rotation — 180-oriented clips come out transposed and mirrored

- Category B, severity medium, confidence confirmed
- Features: MEDIA-20
- Document: `huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
- Why: Orientation-180 videos are geometrically wrong and can break the subsequent merge.
- Fix: Use -vf hflip,vflip for the 180 case.

### `HW-13-0055` - Progress bar uses a hardcoded 430 width for playback but the real screen width for seeking — bar and seek position disagree

- Category B, severity medium, confidence confirmed
- Features: MEDIA-22
- Document: `huawei_industry_tree/13_media_entertainment/docs/22_video_like.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_like-0000002332695009
- Why: On any non-430vp device the bar overflows/underfills and a drag released mid-screen seeks to a different timestamp than shown.
- Fix: Use screenW consistently.

### `HW-13-0057` - ffmpeg result code ignored: extraction failure still toasts 'Extract successfully' and opens the save picker

- Category B, severity medium, confidence confirmed
- Features: MEDIA-23
- Document: `huawei_industry_tree/13_media_entertainment/docs/23_audio_extractor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_extractor-0000002298797484
- Why: Every ffmpeg failure reports success and offers a nonexistent/empty .aac for saving.
- Fix: Branch on code === 0.

### `HW-13-0064` - A new 100 ms interval is spawned on EVERY slider onChange event (including each Moving tick), all sharing one stall counter

- Category B, severity medium, confidence confirmed
- Features: MEDIA-28
- Document: `huawei_industry_tree/13_media_entertainment/docs/28_buffered_progress_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/buffered_progress_bar-0000002351008293
- Why: Multiplied side effects and premature buffer-timeout stop on routine seeking.
- Fix: Gate on SliderChangeMode.End; track/clear the id.

### `HW-13-0065` - createAVPlayer in aboutToAppear races XComponent.onLoad — 'initialized' can bind an empty surfaceId (never re-applied)

- Category B, severity medium, confidence probable
- Features: MEDIA-28
- Document: `huawei_industry_tree/13_media_entertainment/docs/28_buffered_progress_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/buffered_progress_bar-0000002351008293
- Why: When the state callback wins, audio plays with no picture; nothing re-binds after onLoad.
- Fix: Set url from XComponent.onLoad (cf. MEDIA-24).

### `HW-13-0066` - Lock/unlock gesture threshold is an absolute 750vp of localY — unreachable on shorter screens

- Category B, severity medium, confidence probable
- Features: MEDIA-29
- Document: `huawei_industry_tree/13_media_entertainment/docs/29_lock_speed.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/lock_speed-0000002317513156
- Why: On devices under 750vp usable height the sample's core feature cannot trigger; on tall screens it triggers far above the hint area.
- Fix: Compute from componentHeight * ratio.

### `HW-13-0067` - HTTP response parser only accepts the audio payload on line index 0 and can resolve undefined

- Category B, severity medium, confidence confirmed
- Features: MEDIA-30
- Document: `huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
- Why: SSE-style responses carry multiple data: lines; whenever the payload is not line 0 the download silently produces nothing.
- Fix: Drop the i === 0 condition.

### `HW-13-0069` - GetRendererState leaks a native OH_AudioStreamBuilder per call — polled every 50 ms during playback

- Category B, severity medium, confidence confirmed
- Features: MEDIA-30
- Document: `huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
- Why: Up to 20 native objects leaked per second for the whole playback duration.
- Fix: Remove the builder creation.

### `HW-13-0070` - Permission helper inverts the grant check AND returns on the first element

- Category B, severity medium, confidence confirmed
- Features: MEDIA-31
- Document: `huawei_industry_tree/13_media_entertainment/docs/31_drum.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drum-0000002332280266
- Why: The helper reports the opposite of the user's decision and never checks later permissions.
- Fix: Accumulate and return after the loop with the correct comparison.

### `HW-13-0073` - AVSession cover PixelMap is released immediately after being stored for later use

- Category B, severity medium, confidence confirmed
- Features: MEDIA-32
- Document: `huawei_industry_tree/13_media_entertainment/docs/32_video_screenshot.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_screenshot-0000002368729597
- Why: The playback center gets a released native object — no cover image (or an error).
- Fix: Drop the premature release.

### `HW-13-0074` - startBackgroundRunning used without declaring KEEP_BACKGROUND_RUNNING (while an unused INTERNET is declared)

- Category E, severity medium, confidence confirmed
- Features: MEDIA-32
- Document: `huawei_industry_tree/13_media_entertainment/docs/32_video_screenshot.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_screenshot-0000002368729597
- Why: The call fails with 201 every time — advertised background playback never starts.
- Fix: Add KEEP_BACKGROUND_RUNNING (drop INTERNET).

### `HW-13-0075` - No failure path ever resets isDownLoading — one failed ts-segment hangs the whole m3u8 download forever

- Category B, severity medium, confidence confirmed
- Features: MEDIA-34
- Document: `huawei_industry_tree/13_media_entertainment/docs/34_m3u8_download.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/m3u8_download-0000002492523889
- Why: Any single segment error (or re-download into an existing path) permanently hangs the download with loading state stuck.
- Fix: Register 'fail'; reset isDownLoading in err/fail.

### `HW-13-0076` - videoInfo[0] dereferenced without a parse-result check — the worker crashes on the shipped placeholder URL / any network error

- Category B, severity medium, confidence confirmed
- Features: MEDIA-34
- Document: `huawei_industry_tree/13_media_entertainment/docs/34_m3u8_download.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/m3u8_download-0000002492523889
- Why: First-run behavior is a worker crash with the loading state stuck.
- Fix: Check videoInfo.length before indexing.

### `HW-13-0081` - stop() unregisters 'periodReach' though only 'writeData' was registered; start() re-registers per call — listeners accumulate

- Category B, severity medium, confidence confirmed
- Features: MEDIA-36
- Document: `huawei_industry_tree/13_media_entertainment/docs/36_pcm_transcode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_transcode-0000002513212618
- Why: Play/stop cycles stack writeData callbacks that double-advance the read offset, corrupting later playbacks.
- Fix: Fix the event name; add a registration guard.

### `HW-13-0082` - Encoder input loop never checks gcount() or open() — the last frame encodes garbage and a missing input spins forever

- Category B, severity medium, confidence confirmed
- Features: MEDIA-36
- Document: `huawei_industry_tree/13_media_entertainment/docs/36_pcm_transcode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_transcode-0000002513212618
- Why: Every m4a/amr transcode ends with a garbage frame; a missing input file turns the input thread into an infinite loop.
- Fix: Use gcount() and validate open().

### `HW-13-0084` - PluginRender destroy nulls the pointer BEFORE delete — the whole render stack (EGL context, NativeImage, render thread) leaks

- Category D, severity medium, confidence confirmed
- Features: MEDIA-37
- Document: `huawei_industry_tree/13_media_entertainment/docs/37_opengl_offscreen_render.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/opengl_offscreen_render-0000002553789479
- Why: Every surface destroy leaks the GL context and leaves the render thread running.
- Fix: Swap the two statements.

### `HW-13-0085` - Texture-id sentinels inconsistent (9999 for create, 0 for destroy) — GL resources can never be recreated after a recycle; transform is hardcoded to FlipV

- Category B, severity medium, confidence confirmed
- Features: MEDIA-37
- Document: `huawei_industry_tree/13_media_entertainment/docs/37_opengl_offscreen_render.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/opengl_offscreen_render-0000002553789479
- Why: After any surface change the text overlay disappears and the video texture is not recreated; rotated videos render with the wrong orientation.
- Fix: Standardize on 0 + glGenTextures guard; use the queried transform.

### `HW-13-0086` - Doc's step-2 snippet creates a NEW ThreadWorker inside onClick — an unstoppable metronome that leaks a worker per tap (code does it correctly)

- Category E, severity medium, confidence confirmed
- Features: MEDIA-38
- Document: `huawei_industry_tree/13_media_entertainment/docs/38_news-v1_2-ts_126.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/news-v1_2-ts_126-0000002444273305
- Why: Following the doc, the 'stop' message goes to a fresh worker with no running timer while the old interval keeps beating.
- Fix: Fix the snippet.

### `HW-13-0087` - Double-start race: isPlaying is set by the worker round-trip (~1 s), and the worker overwrites its timer id — orphaned intervals beat forever

- Category B, severity medium, confidence confirmed
- Features: MEDIA-38
- Document: `huawei_industry_tree/13_media_entertainment/docs/38_news-v1_2-ts_126.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/news-v1_2-ts_126-0000002444273305
- Why: Beats fire twice per second forever, surviving stop.
- Fix: Set state on click; if (timer) return in the worker.

### `HW-13-0088` - Thumbnail aspect math inverted: height = width * (w/h) — doc reproduces the same formula

- Category B, severity medium, confidence confirmed
- Features: MEDIA-39
- Document: `huawei_industry_tree/13_media_entertainment/docs/39_video_thumbnail.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_thumbnail-0000002565954707
- Why: A 16:9 video requests a 200x356 portrait frame instead of 200x112; only the popup's re-stretch hides it.
- Fix: Divide instead of multiply.

### `HW-13-0091` - ASR onError shuts the engine but never clears the 5 ms interval or closes the audio fd

- Category D, severity medium, confidence confirmed
- Features: MEDIA-41
- Document: `huawei_industry_tree/13_media_entertainment/docs/41_video_subtitle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_subtitle-0000002568904821
- Why: After any recognition error the timer spins at 5 ms against a dead engine forever.
- Fix: clearInterval + closeSync in onError/onComplete.

### `HW-13-0092` - writeOffset reset to 0 BEFORE finish() — the final subtitle gets an inverted/zero time range and never displays

- Category B, severity medium, confidence probable
- Features: MEDIA-41
- Document: `huawei_industry_tree/13_media_entertainment/docs/41_video_subtitle.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_subtitle-0000002568904821
- Why: The last recognized sentence is stored with endTime 0 (< startTime) and can never match the display range check.
- Fix: Move the reset (or snapshot the offset).

### `HW-13-0093` - PCM renderer ignores readSync's return and can read past the rawfile window; after EOF the player is permanently silent while UI/AVSession still show 'playing'

- Category B, severity medium, confidence confirmed
- Features: MEDIA-42
- Document: `huawei_industry_tree/13_media_entertainment/docs/42_demo_audio_session.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/demo_audio_session-0000002668471732
- Why: End of track = stale-tail audio, then a dead player that the whole 'state sync' demo shows as playing.
- Fix: Clamp length; on EOF set isPlaying=false, update AVSession, allow rewind.

### `HW-13-0094` - A state-sync demo with no audioInterrupt handling in any of its five players

- Category D, severity medium, confidence confirmed
- Features: MEDIA-42
- Document: `huawei_industry_tree/13_media_entertainment/docs/42_demo_audio_session.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/demo_audio_session-0000002668471732
- Why: Another app taking focus pauses the stream while isPlaying and AVSession stay PLAY — desynchronizing exactly what the sample teaches.
- Fix: Register audioInterrupt in each player.

### `HW-13-0096` - 1 sample project swallows errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: MEDIA-30
- Document: `huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-13-0100` - formatVideo swallows per-clip conversion failures, and mergeVideo then lists .ts files that were never produced

- Category B, severity medium, confidence confirmed
- Features: MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/25_splash_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page-0000002303412054
- Why: The catch inside the conversion loop turns a failed ffmpeg run into a log line, so mergeVideo cannot tell that videoN.ts is missing. It writes a concat manifest naming every clip by index, and videoMultMerge is handed a list referring to files that do not exist. The merge then fails or silently produces a shorter video, and the failure is attributed to the merge step rather than to the conversion that actually failed. Because the loop indexes names by position, one failed conversion also shifts nothing, so the manifest looks well formed.
- Fix: Have formatVideo return the list of successfully converted file names, and build the concat manifest from that list. Abort the merge with a clear error when the list is shorter than the input.

### `HW-13-0101` - mergeVideo leaks the manifest file descriptor whenever the merge rejects, because the close is after the await instead of in a finally

- Category B, severity medium, confidence confirmed
- Features: MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/25_splash_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page-0000002303412054
- Why: The close on line 104 is only reached when every await before it resolves. The merge callback rejects on any non-zero ffmpeg return code, which is the ordinary failure path this code explicitly handles, and the write on line 80 can fail too. Control then jumps to the catch, which logs and returns false without closing. Each failed merge attempt leaks one descriptor, and the user is expected to retry after a failure, so the leak accumulates exactly when the feature is misbehaving.
- Fix: Move the open outside the try, and close the descriptor in a finally block that runs on both paths.

### `HW-13-0102` - saveVideo leaks both the source and the destination descriptor when the copy into the media library fails

- Category B, severity medium, confidence confirmed
- Features: MEDIA-25
- Document: `huawei_industry_tree/13_media_entertainment/docs/25_splash_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page-0000002303412054
- Why: copyFile is awaited between the two opens and the two closes. A failure there, or a failure of the second open after the first succeeded, jumps straight to the catch, which only logs. The destination descriptor points into the media library through a dialog-granted uri, so leaking it also holds the grant open. Saving is the terminal step of the merge flow and is the action a user repeats after a failure.
- Fix: Declare both File variables before the try, and close whichever ones are non-null in a finally block.

### `HW-13-0105` - The compositor waits for rendering with a fixed 50 ms sleep instead of glFinish, so the readback is a race on slow devices

- Category B, severity medium, confidence confirmed
- Features: MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/40_audio-v1_2-ts_64.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_64-0000002444273313
- Why: GL commands are queued, not synchronous. The sleep is an attempt to let the queue drain, but 50 ms is unrelated to the real cost, which scales with the image resolution and the device GPU. On a slow device or a large photo the draw is still in flight and glReadPixels captures a partly rendered frame, producing an image with missing or half-composited text. On a fast device the sleep is 50 ms of latency added to every save for nothing. glFinish blocks exactly as long as needed and no longer, and glReadPixels on the default framebuffer implies a flush on conforming drivers, which is why the failure is intermittent rather than constant.
- Fix: Replace the sleep_for with glFinish() immediately after glDrawArrays, and drop the thread and chrono includes if nothing else uses them.

### `HW-13-0106` - The page is published under an audio URL slug although its subject is OpenGL ES image and text composition

- Category E, severity medium, confidence confirmed
- Features: MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/40_audio-v1_2-ts_64.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_64-0000002444273313
- Why: The slug is the stable identifier the page is linked and searched by. audio-v1_2-ts_64 suggests an audio document with a version and a task number, so a reader looking for image composition will not find this page and a reader following the slug expecting audio content lands on a graphics article. This is the second page in this corpus with the defect, after 11_news_reading/docs/24_auto_flip_read.md, which suggests slugs are being assigned from a task tracker rather than from the content.
- Fix: Republish under a slug naming the subject and redirect the current one. Check the rest of the architecture-guides set for slugs inherited from unrelated tasks.

### `HW-13-0001` - Music demo declares MICROPHONE and KEEP_BACKGROUND_RUNNING but has no recording or continuous-task code

- Category D, severity low, confidence confirmed
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Excess user-grant microphone permission in an audio player is a privacy red flag, and the background-running permission promises background playback the sample does not implement.
- Fix: Remove both declarations, or add the AVSession/continuous-task background flow the architecture doc describes.

### `HW-13-0004` - Systematic: three media docs' constraints claim API 20 while their samples target API 16/17

- Category E, severity low, confidence confirmed
- Features: MEDIA-18
- Document: `huawei_industry_tree/13_media_entertainment/docs/18_video_playlist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_playlist-0000002286827110
- Why: The constraint overstates the requirement by 3-4 API versions: readers on 5.0.4/5.0.5 are told the samples need API 20 although the projects are configured for exactly those older releases.
- Fix: Align the three 约束与限制 sections with the samples' actual compatibleSdkVersion (or bump the samples).

### `HW-13-0006` - Worker cancels its setInterval with clearTimeout

- Category B, severity low, confidence probable
- Features: MEDIA-38
- Document: `huawei_industry_tree/13_media_entertainment/docs/38_news-v1_2-ts_126.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/news-v1_2-ts_126-0000002444273305
- Why: The Timer API defines clearInterval as the cancel function for setInterval; relying on clearTimeout accepting an interval id is engine-dependent behavior, and the mismatch confuses readers about which timer type is in use.
- Fix: Replace clearTimeout with clearInterval in Worker.ets.

### `HW-13-0010` - Ranking rows pass the song title as the author argument — authors are never displayed

- Category B, severity low, confidence confirmed
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Find tab shows '歌曲1 歌曲1' instead of '歌曲1 歌手1'.
- Fix: Fix the arguments.

### `HW-13-0011` - Time formatter boundary `second > 60` renders exactly one minute as 00:60; random-mode helper is deterministic and can spin forever

- Category B, severity low, confidence confirmed
- Features: MEDIA-01
- Document: `huawei_industry_tree/13_media_entertainment/docs/01_practice-audio-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-audio-app-architecture-v1-0000002041168218
- Why: Progress label shows the invalid 00:60 at the minute boundary; the random-play helper is broken by construction if ever wired up.
- Fix: Use >= 60; drop the constant setSeed.

### `HW-13-0013` - formatFileSize mixes a decimal threshold (>=1000) with a binary divisor (/1024)

- Category B, severity low, confidence confirmed
- Features: MEDIA-03
- Document: `huawei_industry_tree/13_media_entertainment/docs/03_get_video_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_video_info-0000002273254857
- Why: 1000-byte file shows as 0.98KB; unit boundaries flip inconsistently on the sample's only info card.
- Fix: Match threshold and divisor.

### `HW-13-0016` - Doc snippet reads res.compressPath/res.compressSize — fields that do not exist on the sample's FileProcessResult

- Category D, severity low, confidence confirmed
- Features: MEDIA-04
- Document: `huawei_industry_tree/13_media_entertainment/docs/04_video_compress_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292
- Why: Copying the doc snippet produces compile errors; the doc was not updated after the rename.
- Fix: Update the snippet to filePath/fileSize.

### `HW-13-0017` - Number($r('app.float.*_zindex')) is always NaN — the resource-driven zIndex values never apply

- Category B, severity low, confidence confirmed
- Features: MEDIA-05
- Document: `huawei_industry_tree/13_media_entertainment/docs/05_custom_slider.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_slider-0000002284049981
- Why: The intended stacking only works because declaration order happens to match; any reorder breaks it.
- Fix: Use resourceManager.getNumber or numeric constants.

### `HW-13-0019` - removeAll splices inside a forward loop without i-- — consecutive matches are skipped (dead code today, canonical bug in the reusable data source)

- Category B, severity low, confidence confirmed
- Features: MEDIA-06
- Document: `huawei_industry_tree/13_media_entertainment/docs/06_hiararchicle_filtering.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hiararchicle_filtering-0000002287162465
- Why: The element shifted into position i after a splice is never examined; the teaching-grade BasicDataSource ships the classic mutate-while-iterating bug.
- Fix: Decrement i after each splice.

### `HW-13-0020` - Doc references NewButtonType which exists nowhere in the sample (actual: SelectionButtonType)

- Category D, severity low, confidence confirmed
- Features: MEDIA-06
- Document: `huawei_industry_tree/13_media_entertainment/docs/06_hiararchicle_filtering.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hiararchicle_filtering-0000002287162465
- Why: Readers search ButtonData.ets for a type that does not exist.
- Fix: Fix the doc.

### `HW-13-0021` - Constraints claim 'API Version 24 Release' — an API level that does not exist

- Category E, severity low, confidence probable
- Features: MEDIA-06
- Document: `huawei_industry_tree/13_media_entertainment/docs/06_hiararchicle_filtering.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hiararchicle_filtering-0000002287162465
- Why: The environment requirement is unverifiable/wrong (distinct from the known API-20 boilerplate finding MEDIA-18_001).
- Fix: Correct to API 20.

### `HW-13-0025` - debounceTimer guard `!== null` is always true (declared without initializer, never set null)

- Category B, severity low, confidence confirmed
- Features: MEDIA-07
- Document: `huawei_industry_tree/13_media_entertainment/docs/07_sub_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sub_window-0000002254941572
- Why: The pending-collapse detection the comments describe never actually branches; clearTimeout(undefined) is a silent no-op.
- Fix: Track the timer id consistently.

### `HW-13-0026` - timeFormat lacks % 3600 on minutes: one hour renders as 1:61:01

- Category B, severity low, confidence confirmed
- Features: MEDIA-08
- Document: `huawei_industry_tree/13_media_entertainment/docs/08_orientation_switching.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/orientation_switching-0000002255691184
- Why: Any video >= 1 h shows an invalid time; the helper is presented as reusable.
- Fix: Add the modulo.

### `HW-13-0028` - GET_NETWORK_INFO's usedScene references nonexistent 'FormAbility'

- Category B, severity low, confidence confirmed
- Features: MEDIA-09
- Document: `huawei_industry_tree/13_media_entertainment/docs/09_data_network_pause_playback.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_network_pause_playback-0000002291777477
- Why: The declared usage scene points at an ability that does not exist.
- Fix: Fix the ability name.

### `HW-13-0029` - Lyric highlight timeline is anchored to first render, not to the song clock — the 00:01.01 initial offset is ignored and three clocks drift

- Category B, severity low, confidence confirmed
- Features: MEDIA-10
- Document: `huawei_industry_tree/13_media_entertainment/docs/10_lyric_dynamic_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/lyric_dynamic_effect-0000002291790569
- Why: Lyrics lead the timers by the initial offset minus load latency and drift as animateTo onFinish gaps accumulate.
- Fix: Drive highlights from the same timer that drives progress.

### `HW-13-0031` - Doc's playLrc snippet pastes processEnhancedLrc's sort block into the onFinish callback — non-compiling and absent from the sample

- Category D, severity low, confidence confirmed
- Features: MEDIA-10
- Document: `huawei_industry_tree/13_media_entertainment/docs/10_lyric_dynamic_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/lyric_dynamic_effect-0000002291790569
- Why: Copying the doc yields code that cannot compile.
- Fix: Remove the pasted block.

### `HW-13-0035` - Progress Slider enables SLIDE_ONLY interaction but has no onChange — dragging does nothing

- Category B, severity low, confidence confirmed
- Features: MEDIA-12
- Document: `huawei_industry_tree/13_media_entertainment/docs/12_video_switching_association.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_switching_association-0000002296452929
- Why: The thumb drags then snaps back on the next onUpdate; setCurrentTime is never called.
- Fix: Add onChange with controller.setCurrentTime.

### `HW-13-0038` - Any import error returns an empty map that the caller assigns over the whole playlist

- Category B, severity low, confidence confirmed
- Features: MEDIA-13
- Document: `huawei_industry_tree/13_media_entertainment/docs/13_music_player.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_player-0000002296726005
- Why: One failed/cancelled import clears the user's entire visible playlist.
- Fix: Initialize newMap from existing entries (or return early on error).

### `HW-13-0041` - Failed GIF generation still enables Save and sets the output path — only the toast differs

- Category B, severity low, confidence confirmed
- Features: MEDIA-14
- Document: `huawei_industry_tree/13_media_entertainment/docs/14_video_demo_videocreategif.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_demo_videocreategif-0000002312993113
- Why: Save then opens a nonexistent file (and hits the closeSync(undefined) fault above).
- Fix: Branch on code before enabling Save.

### `HW-13-0042` - Comment/doc say one preset bullet comment every 3 s; the interval is 1000 ms

- Category D, severity low, confidence confirmed
- Features: MEDIA-15
- Document: `huawei_industry_tree/13_media_entertainment/docs/15_bullet_comment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bullet_comment-0000002284943480
- Why: Comments generate 3x faster than designed; the 20 presets recycle every 20 s.
- Fix: Use 3000 (or fix the comment).

### `HW-13-0043` - Submitting a comment resets the input to the PLACEHOLDER resource — the next submit pushes the placeholder as a fake comment

- Category B, severity low, confidence confirmed
- Features: MEDIA-16
- Document: `huawei_industry_tree/13_media_entertainment/docs/16_short_video_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/short_video_review-0000002286277826
- Why: Placeholder wording becomes real text and can be submitted as a comment.
- Fix: Assign the empty string.

### `HW-13-0044` - Every open of the comment dialog registers another permanent window 'keyboardHeightChange' listener

- Category B, severity low, confidence confirmed
- Features: MEDIA-16
- Document: `huawei_industry_tree/13_media_entertainment/docs/16_short_video_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/short_video_review-0000002286277826
- Why: Stale listeners keep firing after the dialog is destroyed and call close() on dead controllers.
- Fix: Store and remove the callback.

### `HW-13-0052` - checkVideoSize passes exactly when it has no data — the resolution guard is bypassed when size lookups fail

- Category B, severity low, confidence confirmed
- Features: MEDIA-20
- Document: `huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
- Why: Mismatched-resolution videos are merged, failing later with an opaque ffmpeg error.
- Fix: Return false unless videoSizes.length === videoList.length.

### `HW-13-0054` - visibleMap initializer duplicates 'rain.mp4' and omits 'heart.mp4'

- Category B, severity low, confidence confirmed
- Features: MEDIA-21
- Document: `huawei_industry_tree/13_media_entertainment/docs/21_multi_camera_video.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_camera_video-0000002297933590
- Why: Works only because undefined is falsy like the intended false; any logic distinguishing 'explicitly hidden' from 'unknown key' breaks.
- Fix: Fix the duplicate literal.

### `HW-13-0056` - Random emoji index divides by 255 — byte 255 indexes emojis[18] (out of range)

- Category B, severity low, confidence confirmed
- Features: MEDIA-22
- Document: `huawei_industry_tree/13_media_entertainment/docs/22_video_like.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_like-0000002332695009
- Why: 1/256 of taps draw 'undefined' instead of an emoji.
- Fix: Use /256.

### `HW-13-0059` - Slider seeks with duration still -1 before prepared — negative seek on an unprepared player

- Category B, severity low, confidence confirmed
- Features: MEDIA-24
- Document: `huawei_industry_tree/13_media_entertainment/docs/24_video_player_resume.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_player_resume-0000002299260088
- Why: Touching the always-enabled slider before prepare completes calls seek(-value) in an invalid state.
- Fix: Guard on getDuration() > 0.

### `HW-13-0060` - Back from the detail page force-resumes playback even when the user paused

- Category B, severity low, confidence confirmed
- Features: MEDIA-26
- Document: `huawei_industry_tree/13_media_entertainment/docs/26_video_transition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_transition-0000002349370885
- Why: A pause can never survive the back transition; the home card always resumes.
- Fix: Only play() if it was playing.

### `HW-13-0068` - OpenMode flags combined with logical || (evaluates to CREATE only); FileUtil closes fd 0 on error and returns a path never written

- Category B, severity low, confidence confirmed
- Features: MEDIA-30
- Document: `huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
- Why: Signature `||-instead-of-|` bug plus wrong-fd close and a bogus success path feeding the ffmpeg step.
- Fix: Use |, guard the close, return undefined on error.

### `HW-13-0071` - App startup audibly plays all three drum sounds and fires haptics as 'preloading'

- Category B, severity low, confidence confirmed
- Features: MEDIA-31
- Document: `huawei_industry_tree/13_media_entertainment/docs/31_drum.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drum-0000002332280266
- Why: Every cold start plays drum/cymbal sounds and vibrates three times without user interaction.
- Fix: Call loadSound only.

### `HW-13-0077` - saveM3U8Playlist not awaited — completion posted with JSON.stringify(Promise) ('{}') before the playlist is written; download dir name carries a stray '}'

- Category B, severity low, confidence confirmed
- Features: MEDIA-34
- Document: `huawei_industry_tree/13_media_entertainment/docs/34_m3u8_download.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/m3u8_download-0000002492523889
- Why: The success message carries a useless payload racing the actual write; every download directory is named 'download<ts>}'.
- Fix: Add await; remove the brace.

### `HW-13-0079` - decibelRMSCount===0 guard sets 0 but falls through to 0/0 — NaN amplitude pushed into the waveform

- Category B, severity low, confidence confirmed
- Features: MEDIA-35
- Document: `huawei_industry_tree/13_media_entertainment/docs/35_pcm_audio_edit.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_audio_edit-0000002476817872
- Why: A periodReach before any samples corrupts the drawn waveform with NaN heights.
- Fix: return after the guard.

### `HW-13-0089` - One rejected fetchFrame latches isFetching forever — thumbnails stop for the session

- Category B, severity low, confidence confirmed
- Features: MEDIA-39
- Document: `huawei_industry_tree/13_media_entertainment/docs/39_video_thumbnail.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_thumbnail-0000002565954707
- Why: A single transient AVImageGenerator error permanently disables the popup preview.
- Fix: try/finally.

### `HW-13-0095` - AVPlaybackState dedupe compares object identity — always false, dead guard

- Category B, severity low, confidence confirmed
- Features: MEDIA-42
- Document: `huawei_industry_tree/13_media_entertainment/docs/42_demo_audio_session.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/demo_audio_session-0000002668471732
- Why: Duplicate state updates always sent; the guard is dead code.
- Fix: Compare .state values.

### `HW-13-0103` - The speedDone listener is the only AVPlayer callback that aboutToDisappear does not unregister

- Category B, severity low, confidence confirmed
- Features: MEDIA-33
- Document: `huawei_industry_tree/13_media_entertainment/docs/33_video_mirroring.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_mirroring-0000002344967801
- Why: The speedDone handler closes over the component and calls this.currentAVSession.setAVPlaybackState. release() is asynchronous and its result is discarded, so between the off calls and the actual teardown the page has four detached callbacks and one still live. If release rejects, that one survives entirely and fires against a destroyed component. The omission is also a maintenance trap: the four-line off block reads as complete, so a reader adding a sixth listener follows the same incomplete pattern.
- Fix: Add this.avPlayer.off('speedDone') to the block in aboutToDisappear, and handle the promise returned by release() so a failed teardown is visible.

### `HW-13-0107` - The project tree misspells the entrybackupability directory as entrybackupablility

- Category E, severity low, confidence confirmed
- Features: MEDIA-40
- Document: `huawei_industry_tree/13_media_entertainment/docs/40_audio-v1_2-ts_64.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_64-0000002444273313
- Why: The directory name is referenced from module.json5 through the srcEntry of the backup ExtensionAbility, so recreating the project from the tree as written produces a path that does not resolve. It is also the only spelling of this directory in the corpus that carries the extra syllable, so a reader cross-checking against another industry sees a difference that is not real.
- Fix: Correct entrybackupablility to entrybackupability in the project tree.

