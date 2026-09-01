---
id: MEDIA-10
title: Karaoke lyrics - word-by-word colour sweep driven by a chained animateTo
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/10_lyric_dynamic_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/lyric_dynamic_effect-0000002291790569
sample: huawei_industry_tree/13_media_entertainment/downloads/LyricDynamicEffect.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [common, hilog, util, window, "UIContext.animateTo", AnimateParam, FinishCallbackType, linearGradient, blendMode, BlendApplyType, "resourceManager.getRawFileContent", "util.TextDecoder", List, Scroller, ListScroller, TextTimer, TextTimerController, Progress, "UIContext.getPromptAction", showToast, "@StorageProp", setInterval]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0029, HW-13-0030, HW-13-0031, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when text has to **light up in time with audio**: karaoke
lines, a read-along transcript, subtitles that fill as they are spoken, a
word-level progress readout for a language app.

The technique is narrower and more interesting than "highlight the current
line". Each word is painted by a `linearGradient` whose stop position is an
animated state variable, so the colour boundary sweeps *through* the glyphs
rather than snapping between them. `animateTo` moves that one number from `0`
to `1` over the word's duration, and its `onFinish` schedules the next word.
The lyric file supplies the durations.

**This sample is a good drawing technique attached to a bad clock.** The
sweep is driven by a chain of `animateTo` callbacks started from the first
word's `onAppear`, not from a playback position, and there is no audio at all
- the transport buttons are toasts. Take the rendering; replace the timing
source with your player's `timeUpdate` before shipping anything
(`HW-13-0029`). And note before you plan around it that the "scrolling" the
document advertises does not exist (`HW-13-0030`).

## Feature checklist

- Reads an enhanced-LRC file from `rawfile` at page entry and decodes it as
  UTF-8.
- Parses per-word timestamps (`[mm:ss.xx]word[mm:ss.xx]word...`) into lines of
  words, each with a start time and a duration derived from the next
  timestamp.
- Renders every line as a wrapping flow of words, centred, in a `List`.
- The active line is fully opaque; the others sit at 0.8.
- Inside the active line, already-sung words are solid red, upcoming words
  white, and the current word carries a red-to-white boundary that sweeps
  left to right over exactly that word's duration.
- A linear `Progress` bar and a `TextTimer` count the song out; the timer
  pauses itself at the last timestamp.
- The transport row (play, previous, next, repeat mode) raises a "demo only"
  toast.

## Architecture

One `entry` module, one page, one data model. Everything except the two data
classes lives in `MainPage.ets`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets     full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/LrcSong.ets                 Word { index, text, startTime, lineDuration } + LrcLine
└── pages/MainPage.ets                @Entry - parse, animate, and the whole player UI
```

`entry/src/main/resources/rawfile/Sing.lrc` carries the lyric. Its first
line is `[00:01.01]Dream[00:01.25] It[00:01.50] Possible`, which is worth
looking at: the timestamps are per **word**, not per line, and the words carry
their leading space, which is how the flow layout keeps English spacing.

The documented tree matches the zip - name for name and case for case, which
is not a given in this industry (`HW-13-0003`).

**The design decision worth copying** is the two-level model. A `LrcLine` is
not a string plus a time; it is a `Word[]`, and every `Word` carries both its
own `startTime` and the `lineDuration` until the next timestamp:

```typescript
export class Word {
  index: number = 0;
  text: string = '';
  startTime: number = 0;
  lineDuration: number = 0;
}
```

Precomputing the duration is what lets the renderer be dumb. The animation
never has to look ahead, compare against a clock, or find the next word: it
animates for `lineDuration` and then asks for the next index. That is also
exactly why the sample drifts - a chain of durations has no absolute
reference, so every scheduling gap accumulates forever (`HW-13-0029`).

**The design decision worth avoiding** is starting the chain from
`onAppear`. The first word's `onAppear` fires when ArkUI builds that node,
which is neither `00:01.01` (the word's own timestamp) nor the moment
playback starts. Together with the `setInterval` that drives `Progress` from
`aboutToAppear` and the `TextTimer` that starts from its own `onAppear`, the
page runs three clocks that were never the same clock.

## Implementation steps

1. **Load the lyric as bytes and decode explicitly.**
   `resourceManager.getRawFileContent` returns a `Uint8Array`; put it through
   `util.TextDecoder.create('utf-8', { ignoreBOM: true })` - a BOM in the LRC
   would otherwise corrupt the first timestamp match.
2. **Parse with one global regex** capturing minutes, seconds, centiseconds
   and the text that follows, and convert to milliseconds as
   `mm * 60000 + ss * 1000 + ms * 10`. The `xx` field is centiseconds, hence
   `* 10`.
3. **Derive each word's duration from the next timestamp in the whole song**,
   not from the next word in the line, so the last word of a line runs until
   the first word of the next.
4. **Sort lines by start time once, after parsing**, and keep that block in
   the parse function - the document's step 4 pastes it into the animation
   callback where it cannot compile (`HW-13-0031`).
5. **Give each word its own gradient** with the boundary at the animated
   `value`, and use `BlendMode.DST_IN` on the `Text` inside a
   `BlendApplyType.OFFSCREEN` `Row` so the gradient masks the glyphs instead
   of painting a rectangle behind them.
6. **Animate `value` from 0 to 1** with `duration` set to the word's
   `lineDuration`, `Curve.Linear` (a sweep must be linear) and
   `finishCallbackType: FinishCallbackType.LOGICALLY`.
7. **Advance in `onFinish`**: reset `value`, step `currentIndex`, and when the
   line is exhausted step `currentLineIndex` and continue from its first word.
8. **Drive the highlight from the same clock as the progress bar**, not from a
   chained `onFinish` and not from `onAppear` (`HW-13-0029`).
9. **Scroll the list when the line changes** - the `Scroller` is created and
   bound but never used, so highlighting simply continues below the 416vp
   viewport (`HW-13-0030`).
10. **Clear the interval in `aboutToDisappear`.** The sample does this
    correctly.

## Verified snippets

All snippets are from `LyricDynamicEffect.zip`. Corrected forms are marked.

**Parsing enhanced LRC - `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
processEnhancedLrc(lyric: string[]): void {
  let lrcList: LrcLine[] = [];
  let wordRegex: RegExp = /\[(\d{2}):(\d{2})\.(\d{2})\]([^[\]]+)/g;
  let wordIndex = 0;

  lyric.forEach((line: string) => {
    let words: Word[] = [];
    let lineStartTime: number = 0;
    let match: RegExpExecArray | null;

    while ((match = wordRegex.exec(line)) !== null) {
      let mm: number = Number.parseInt(match[1], 10);
      let ss: number = Number.parseInt(match[2], 10);
      let ms: number = Number.parseInt(match[3], 10);
      let text: string = match[4].trim();
      let time: number = mm * 60 * 1000 + ss * 1000 + ms * 10;

      if (words.length === 0) {
        lineStartTime = time;
      }

      let nextTime = this.extractAllTimestampsFromLyrics(lyric)[wordIndex + 1];
      let lineTimeDuration: number = nextTime !== undefined ? (nextTime! - time) : 1000;
      words.push({ index: wordIndex, text: text, startTime: time, lineDuration: lineTimeDuration });
      wordIndex++;
    }
    if (words.length > 0) {
      lrcList.push({ lineStartTime: lineStartTime, words: words });
    }
  });

  // Sort by time and calculate duration
  lrcList.sort((a, b) => a.lineStartTime - b.lineStartTime);
  if (lrcList.length > 0) {
    this.duration = lrcList[lrcList.length - 1].words[lrcList[lrcList.length - 1].words.length - 1].startTime + 3000;
  }
  this.mLrcEntryList = lrcList;
}
```

**The regex is the format specification.** `\[(\d{2}):(\d{2})\.(\d{2})\]`
matches one timestamp and `([^[\]]+)` takes everything up to the next
bracket, so one `exec` loop over a line yields its words in order. Because
`wordRegex` is declared with `g` and reused across `forEach` iterations, its
`lastIndex` carries over - here that is harmless only because every
`while` loop runs to `null`, which resets it. Declaring the regex inside the
`forEach` would be safer.

`text: match[4].trim()` is the one lossy step: the leading spaces that give
English lyrics their spacing are stripped here and re-added at render time by
a conditional 4vp margin on words matching `/[a-zA-Z0-9]/`. It works for
mixed Chinese/English, and it is why the flow layout does not need explicit
spacers.

Note the cost: `extractAllTimestampsFromLyrics(lyric)` re-scans and re-sorts
the *entire* song once per word. For a 40-line lyric with ~10 words a line
that is 400 full passes. Hoist it above the loop.

**The sweep - same file** (as shipped)

```typescript
ForEach(line.words, (word: Word, wordIndex: number) => {
  Row() {
    Text(word.text)
      .fontSize(16)
      .blendMode(BlendMode.DST_IN, BlendApplyType.OFFSCREEN);
  }
  .margin({ right: word.text.match(/[a-zA-Z0-9]/) ? 4 : 0, bottom: 2 })
  .blendMode(BlendMode.SRC_OVER, BlendApplyType.OFFSCREEN)
  .linearGradient({
    direction: GradientDirection.Right,
    colors: (() => {
      if (this.currentLineIndex !== lineIndex) {
        return [['#FFFFFFFF', 0], ['#FFFFFFFF', 1.0]];       // inactive line: all white
      }
      if (wordIndex < this.currentIndex) {
        return [[0xFF1949, 0.0], [0xFF1949, 1.0]];           // already sung: all red
      } else if (wordIndex === this.currentIndex) {
        return [                                              // singing: boundary at this.value
          [0xFF1949, 0.0],
          [0xFF1949, this.value],
          ['#FFFFFFFF', this.value],
          ['#FFFFFFFF', 1.0]
        ];
      } else {
        return [['#FFFFFFFF', 0], ['#FFFFFFFF', 1.0]];       // not yet sung: all white
      }
    })()
  })
  .onAppear(() => {
    this.lrcDuration = line.words[wordIndex].lineDuration;
    if (lineIndex === this.currentLineIndex && wordIndex === 0) {
      this.playLrc(line.words[wordIndex].lineDuration);
    }
  });
});
```

**Two colour stops at the same offset make a hard edge.** `[0xFF1949, value]`
immediately followed by `['#FFFFFFFF', value]` is a discontinuity, so the
gradient is not a fade - it is a moving boundary. Animating `value` from 0 to
1 slides that boundary across the word. Change the two `value` stops to
`value - 0.1` and `value + 0.1` and you get a soft leading edge instead;
that is the only knob needed to tune the look.

**The blend pair is what confines the paint to the glyphs.** The parent `Row`
declares `BlendMode.SRC_OVER` with `BlendApplyType.OFFSCREEN`, which makes it
an offscreen compositing root holding the gradient; the child `Text` declares
`BlendMode.DST_IN`, which keeps the destination (the gradient) only where the
source (the glyph coverage) is opaque. Drop either half and the effect
becomes a coloured rectangle behind white text.

The `onAppear` on the second-to-last line is the timing bug in one place: it
starts the animation chain when the node is built.

**The animation chain - same file** (as shipped)

```typescript
playLrc(duration: number) {
  this.uiContext.animateTo({
    duration: duration,
    finishCallbackType: FinishCallbackType.LOGICALLY,
    curve: Curve.Linear,
    iterations: 1,
    onFinish: () => {
      this.value = 0;
      this.currentIndex++;
      let currentLine = this.mLrcEntryList[this.currentLineIndex];
      if (this.currentIndex < currentLine.words.length) {
        this.lrcDuration = currentLine.words[this.currentIndex].lineDuration;
        this.playLrc(this.lrcDuration);
      } else {
        this.currentIndex = 0;
        this.currentLineIndex++;
        if (this.currentLineIndex < this.mLrcEntryList.length) {
          let nextLine = this.mLrcEntryList[this.currentLineIndex];
          this.lrcDuration = nextLine.words[this.currentIndex].lineDuration;
          this.playLrc(this.lrcDuration);
        }
      }
    }
  }, () => {
    this.value = 1;
  });
}
```

**The closure is one assignment.** `this.value = 1` is the entire animated
change; `animateTo` interpolates it over `duration` and every dependent
gradient stop follows. `Curve.Linear` is mandatory for a sweep - any easing
curve makes the boundary hesitate at word boundaries.
`FinishCallbackType.LOGICALLY` fires `onFinish` when the state has logically
reached its end value rather than when the last frame is drawn, which is the
right choice when the callback's job is to schedule the next segment.

The recursion is also the drift. Each `onFinish` costs a frame or more before
the next `animateTo` starts, and nothing ever re-syncs against
`word.startTime`, so error accumulates monotonically for the length of the
song (`HW-13-0029`). The document's version of this method is worse still: it
pastes `lrcList.sort(...)` and `this.mLrcEntryList = lrcList` into the
`onFinish` body, where `lrcList` is not in scope (`HW-13-0031`).

**Anchoring to a real clock and scrolling - same file** (corrected, see `HW-13-0029`, `HW-13-0030`)

```typescript
// FIX: one clock. Replace the onAppear-started chain and the bare setInterval.
private tick(positionMs: number): void {
  this.progressValue = positionMs;

  for (let l = 0; l < this.mLrcEntryList.length; l++) {
    const words = this.mLrcEntryList[l].words;
    for (let w = 0; w < words.length; w++) {
      const start = words[w].startTime;
      if (positionMs >= start && positionMs < start + words[w].lineDuration) {
        if (l !== this.currentLineIndex) {
          this.currentLineIndex = l;
          this.scroller.scrollToIndex(l, true, ScrollAlign.CENTER);   // FIX: the scroller was dead
        }
        this.currentIndex = w;
        this.value = (positionMs - start) / words[w].lineDuration;    // FIX: derived, not chained
        return;
      }
    }
  }
}
```

Positions come from the player - `avPlayer.on('timeUpdate', ...)` in a real
build, `setInterval` only in a demo. Deriving `value` from the position
instead of animating it means a seek is free: the highlight is wherever the
song is, with no chain to unwind. If the per-frame smoothness of `animateTo`
is wanted, keep it, but restart it against `word.startTime` on every line
change so error cannot accumulate.

`scrollToIndex(index, smooth, align)` needs a `Scroller` bound to the `List`,
which this sample already has (`List({ initialIndex: 0, scroller: this.scroller })`)
- it simply never calls it. `@State upPlayScrollerIndex` is declared for this
purpose and is never written or read.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the lyric is a
`rawfile` inside the HAP and needs no access at all.

Resource directories are `base`, `en_US` and `zh_CN`. The UI strings
(`build_main_top_text`, the toast text) are resources, so the shell is
localisable - the lyric itself is a fixed asset.

The sweep colour `0xFF1949` is a numeric literal in `MainPage.ets`, repeated
in three of the four gradient branches, and the background gradient
(`#17233c` / `#2b3755` / `#181e34`) is likewise inline. Both belong in
`resources/base/element/color.json` if the page is ever themed.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`
  with `strictMode.caseSensitiveCheck` on, so this constraint is accurate -
  unlike the three docs in `HW-13-0004`.
- **There is no audio.** No `AVPlayer`, no `SoundPool`, no audio file. The
  page animates a lyric against wall-clock time and the transport buttons are
  toasts. Everything about synchronisation has to be rebuilt against a real
  player.
- The parser handles *enhanced* LRC only - per-word timestamps. A standard
  one-timestamp-per-line LRC produces one `Word` per line and the sweep
  becomes a per-line fill.
- `[mm:ss.xx]` with exactly two digits per field is hard-coded. Three-digit
  milliseconds (`[00:01.010]`) and hour-length timestamps do not match, and
  unmatched text is silently dropped - no metadata tag (`[ti:]`, `[ar:]`,
  `[offset:]`) is read, including `offset`, which exists precisely to correct
  the drift this page suffers from.
- The list viewport is a hard-coded 416vp, and the `Row({ space: 270 })`
  spacings assume one screen width.
- `wordHighlights: Map<string, boolean>` is populated during parsing with keys
  of the form `${lineStartTime}_${text}_${time}` and never read afterwards.

## Pitfalls

- **`HW-13-0029`** (B/low, confirmed): the highlight timeline is anchored to
  the first word's `onAppear`, not to a song clock, so the `00:01.01` opening
  offset is ignored and three independent epochs (the `animateTo` chain, the
  `setInterval` progress bar, the `TextTimer`) drift apart as `onFinish` gaps
  accumulate. Fix: drive the highlight from the same clock that drives
  progress, and derive the sweep position from it.
- **`HW-13-0030`** (D/medium, probable): the document promises
  歌词滚动 (lyric scrolling), but nothing ever scrolls - the `Scroller` is
  bound to the `List` and never called, `upPlayScrollerIndex` is dead state,
  and the highlight walks off the bottom of the fixed 416vp viewport. Fix:
  `scrollToIndex` on line change, or drop the claim.
- **`HW-13-0031`** (D/low, confirmed): the document's `playLrc` snippet pastes
  `processEnhancedLrc`'s sort-and-assign block into the `onFinish` callback,
  where `lrcList` is out of scope. The pasted code does not compile and is not
  in the sample. Fix: use the shipped `playLrc`.
- **Quadratic parsing**: `extractAllTimestampsFromLyrics` re-scans and
  re-sorts every line of the song once per word parsed. Hoist the call out of
  the loop.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `UIContext.animateTo`, `getPromptAction`, `px2vp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `AnimateParam`, `FinishCallbackType`, `curve`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-gradient-color.md` - `linearGradient` and its colour stops
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-gradient-color
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-image-effect.md` - `blendMode` and `BlendApplyType.OFFSCREEN`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-image-effect
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `Scroller`, `scrollToIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimer`, `TextTimerController`, `onTimer`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress` and `ProgressType.Linear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `MEDIA-11` - the `AVPlayer` this page needs before the lyric can be synchronised to anything
