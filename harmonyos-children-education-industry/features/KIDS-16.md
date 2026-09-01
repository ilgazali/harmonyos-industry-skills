---
id: KIDS-16
title: Dice roller - a secure random face driving a one-shot GIF animation
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/16_die_rolling.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/die_rolling-0000002412074829
sample: huawei_industry_tree/08_children_education/downloads/DieRolling.zip
kits: ["@kit.CryptoArchitectureKit", "@kit.LocalizationKit", "@kit.ArkUI", "@ohos/gif-drawable"]
apis: ["cryptoFramework.createRandom", generateRandomSync, "GIFComponent.ControllerOptions", "GIFComponent.ScaleType", setScaleType, setSpeedFactor, setOpenHardware, setLoopFinish, loadBuffer, destroy, "ResourceLoader.loadMedia", LoadingProgress, "UIContext.getPromptAction", showToast, "@State"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0115, HW-08-0116, HW-08-0117, HW-08-0118, HW-08-0119, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for two unrelated things it happens to demonstrate together:

- **Playing an animated GIF from a bundled resource**, with a callback when the
  loop ends. This is the only sample in the industry that uses a third-party
  ohpm package, and `@ohos/gif-drawable` is the standard answer - ArkUI's
  `Image` will display a GIF but gives no control over playback or completion.
- **Producing a small uniform random integer** from a cryptographic byte source.
  The sample gets the source right and the reduction wrong, which makes it a
  useful negative example.

**Five findings.** The first is the one that matters: `byte % 6` is not a fair
die, and the document prescribes it.

## Feature checklist

- A die shown as a static image at rest.
- A roll button with a spinner while the animation plays.
- A secure random face chosen on each roll.
- The matching GIF plays once and stops on the rolled face.
- A toast when the animation finishes.

## Architecture

One `entry` module, one page, one logger. 296 lines - the smallest sample in
the industry.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages/Index.ets        the whole feature
└── utils/Logger.ets
```

The documented tree matches the zip.

**Six GIFs, one per face, selected by index:**

```typescript
private gifResource: Resource[] =
  [$r('app.media.1'), $r('app.media.2'), $r('app.media.3'),
   $r('app.media.4'), $r('app.media.5'), $r('app.media.6')];
```

Each animation tumbles and settles on its own face, so the "result" is the last
frame of the chosen clip. That is why nothing displays a number: the animation
*is* the result. It also means the die value is decided **before** the
animation, not by it.

**The playback cycle is controller-per-roll:**

```
click ──> destroy the old controller
      ──> new ControllerOptions, configure scale/speed/hardware
      ──> ResourceLoader.loadMedia(faceResource) ──> loadBuffer ──> autoPlay = true
      ──> setLoopFinish ──> autoPlay = false, hide spinner, toast
```

**A fresh `ControllerOptions` per roll rather than reloading one** is the right
call here - it keeps the loop-finish callback bound to exactly one playback and
avoids having to reset playback state.

## Implementation steps

1. **Install `@ohos/gif-drawable`** and declare it in the *module's*
   `oh-package.json5` (`HW-08-0119`).
2. **Preload the resting face in `aboutToAppear`** so the die is visible before
   the first roll.
3. **Draw a byte and reduce it without bias** (`HW-08-0115`).
4. **Destroy the previous controller, build a new one**, configure scale, speed
   and hardware decoding.
5. **Load the face's buffer, then set `autoPlay`** - in that order, so playback
   starts on a loaded controller.
6. **Register `setLoopFinish`** to stop after the intended number of loops
   (`HW-08-0116`).
7. **Destroy the live controller in `aboutToDisappear`** (`HW-08-0117`).

## Verified snippets

All snippets are from `DieRolling.zip`. Corrected forms are marked.

**Rolling — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-08-0115`)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

.onClick(() => {
  this.rollVisibility = Visibility.Visible;        // spinner on

  // FIX: the sample computes `randData.data[0] % 6`. A byte has 256 values and
  // 256 = 6 x 42 + 4, so faces 1-4 arise from 43 byte values and faces 5-6 from
  // 42 - a 2.4% bias, in a die whose whole purpose is fairness.
  const rand = cryptoFramework.createRandom();
  let b: number;
  do {
    b = rand.generateRandomSync(1).data[0];
  } while (b >= 252);                              // reject the tail: 252 = 6 x 42
  const num: number = b % 6;

  this.model.destroy();                            // release the previous animation
  const modelGIF = new GIFComponent.ControllerOptions();
  modelGIF.setScaleType(GIFComponent.ScaleType.FIT_CENTER)
    .setSpeedFactor(1)
    .setOpenHardware(true);

  ResourceLoader.loadMedia(this.gifResource[num].id, this.context).then((buffer) => {
    modelGIF.loadBuffer(buffer, (error) => {
      if (error) {
        Logger.info('loadMediaByWorker error =' + error);
      }
      this.gifAutoPlay = true;                     // start only once loaded
      this.model = modelGIF;
    });
  });

  modelGIF.setLoopFinish(() => { /* see below */ });
});
```

**Rejection sampling is the standard fix and it is cheap here** - fewer than 2%
of draws are discarded, and the loop terminates with probability 1. Using a
cryptographic source and then reducing with a plain modulus discards exactly
the property the source was chosen for.

**`setOpenHardware(true)` asks for hardware decoding**, which is why the
controller must be released (`HW-08-0117`) - hardware decoders are a small
per-device pool, not just memory.

**`autoPlay` is set inside the `loadBuffer` callback, not beside it.** Setting
it before the buffer arrives would start a controller with nothing to play; the
ordering here is correct and is the part to copy.

**Stopping after one loop — same file** (corrected, see `HW-08-0116`)

```typescript
// FIX: the sample tests `if (this.gifLoopCount > 0)` against a field that is
// initialised to 1 and never written, so the condition is a constant. Raising
// gifLoopCount to 3 changes nothing.
private gifLoopCount: number = 1;
private loopsDone: number = 0;

modelGIF.setLoopFinish(() => {
  this.loopsDone++;
  if (this.loopsDone >= this.gifLoopCount) {
    this.gifAutoPlay = false;
    this.rollVisibility = Visibility.None;         // spinner off
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.finish'),
      bottom: 150
    });
  }
});
```

**`setLoopFinish` is the reason to use this library rather than `Image`.**
ArkUI's `Image` plays an animated GIF but reports nothing, so there is no way to
know the die has settled - which is exactly when the spinner must be hidden and
the result announced.

**Preloading the resting face — same file** (as shipped)

```typescript
async aboutToAppear() {
  this.model.destroy();
  let modelX = new GIFComponent.ControllerOptions();
  modelX.setScaleType(GIFComponent.ScaleType.FIT_CENTER)
    .setSpeedFactor(1)
    .setOpenHardware(true);

  ResourceLoader.loadMedia($r('app.media.1').id, this.context).then((buffer) => {
    modelX.loadBuffer(buffer, (error) => {
      if (error) { Logger.info('loadMediaByWorker error =' + error); }
      this.model = modelX;
    });
  });
}
```

**`gifAutoPlay` stays `false` here**, so the first face loads and shows its
first frame without animating - the die at rest. The flag is the only thing
separating "loaded" from "playing".

**The spinner — same file** (as shipped)

```typescript
Button({ stateEffect: true }) {
  Row() {
    LoadingProgress()
      .width(24).height(24)
      .color(Color.White)
      .visibility(this.rollVisibility)     // Visible during the roll, None otherwise
      .margin({ right: 5 });
    Text($r('app.string.roll'))
      .fontColor(Color.White)
      .fontWeight(500)
      .fontSize(21);
  }
  .alignItems(VerticalAlign.Center);
}
```

**`Visibility.None` rather than `Hidden`** collapses the spinner's slot, so the
button's label re-centres when it is not rolling. `Hidden` would leave a gap.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The GIFs are bundled
resources read through `ResourceLoader`.

The one external dependency is declared only at project level:

```json
// oh-package.json5 (root)
"dependencies": { "@ohos/gif-drawable": "^2.1.1" }

// entry/oh-package.json5
"dependencies": {}
```

The module that imports it declares nothing, and the caret range is looser than
the exact `2.1.1` the document specifies (`HW-08-0119`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Requires `@ohos/gif-drawable`** from ohpm; the animation is the whole app.
- **One die.** No count selector, no sum, no history of rolls.
- **The result is only ever shown as an animation frame.** `num` is used to pick
  a clip and then discarded - nothing stores or displays the number, so the app
  cannot report a result or keep a total.
- **The face is decided before the animation**, so the clip is a replay of a
  decision already made rather than a simulation.
- **Six GIFs must exist as `app.media.1` … `app.media.6`**, indexed by string
  interpolation of the array position - a resource naming contract nothing
  validates.
- No `aboutToDisappear`, so the live controller and its hardware decoder outlive
  the page (`HW-08-0117`).

## Pitfalls

- **`HW-08-0115` — `randData.data[0] % 6` is a biased die.** 256 is not a
  multiple of 6, so faces 1-4 come up 43/256 of the time and faces 5-6 42/256.
  The document prescribes the same reduction, and the sample's choice of
  `cryptoFramework` over `Math.random` is undone by it.
- **`HW-08-0116` — `gifLoopCount` is initialised to 1 and never written,** so
  `if (this.gifLoopCount > 0)` is a constant. The field, its comment and the
  comment above the callback all describe a counter that does not exist;
  changing it has no effect.
- **`HW-08-0117` — the GIF controller is never destroyed on teardown.** The
  sample calls `destroy()` before every replacement, showing it knows the
  resource must be released, and omits the one place where no replacement comes -
  leaking a decoded buffer and a hardware decoder per visit.
- **`HW-08-0118` — `.width('100%').width('70%')` and `.fontSize(12) … .fontSize(21)`**
  each set one attribute twice; the first of each pair is dead and reads as a
  deliberate value.
- **`HW-08-0119` — `entry/oh-package.json5` declares no dependencies** although
  the module imports `@ohos/gif-drawable`, and the root pins `^2.1.1` where the
  document specifies `@2.1.1` exactly.

## References

- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - `createRandom`, `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `DataBlob.data` as a `Uint8Array`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-loadingprogress.md` - the spinner
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-loadingprogress
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-visibility.md` - `None` versus `Hidden`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-visibility
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - resource ids
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `KIDS-15`, `KIDS-10`, `KIDS-02` - the industry's three other `cryptoFramework` consumers, each with a different variant of the same byte-reduction mistake
- `KIDS-13` - the other media sample, using Media Kit's `AVPlayer` instead of a third-party library
