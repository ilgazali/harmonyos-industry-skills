---
id: KIDS-13
title: Vocabulary cards - a stacked carousel with AVPlayer pronunciation
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/13_vocabulary_learning_cards.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/vocabulary_learning_cards-0000002353707870
sample: huawei_industry_tree/08_children_education/downloads/VocabularyLearningCards.zip
kits: ["@kit.MediaKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["media.createAVPlayer", "media.AVPlayer", "media.AVFileDescriptor", fdSrc, prepare, play, pause, stop, reset, release, "avPlayer.on('stateChange')", "avPlayer.on('error')", "resourceManager.getRawFdSync", Stack, PanGesture, PanDirection, "UIContext.animateTo", offset, zIndex, blur, ForEach, "@Link", "@Prop", safeAreaPadding]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0097, HW-08-0098, HW-08-0099, HW-08-0100, HW-08-0101, HW-08-0102, HW-08-0103, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for two things that happen to share a sample:

- **Playing a bundled audio file** through `AVPlayer` from a `rawfile`. This is
  the only Media Kit sample in the industry, and the `getRawFdSync` →
  `AVFileDescriptor` → `fdSrc` path is the part to take.
- **A stacked-card carousel** where the neighbours stay visible behind the
  active card, offset and blurred, rather than sliding fully off-screen.

The carousel technique is worth understanding because it is entirely arithmetic
on one index - there is no `Swiper`, no per-card animation state, and no list.
Every card computes its own depth from `currentIndex` and styles itself.

**Seven findings.** The one to know before copying anything: every raw file
descriptor the sample opens is leaked, and the reference for that call states
the obligation to close it in the same paragraph as the signature.

## Feature checklist

- A stack of word cards, the active one large and sharp, the neighbours smaller
  and blurred.
- Each card shows a picture, the pinyin, the Chinese word and the English word.
- Two buttons play the Chinese and English pronunciations.
- Horizontal swipes rotate the stack, wrapping at both ends.

## Architecture

One `entry` module, two pages, one model, one player utility. 665 lines.

```
entry/src/main/ets
├── constants/CommonConstants.ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/CardsModel.ets           CardsModel + CARDS_DATA (3 animals)
├── pages
│   ├── MainPage.ets               holds currentIndex, applies the top inset
│   └── SwiperStackComponent.ets   the stack, the geometry, the tap handlers
└── utils/MediaPlayer.ets          singleton AVPlayer wrapper
```

The documented tree matches the zip.

**The carousel is two functions and no state.** Every card asks where it sits
relative to the active one, and styles itself from the answer:

```typescript
getImgCoefficients(index: number): number {
  const coefficient: number = this.currentIndex - index;
  const tempCoefficient: number = Math.abs(coefficient);
  if (tempCoefficient <= this.halfCount) {
    return coefficient;                      // near side: signed distance
  }
  const dataLength: number = this.swiperData.length;
  let tempOffset: number = dataLength - tempCoefficient;
  if (tempOffset <= this.halfCount) {
    return coefficient > 0 ? -tempOffset : tempOffset;   // wrapped: other way round
  }
  return 0;
}
```

**`halfCount` is what makes the stack circular.** A card more than half the deck
away is nearer going the other way round, so the coefficient flips sign - which
is why card 0 sits to the *right* of the last card rather than eight places
left. That single comparison replaces any explicit wrap-around bookkeeping.

The coefficient then drives four attributes at once:

| Attribute | Expression | Effect |
| --- | --- | --- |
| `offset.x` | `-127 * coefficient` when `|coefficient| === 1` | neighbours peek out |
| `zIndex` | `2 - Math.abs(coefficient)` | nearer cards on top |
| `blur` | `12` unless active | depth of field |
| sizes | ternaries on `index !== currentIndex` | the active card is larger |

**The player is a singleton with a state machine, not a play() call.** Setting
`fdSrc` starts a chain: `idle` → apply source, `initialized` → `prepare()`,
`prepared` → `play()`. The page never calls `play` at all - it assigns a source
and the handler does the rest.

## Implementation steps

1. **Open the rawfile descriptor**, build an `AVFileDescriptor`, assign it to
   `fdSrc` - **and close the fd afterwards** (`HW-08-0097`).
2. **Reset the player to idle before assigning a new source** (`HW-08-0099`).
3. **Register `stateChange` and `error` once**, when the player is created.
4. **Compute each card's depth coefficient** from `currentIndex` and the deck
   size.
5. **Style offset, zIndex, blur and size from that one number.**
6. **Track the drag in `onActionUpdate` and commit in `onActionEnd`**
   (`HW-08-0100`).
7. **Release the player and close its descriptors in `aboutToDisappear`.**

## Verified snippets

All snippets are from `VocabularyLearningCards.zip`. Corrected forms are marked.

**Playing a bundled sound — `entry/src/main/ets/pages/SwiperStackComponent.ets`** (corrected, see `HW-08-0097` and `HW-08-0099`)

```typescript
Image($r('app.media.chinese_pronunciation'))
  .onClick(async () => {
    const instance: MediaPlayer = await MediaPlayer.getInstance();
    const player = await instance.getAVPlayer(this.context);

    // FIX: the sample assigns fdSrc directly from whatever state the player is
    // in, and relies on the 'error' handler to reset it back to idle.
    await instance.reset();

    MediaPlayer.audioUrl = item.chineseAudioUrl;
    const fileDescriptor = this.context.resourceManager.getRawFdSync(MediaPlayer.audioUrl);
    const avFileDescriptor: media.AVFileDescriptor = {
      fd: fileDescriptor.fd,
      offset: fileDescriptor.offset,
      length: fileDescriptor.length
    };
    player.fdSrc = avFileDescriptor;
    // FIX: the sample never closes it. The reference for getRawFdSync says:
    // "To prevent resource leakage, call closeRawFdSync or closeRawFd to close
    // the fd after use." Close it when the source is replaced or on release.
  });
```

**`offset` and `length` are not optional.** A rawfile is a slice of the bundled
HAP, not a standalone file, so the descriptor is the archive's fd plus where
inside it the asset begins and how long it is. Passing the `fd` alone plays the
wrong bytes.

**`AVFileDescriptor` is the right source type for bundled audio.** `url` is for
network and sandbox paths; assets that ship inside the app come through
`getRawFdSync` and `fdSrc`.

**The state machine — `entry/src/main/ets/utils/MediaPlayer.ets`** (as shipped)

```typescript
avPlayer.on('stateChange', (state) => {
  MediaPlayer.state = state;
  switch (state) {
    case 'idle':
      MediaPlayer.isPlaying = false;
      if (MediaPlayer.audioUrl !== '') {            // re-apply after a reset
        let fileDescriptor = context.resourceManager.getRawFdSync(MediaPlayer.audioUrl);
        avPlayer.fdSrc = { fd: fileDescriptor.fd, offset: fileDescriptor.offset,
                           length: fileDescriptor.length };
      }
      break;
    case 'initialized':
      avPlayer.prepare();            // source accepted -> decode it
      break;
    case 'prepared':
      avPlayer.play();               // decoded -> start
      break;
    case 'playing':
      MediaPlayer.isPlaying = true;
      break;
    // ... paused, completed, stopped, released, error
  }
});
```

**Driving playback from `stateChange` rather than from a chain of `await`s is
the correct shape for `AVPlayer`.** The transitions are asynchronous and
partially driven by the framework, so the handler is the only place that knows
the player is genuinely ready; calling `prepare()` and `play()` in sequence from
the tap handler would race.

**Read this pairing carefully before copying it**, though: the `idle` branch
exists to re-apply the source after a reset, and in this sample the only thing
that causes a reset is the `error` callback (`HW-08-0099`).

**The card geometry — `entry/src/main/ets/pages/SwiperStackComponent.ets`** (as shipped)

```typescript
.offset({ x: this.getOffSetX(index), y: 0 })
.blur(index !== this.currentIndex ? 12 : 0)
.zIndex(index !== this.currentIndex && this.getImgCoefficients(index) === 0 ?
  0 : 2 - Math.abs(this.getImgCoefficients(index)))
.width(CommonConstants.LARGE_WIDTH)
.height(index !== this.currentIndex ? 224 : CommonConstants.FULL_HEIGHT)
```

**`zIndex` has a special case for coefficient 0 on a non-active card** - with
three cards, `getImgCoefficients` returns 0 for anything more than `halfCount`
away, so that guard pushes the far card behind everything instead of letting it
tie with the active one.

**Only immediate neighbours are offset:**

```typescript
getOffSetX(index: number): number {
  let offsetIndex: number = this.getImgCoefficients(index);
  if (Math.abs(offsetIndex) === 1) {
    return -127 * offsetIndex;      // one card either side peeks out
  }
  return 0;                         // everything else hides behind the centre
}
```

That is deliberate for a three-card deck and is the limit of the technique - a
longer deck would need a per-depth offset rather than a single `=== 1` test.

**Wrapping the index — same file** (as shipped)

```typescript
startAnimation(isLeft: boolean, duration: number): void {
  this.uiContext.animateTo({ duration: duration }, () => {
    let dataLength: number = this.swiperData.length;
    let tempIndex: number = isLeft ? this.currentIndex + 1
                                   : this.currentIndex - 1 + dataLength;
    this.currentIndex = tempIndex % dataLength;
  });
}
```

**`- 1 + dataLength` before the modulus** is what keeps the backwards case
positive: `(-1) % 3` is `-1` in JavaScript, not `2`. Adding the length first is
the standard fix and the sample gets it right.

**Changing `currentIndex` inside `animateTo` animates every card at once**,
because all four styling attributes are functions of it - one assignment,
one coordinated transition.

**Teardown — same file** (as shipped)

```typescript
async aboutToDisappear(): Promise<void> {
  let instance: MediaPlayer = await MediaPlayer.getInstance();
  MediaPlayer.audioUrl = '';        // stop the idle branch re-arming the source
  await instance.reset();
  AppStorage.setOrCreate('curAudioId', '');
}
```

**Clearing `audioUrl` before the reset is load-bearing**, not tidying: `reset()`
drives the player to `idle`, and the `idle` handler re-applies `audioUrl` if it
is non-empty - so resetting without clearing it first would start playback again
on the way out.

The page resets but never calls `release()`, so the player object and its
descriptors survive the page.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Audio files ship in
`resources/rawfile` and are read through the resource manager, so no storage or
media permission applies.

`MainPage` applies only the top inset:

```typescript
.safeAreaPadding({ top: this.uiContext.px2vp(this.topRectHeight) })
```

`bottomRectHeight` is bound with `@StorageProp` and never used.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Three cards.** `CARDS_DATA` holds panda, tiger and koala; the offset rule
  (`|coefficient| === 1`) and the `zIndex` formula (`2 - |coefficient|`) are
  tuned for a deck that small.
- **The pinyin carries hand-inserted padding spaces** (`'lǎo    hǔ'`) to align
  syllables under the characters - layout encoded into the data, which breaks at
  any other font size.
- **There is no progress, no pause control and no repeat** - a tap starts a
  sound and nothing can stop it short of switching cards.
- **The card does not follow the finger.** The swipe commits at gesture
  recognition and the animation runs independently (`HW-08-0100`).
- **The player is never released**, only reset, so its descriptors and its
  static state outlive the page.
- Only the top safe-area inset is applied; the bottom one is bound and ignored.

## Pitfalls

- **`HW-08-0097` — every `getRawFdSync` descriptor is leaked.** Four call sites,
  no `closeRawFdSync` anywhere, and a child taps the two pronunciation buttons
  on every card. The reference states the obligation in the same paragraph as
  the signature.
- **`HW-08-0098` — `initialize()` assigns the newly created player to its own
  parameter** instead of `this.avPlayer`, so that branch leaves the field null
  and every guarded method silently returns. The method is also never called;
  `getAVPlayer` does it correctly a hundred lines below.
- **`HW-08-0099` — playing a second word routes through the `error` handler.**
  No UI path resets the player, and the `idle` branch that re-applies the source
  is only reached because `on('error')` calls `reset()`.
- **`HW-08-0100` — the card advances from `onActionStart` and `onActionEnd` is
  empty,** so a movement that merely clears the recogniser's threshold flips the
  card and a deliberate swipe can be neither completed nor abandoned.
- **`HW-08-0101` — all mutable player state is `static`** beside a singleton, and
  `release()` nulls the instance while leaving eight statics stale - so a fresh
  instance can report itself as playing before it has a player.
- **`HW-08-0102` — `pinyin` and `englishName` are plain strings** beside a
  `ResourceStr` `chineseName` on the same object, in a bilingual app whose
  English half is therefore not localisable.
- **`HW-08-0103` — `getInstance()` is `async` with nothing to await,** and
  `hilog.error(0x0000, 'err' + err, '')` puts the message in the tag parameter -
  on the one log line reporting the player failure the sample depends on.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFdSync` and the requirement to call `closeRawFdSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/02_media/media-kit-intro.md` - `AVPlayer` and its state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/media-kit-intro
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `PanDirection`, the handler set
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-stack-layout.md` - the stacked layout
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-stack-layout
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-z-order.md` - `zIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-z-order
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-image-effect.md` - `blur`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-image-effect
- `KIDS-16` - the industry's other media sample, using a third-party GIF library instead of Media Kit
- `SPORT-13` - a card stack reordered by drag, with a sequential `GestureGroup`
