---
id: EDU-17
title: Word spelling drill - shake the input on a wrong answer, play the pronunciation as a hint
industry: 04_education
doc: huawei_industry_tree/04_education/docs/17_word_spelling.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
sample: huawei_industry_tree/04_education/downloads/WordSpelling.zip
kits: ["@kit.ArkUI", "@kit.MediaKit", "@kit.LocalizationKit", "@kit.ArkTS", "@kit.AbilityKit"]
apis: [TextInput, onWillChange, onChange, defaultFocus, copyOption, keyframeAnimateTo, KeyframeState, translate, Swiper, SwiperController, changeIndex, disableSwipe, onAnimationStart, "@Watch", "@Link", "@Prop", "@StorageLink", "window.getLastWindow", "on('keyboardHeightChange')", "media.createAVPlayer", fdSrc, "resourceManager.getRawFdSync", "resourceManager.closeRawFdSync", animateTo, PromptAction]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0125, HW-04-0126, HW-04-0127, HW-04-0128, HW-04-0129, HW-04-0130, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **type-the-answer drill**: the learner sees a prompt, types
a word, submits, and gets immediate feedback - a shake on a wrong answer, the
correct spelling revealed, an optional pronunciation hint.

Three techniques are worth taking:

- **`keyframeAnimateTo` for the shake.** Three keyframes and `iterations: 3`
  gives a whole error animation with no state machine and no timers.
- **`onWillChange` as an input lock.** Returning `false` from it rejects the
  edit *before* it lands, so a solved word cannot be retyped - cleaner than
  `enabled(false)`, which also greys the text out.
- **`keyboardHeightChange` to resize the page.** The card and the button row stay
  visible above the soft keyboard instead of being pushed off-screen.

What to be careful with: this sample couples advancing to the next word to the
pronunciation audio completing, and the audio's failure path never completes
(`HW-04-0125`).

## Feature checklist

- One word per screen with a running `n/N` counter.
- The prompt is the Chinese translation; typing the English word and pressing
  check grades it.
- A wrong answer turns the text red and shakes the input three times; the correct
  spelling appears below.
- Editing after a wrong answer resets the grade to unrated.
- A correct answer turns the text green, locks the input, plays the
  pronunciation and shows the phonetic transcription.
- The bulb button plays the pronunciation and shows the transcription as a hint.
- The page shrinks to sit above the soft keyboard.
- Finishing the last word toasts and returns to the home page.

## Architecture

One `entry` module, two pages, four components.

```
entry/src/main/ets
├── components
│   ├── WordSpellingCard.ets    THE CARD: TextInput + shake + reveal
│   ├── ImageButton.ets         the round icon buttons
│   ├── StudyStatusCard.ets     home-page widgets
│   └── WordPlanCard.ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── media/AVPlayerManager.ets   pronunciation playback from a rawfile
├── model/WordData.ets          WordDetail + GradingStatus
├── pages
│   ├── HomePage.ets            @Entry
│   └── WordSpellingPage.ets    the drill: state, grading, navigation, keyboard
└── utils
    ├── FileReadUtil.ets        the word list from a rawfile JSON
    └── LogUtil.ets
```

The documented tree matches the zip.

**`GradingStatus` is the whole state machine** - `UNRATED`, `CORRECT`,
`INCORRECT` - and it drives five things at once: the input colour, whether the
input accepts edits, whether the translation or the answer is shown below,
whether the check button or the next button is displayed, and (through `@Watch`)
the shake animation.

**The page owns the state; the card renders and mutates it.**
`WordSpellingCard` takes `status` and `currentInput` as `@Link`, so typing and
the shake live in the card while grading and navigation live in the page. The
card never decides whether an answer is right.

**The `Swiper` is a controlled carousel, not a swipeable one.**
`.disableSwipe(true)` with `.index($$this.currentWordIndex)` and
`swiperController.changeIndex(i, true)` means the page advances words with an
animation the learner cannot bypass, and `onAnimationStart` resets the grade and
the input for the incoming word - one place, both fields.

**The coupling that goes wrong:** on a correct answer the page sets
`showTips = true`, and the *only* thing that sets it back is the audio-completed
callback. Both the automatic advance and the manual next button are gated on
`!showTips` (`HW-04-0125`).

## Implementation steps

1. **Model the grade as a three-state enum** and derive every visual from it
   rather than tracking separate booleans.
2. **Lock a solved input with `onWillChange`**, not with `enabled`:
   ```typescript
   .onWillChange(() => this.status !== GradingStatus.CORRECT)
   ```
   Returning `false` rejects the edit before it is applied.
3. **Reset the grade in `onChange`** so that editing after a wrong answer clears
   the red state immediately.
4. **Shake with `keyframeAnimateTo`** driven by a `@Watch` on the grade - see the
   snippet below.
5. **Play the pronunciation as a hint, not as a gate.** Clear the tips flag on a
   timer regardless of the playback outcome, and do not guard the manual next
   button on it (`HW-04-0125`).
6. **Subscribe to `keyboardHeightChange`, and release it** in
   `aboutToDisappear` (`HW-04-0126`).
7. **Keep every timer id** and clear it on teardown - one of them pops the
   navigation stack (`HW-04-0127`).
8. **Release the rawfile descriptor** with `closeRawFdSync(path)` when playback
   ends; this sample does, `EDU-11`'s copy of the same class does not
   (`HW-04-0128`).

## Verified snippets

All snippets are from `WordSpelling.zip`. Corrected forms are marked.

**The shake — `entry/src/main/ets/components/WordSpellingCard.ets`** (as shipped)

```typescript
const DURATION_40_MS = 40;
const DURATION_80_MS = 80;
const X_OFFSET_VP = 8;

@State xOffset: number = 0;
@Link @Watch('onStatusChange') status: GradingStatus;

onStatusChange() {
  if (this.status === GradingStatus.INCORRECT) {
    this.getUIContext().keyframeAnimateTo({ iterations: 3 }, [
      { duration: DURATION_40_MS, event: () => { this.xOffset = X_OFFSET_VP; } },
      { duration: DURATION_80_MS, event: () => { this.xOffset = -X_OFFSET_VP; } },
      { duration: DURATION_40_MS, event: () => { this.xOffset = 0; } }
    ]);
  }
}

TextInput()
  .translate({ x: this.xOffset })
```

**Three keyframes and `iterations` is the entire animation.** The 40/80/40 ms
split is what makes it read as a shake rather than a slide: out fast, across
slowly through centre to the other side, back fast. `keyframeAnimateTo` runs the
list in order and repeats it, so one declaration produces six direction changes.

**`@Watch` on the `@Link` is the trigger.** The card does not know it got the
answer wrong - the page sets `status`, the watcher fires in the card, and the
animation runs where the animated property lives. Driving it from the page would
mean lifting `xOffset` out of the component that uses it.

**Lock and reset — same file** (as shipped)

```typescript
TextInput()
  .copyOption(CopyOptions.None)                  // no copy/paste of the answer
  .defaultFocus(true)                            // see HW-04-0130
  .fontColor(this.status === GradingStatus.CORRECT ? $r('app.color.correct_text')
           : this.status === GradingStatus.INCORRECT ? $r('app.color.incorrect_text')
           : $r('app.color.normal_text'))
  .onChange((value) => {
    this.status = GradingStatus.UNRATED;         // any edit clears the red state
    this.currentInput = value;
  })
  .onWillChange(() => {
    return this.status !== GradingStatus.CORRECT;  // reject edits once solved
  })
```

**`onWillChange` fires before the change is applied and can veto it.** That is
the difference from `onChange`, which reports a change already made. Using it to
lock a solved word keeps the text black-on-green and the caret available, where
`.enabled(false)` would grey the whole field out.

`copyOption(CopyOptions.None)` matters for a spelling drill specifically - it
removes the paste route around the exercise.

**Grading and the tips coupling — `entry/src/main/ets/pages/WordSpellingPage.ets`** (corrected, see `HW-04-0125`, `HW-04-0127`)

```typescript
@State @Watch('onTipChange') showTips: boolean = false;
private tipsTimer?: number;
private exitTimer?: number;

aboutToAppear(): void {
  this.wordList = FileReadUtil.readWordDataFromRawfile(this.resourceManager);
  this.avPlayerManager.setAVPlayerCompletedCallback(() => { this.scheduleHideTips(); });
  // ... keyboard subscription
}

// FIX: the sample hides the tips ONLY from the audio-completed callback, so an
//      audio failure leaves showTips true forever and blocks both advance paths.
private scheduleHideTips() {
  clearTimeout(this.tipsTimer);
  this.tipsTimer = setTimeout(() => { this.showTips = false; }, CLOSE_TIPS_DELAY_MS);
}

onTipChange() {
  if (!this.showTips && this.status === GradingStatus.CORRECT) {
    this.goToNextWord();
  }
}

// the check button
onButtonClick: () => {
  if (this.currentInput.toLowerCase() === this.wordList[this.currentWordIndex].word.toLowerCase()) {
    this.status = GradingStatus.CORRECT;
    this.avPlayerManager.prepareExampleAudio(this.resourceManager!, this.wordList[this.currentWordIndex].audio);
    this.showTips = true;
    this.scheduleHideTips();          // FIX: guarantee the flag clears even if audio fails
  } else {
    this.status = GradingStatus.INCORRECT;
  }
}

// the next button - FIX: the sample wraps this in `if (!this.showTips)`,
// which removes the user's only escape when the audio never completes
onButtonClick: () => { this.goToNextWord(); }

goToNextWord() {
  if (this.currentWordIndex < this.wordList.length - 1) {
    this.swiperController.changeIndex(this.currentWordIndex + 1, true);
  } else {
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.toast_finished'), duration: TOAST_DURATION_MS });
    this.exitTimer = setTimeout(() => { this.pathInfos.pop(); }, TOAST_DURATION_MS);
  }
}

aboutToDisappear(): void {
  clearTimeout(this.tipsTimer);        // FIX: neither timer is cleared in the sample
  clearTimeout(this.exitTimer);
  this.window?.off('keyboardHeightChange');   // FIX: absent in the sample
  this.avPlayerManager.releasePlayer(this.resourceManager!);
}
```

**Comparing lower-cased is the right grading rule** for a spelling drill -
capitalisation is not what is being tested - and it belongs in the page, not the
card, so the card stays a pure input.

**An optional hint must never gate the core loop.** The audio is a courtesy;
`showTips` was made to depend on it, and then the manual next button was made to
depend on `showTips`. Either coupling alone is survivable; together they remove
every route forward when playback fails.

**The exit timer is the dangerous one.** It calls `pathInfos.pop()` 1.5 s after
the last word. Press back during the toast and the page pops, the timer survives,
and it pops a second entry.

**Resizing for the keyboard — same file** (corrected, see `HW-04-0126`)

```typescript
private window?: window.Window;

// FIX: the sample drops both the window handle and the promise rejection
window.getLastWindow(this.getUIContext().getHostContext())
  .then(currentWindow => {
    this.window = currentWindow;
    currentWindow.on('keyboardHeightChange', data => {
      this.getUIContext().animateTo({ curve: Curve.Ease, duration: KEYBOARD_ANIMATION_MS }, () => {
        this.keyboardHeight = this.getUIContext().px2vp(data);   // the event reports px
      });
    });
  })
  .catch((err: BusinessError) => {
    LOGGER.error(`getLastWindow failed: ${err.code}`);
  });

// and the layout consumes it
Column() { /* counter, Swiper, buttons */ }
  .size({
    height: this.keyboardHeight === 0 ? '100%' : `calc(100% - ${this.keyboardHeight}vp)`,
    width: '100%'
  })
  .justifyContent(FlexAlign.SpaceBetween)
```

**`calc(100% - Nvp)` plus `SpaceBetween` is what keeps the buttons reachable.**
Shrinking the container rather than scrolling it means the counter stays at the
top and the button row is pushed to the new bottom edge, directly above the
keyboard. Wrapping the height change in `animateTo` with the same duration the
keyboard uses makes the two move together.

**The event reports pixels**, so `px2vp` is required before it goes into a `calc`
expressed in vp.

**Pronunciation playback — `entry/src/main/ets/media/AVPlayerManager.ets`** (as shipped — the good copy, see `HW-04-0128`)

```typescript
private filePath: string = '';

async prepareExampleAudio(resourceManager: resourceManager.ResourceManager, filePath: string): Promise<void> {
  await this.stopAudio(resourceManager);
  try {
    this.avPlayer = await media.createAVPlayer();
    this.setAVPlayerCallback(resourceManager);
    const file = resourceManager.getRawFdSync(filePath);
    this.filePath = filePath;                    // remembered so it can be released
    this.avPlayer.fdSrc = { fd: file.fd, offset: file.offset, length: file.length };
  } catch (error) {
    LOGGER.error(`Failed to create AVPlayer. Error code: ${error.code}`);
  }
}

async releasePlayer(resourceManager: resourceManager.ResourceManager) {
  if (this.avPlayer) {
    await this.avPlayer.release();
    this.avPlayer = undefined;
    if (this.filePath) {
      resourceManager.closeRawFdSync(this.filePath);   // the rawfile counterpart of fileIo.close
      this.filePath = '';
    }
  }
}
```

**A rawfile descriptor is released by *path*, not by fd.**
`closeRawFdSync(filePath)` is the counterpart of `getRawFdSync(filePath)` -
which is why the path has to be remembered. `EDU-11` ships the same class with
this missing, and its cleanup is written entirely around a numeric `fileFd` that
the rawfile branch never sets.

`{ fd, offset, length }` is mandatory for a rawfile: the descriptor points at the
HAP, and the triple says which region of it is this audio clip.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block - the
word list and the audio clips are bundled rawfiles, and playback needs no
permission.

`module.json5` declares `"routerMap": "$profile:router_map"`, with
`wordSpellingPageBuilder` exported from `WordSpellingPage.ets`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The word list and every pronunciation clip are bundled rawfiles read once in
  `aboutToAppear`; there is no download, no user word list and no persistence of
  which words were answered.
- **All cards are built at page entry** - `ForEach`, not `LazyForEach` - so a
  long word list means a long list of `TextInput`s (`HW-04-0129`).
- Grading is exact-match after lower-casing: no partial credit, no accent or
  hyphen tolerance, no per-letter feedback.
- Swiping is disabled; the only way between words is the check and next buttons.
- The close button raises a 演示 toast - the exit-the-drill flow is not
  implemented.
- The keyboard-driven resize assumes the page is the full window; it does not
  account for split-screen or a floating window.

## Pitfalls

- **`HW-04-0125` — a failed pronunciation blocks the drill.** `showTips` is
  cleared only by the audio-completed callback, and both the automatic advance
  and the manual next button are gated on `!showTips`. Clear the flag on a timer
  and ungate the button.
- **`HW-04-0126` — `keyboardHeightChange` is never released,** and
  `getLastWindow` has no `catch`. The handler outlives the page and a second
  visit adds another.
- **`HW-04-0127` — two `setTimeout` handles are discarded;** the 1.5 s one calls
  `pathInfos.pop()` and can pop an extra page if the user leaves during the
  toast.
- **`HW-04-0128` — the industry ships two divergent copies of
  `AVPlayerManager`.** This one releases the rawfile descriptor; `EDU-11`'s does
  not. Neither clears `avPlayer` after the release in `stopAudio`.
- **`HW-04-0129` — the `Swiper`'s `ForEach` has no key generator** and builds
  every card up front; `EDU-07` shows the `LazyForEach` + `cachedCount` form for
  the same screen shape.
- **`HW-04-0130` — every card declares `defaultFocus(true)`,** so the whole word
  list competes for the initial focus.
- **Do not use `.enabled(false)` to lock a solved input.** `onWillChange`
  returning `false` rejects the edit without greying the field.

## References

- `documentation/harmonyos-references/02_application-framework/ts-keyframeanimateto.md` - `keyframeAnimateTo`, `KeyframeState`, `iterations`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-keyframeanimateto
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `onWillChange` versus `onChange`, `copyOption`, `defaultFocus`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `SwiperController.changeIndex`, `disableSwipe`, `onAnimationStart`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getLastWindow` and `on('keyboardHeightChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `fdSrc` and the player state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFdSync` / `closeRawFdSync` and the `{fd, offset, length}` triple
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `LazyForEach` with `cachedCount` in a `Swiper`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `EDU-11` - the same `AVPlayerManager`, without the descriptor release
- `EDU-07` - the same one-item-per-screen drill built on `LazyForEach`
