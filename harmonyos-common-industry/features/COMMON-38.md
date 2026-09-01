---
id: COMMON-38
title: Slide-puzzle human verification - a Slider whose content area is replaced by a jigsaw image
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/38_slide_verification.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/slide_verification-0000002348282886
sample: huawei_industry_tree/19_common_technical_solutions/downloads/SlideVerification.zip
kits: ["@kit.ArkUI"]
apis: [Slider, SliderStyle, SliderChangeMode, SliderInteraction, "Slider.onChange", "Slider.contentModifier", "Slider.sliderInteractionMode", "Slider.blockStyle", SliderBlockType, "Slider.blockSize", "Slider.trackThickness", SliderConfiguration, "SliderConfiguration.triggerChange", ContentModifier, wrapBuilder, WrappedBuilder, CustomDialog, CustomDialogController, TextTimer, TextTimerConfiguration, "TextTimer.contentModifier", "@Link", "PromptAction.showToast"]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-19-0114, HW-19-0115, HW-19-0116, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when a login or registration screen needs a **drag-the-piece-into-
the-gap challenge** before it will proceed, and you want it built from a native
`Slider` rather than from a custom gesture handler or a web view.

The reusable technique is `Slider.contentModifier`: it replaces the slider's
entire content area with your own builder, so the "slider" can render as a
background photo, a cut-out gap, a moving puzzle piece and a drag track - while
the framework still handles the drag, the value range and the change events.

**Read HW-19-0114 first.** As shipped the puzzle has the same answer every time,
which removes the property that makes a slide puzzle a challenge at all.

## Feature checklist

The application must:

- Present a background image with a gap and a matching puzzle piece.
- Move the piece with the slider value, so the drag position and the piece
  position stay in step.
- Accept the attempt when the piece lands within a tolerance of the gap.
- **Randomise the gap position per challenge** (HW-19-0114).
- Offer a refresh control that resets the slider and re-randomises.
- Restrict the interaction to dragging - no tap-to-jump.
- Report the result and close the dialog.
- Follow the challenge with a resend countdown rendered through the same
  content-modifier technique.
- State that the verdict is client-side only (HW-19-0115).

## Architecture

Single-module project (`entry` HAP), and the only sample in this industry built
against **API 24 / DevEco 6.1.1**:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup |
| `constants/CommonConstant.ets` | `INITIAL_POSITION = -280`, `INTERVAL_MIN = 360`, `INTERVAL_MAX = 380` |
| `view/VerificationDialog.ets` | the `@CustomDialog`, the `buildSlider` content builder and `MySliderStyle` |
| `view/BuildTextTimer.ets` | `MyTextTimerModifier` and `buildTextTimer` - the resend countdown |
| `pages/LoginPage.ets` | the login form, the `CustomDialogController` and the `isPeople` state |

**How the puzzle is drawn.** The dialog's `Slider` has an ordinary
`value`/`min`/`max` (0-560) and a `contentModifier(new MySliderStyle(...))`. The
builder receives a `SliderConfiguration` and lays out four images in a `Stack`:

| Image | Role | Position |
| --- | --- | --- |
| `backgroundImage` | the photo | fills a 326x120 clipped `Row` |
| `select1` | the gap | fixed `margin.left = 90` |
| `select2` | the moving piece | `margin.left = CommonConstant.INITIAL_POSITION + config.value` |
| `refresh` | reset control | top-right; sets `config.value = 0` and calls `triggerChange` |

So the piece's x position is `-280 + value`, and it aligns with the gap when
`value` is around 370 - which is exactly the `[360, 380]` acceptance band. The
three numbers are one design, and changing any of them requires changing the
others.

**The second slider is the drag track.** Below the picture the builder places a
*real* `Slider` in `SliderStyle.InSet` with an image block, a 40 vp track and
`sliderInteractionMode(SliderInteraction.SLIDE_ONLY)`. Its `onChange` calls
`config.triggerChange(value, mode)`, which feeds the value back into the outer
slider - and that is what re-renders the puzzle piece and eventually fires the
outer `onChange`.

**The verdict** is taken on `SliderChangeMode.End` - drag released:

```ts
if (mode === SliderChangeMode.End) {
  if (value >= CommonConstant.INTERVAL_MIN && value <= CommonConstant.INTERVAL_MAX) {
    this.isPeople = true;
  }
  this.controller?.close();
  this.uiContext.getPromptAction().showToast({ message: this.isPeople ? ... : ..., ... });
}
```

The dialog closes on release either way; the toast reports which happened.
`isPeople` is `@Link`-bound to the login page's `@State`, so the page sees the
result directly.

**The countdown** reuses the pattern of COMMON-37: a `ContentModifier<TextTimerConfiguration>`
whose builder computes `config.count / 1000 - config.elapsedTime / 100` - count in
milliseconds, elapsedTime in hundredths - and shows a "resend" link when
`config.started` is false.

## Implementation steps

1. **Model the geometry as one set of related numbers**: the gap's x, the piece's
   base offset, and the tolerance band. Derive the band from the gap rather than
   hardcoding both (HW-19-0114).
2. **Build the dialog** as a `@CustomDialog` with `@Link isPeople: boolean` so the
   host page sees the outcome.
3. **Declare the outer `Slider`** with the drag range and attach
   `contentModifier(new MySliderStyle(...))`.
4. **Write the content builder**: a `Stack` of background, gap and piece, with the
   piece positioned from `config.value`; plus the visible drag `Slider` whose
   `onChange` calls `config.triggerChange(value, mode)`.
5. **Restrict the interaction** with `sliderInteractionMode(SliderInteraction.SLIDE_ONLY)`
   so the user cannot tap the track to jump the piece into place.
6. **Add a refresh control** that sets `config.value = 0` and calls
   `config.triggerChange(config.value, SliderChangeMode.Click)` - the official
   `SliderConfiguration` example uses exactly this assign-then-trigger pair - and
   re-randomises the gap.
7. **Decide on `SliderChangeMode.End`**, close the dialog and toast the result.
8. **Follow with the countdown** through a `TextTimer` content modifier.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The dialog and the verdict

`SlideVerification.zip#SlideVerification/entry/src/main/ets/view/VerificationDialog.ets`

```ts
@CustomDialog
export struct VerificationDialog {
  uiContext = this.getUIContext();
  @Link isPeople: boolean;
  sliderValue: number = 0;
  sliderMin: number = 0;
  sliderMax: number = 560;
  sliderChangeMode: number = 0;
  verificationPrompt: string | Resource = $r('app.string.dialog_text1');
  controller?: CustomDialogController;

  build() {
    Column() {
      Row() {
        Text($r('app.string.dialog_text1'))
          .fontSize(16)
          .fontWeight(500)
          .layoutWeight(1)
          .height(50);
        Image($r('app.media.close'))
          .width(24)
          .onClick(() => {
            this.isPeople = false;
            this.controller?.close();
          });
      }
      .width(326);

      Slider({
        value: this.sliderValue,
        min: this.sliderMin,
        max: this.sliderMax,
      })
        .onChange((value: number, mode: SliderChangeMode) => {
          if (mode === SliderChangeMode.End) {
            if (value >= CommonConstant.INTERVAL_MIN && value <= CommonConstant.INTERVAL_MAX) {
              this.isPeople = true;
            }
            this.controller?.close();
            this.uiContext.getPromptAction().showToast({
              message: this.isPeople ? $r('app.string.Prompt') : $r('app.string.dialog_text3'),
              duration: 2000,
              bottom: 130
            });
          }
        })
        .contentModifier(new MySliderStyle(SliderChangeMode.End));
    }.height('100%');
  }
}
```

### The puzzle content builder

`SlideVerification.zip#SlideVerification/entry/src/main/ets/view/VerificationDialog.ets`

```ts
// Customize slider content
@Builder
function buildSlider(config: SliderConfiguration) {
  Column() {
    Stack() {
      Row() {
        Image($r('app.media.backgroundImage'))
          .width(336)
          .clip(true)
          .padding({ top: -40 });
      }.clip(true)
      .borderRadius(4)
      .width(326).height(120);

      Image($r('app.media.select1'))          // the gap - fixed position, see HW-19-0114
        .height(42)
        .margin({ left: 90, top: 32 });

      Image($r('app.media.select2'))          // the moving piece
        .height(62)
        .margin({ left: CommonConstant.INITIAL_POSITION + config.value, top: 32 });

      Image($r('app.media.refresh'))
        .height(16)
        .margin({ left: 296, bottom: 80 })
        .onClick(() => {
          config.value = 0;
          config.triggerChange(config.value, SliderChangeMode.Click);
        });
    };

    Stack() {
      Text($r('app.string.dialog_text2'))
        .fontWeight(400)
        .fontSize(14);
      Slider({
        value: config.value,
        min: config.min,
        max: config.max,
        style: SliderStyle.InSet
      })
        .sliderInteractionMode(SliderInteraction.SLIDE_ONLY)
        .blockStyle({ type: SliderBlockType.IMAGE, image: $r('app.media.slider') })
        .blockSize({ width: 45, height: 45 })
        .blockColor('#191970')
        .trackColor('#0d000000')
        .trackBorderRadius(2)
        .trackThickness(40)
        .selectedColor('#4169E1')
        .onChange((value: number, mode: SliderChangeMode) => {
          config.triggerChange(value, mode);
        });
    }
    .margin({ top: 3 })
    .height(50)
    .width(340);
  };
}

class MySliderStyle implements ContentModifier<SliderConfiguration> {
  sliderChangeMode: number = -1;

  constructor(sliderChangeMode: number) {
    this.sliderChangeMode = sliderChangeMode;
  }

  applyContent(): WrappedBuilder<[SliderConfiguration]> {
    return wrapBuilder(buildSlider);
  }
}
```

### The geometry constants (as shipped - see HW-19-0114)

`SlideVerification.zip#SlideVerification/entry/src/main/ets/constants/CommonConstant.ets`

```ts
export class CommonConstant {
  static readonly INITIAL_POSITION: number = -280;
  static readonly INTERVAL_MIN: number = 360;
  static readonly INTERVAL_MAX: number = 380;
}
```

### The countdown content builder

`SlideVerification.zip#SlideVerification/entry/src/main/ets/view/BuildTextTimer.ets`

```ts
export class MyTextTimerModifier implements ContentModifier<TextTimerConfiguration> {
  applyContent(): WrappedBuilder<[TextTimerConfiguration]> {
    return wrapBuilder(buildTextTimer);
  }
}

// Customize countdown content
@Builder
function buildTextTimer(config: TextTimerConfiguration) {
  Column() {
    Stack({ alignContent: Alignment.Center }) {
      Column() { // 后台倒计时到负数时给组件任然设置显示0
        Text(config.started ? (Math.max(config.count / 1000 - config.elapsedTime / 100, 0)).toFixed(0) + 's' :
          $r('app.string.retrieve'))
          .fontColor(config.started ? Color.Black : '#0A59F7')
          .fontWeight(400)
          .fontSize(14);
      };
    };
  };
}
```

The `Math.max(..., 0)` is deliberate - the comment says it keeps the display at 0
when the countdown has gone negative in the background.

### Hosting the dialog

`SlideVerification.zip#SlideVerification/entry/src/main/ets/pages/LoginPage.ets`

```ts
@State isPeople: boolean = false;

dialogController: CustomDialogController | null = new CustomDialogController({
  builder: VerificationDialog({
    isPeople: this.isPeople,
  }),
  // ...
});

// the form reacts to the result
TextInput({ controller: this.controller, placeholder: this.isPeople ? $r('app.string.LoginPage_text5') : '' })
// ...
if (this.isPeople) { /* ... */ }
```

## Permissions & config

**No permissions are required** and none are declared - the whole feature is UI,
and there is no network call anywhere in the project (which is the substance of
HW-19-0115).

`SlideVerification.zip#SlideVerification/entry/build-profile.json5` pins both
`targetSdkVersion` and `compatibleSdkVersion` to `6.1.1(24)` - **higher than every
other sample in this industry**, which build against `6.0.0(20)`. The document's
section reflects that, though under a non-standard heading (HW-19-0116).

## Constraints

- **API level 24 / DevEco Studio 6.1.1** - the document says
  "本示例基于API Version 24 Release及以上版本进行开发与验证" and
  "本示例需要使用DevEco Studio 6.1.1 Release及以上版本进行编译运行", and the build
  profile confirms `6.1.1(24)`. This is the only solution in the industry with
  that floor.
- **`Slider.contentModifier` replaces the content area entirely.** The framework
  still owns the value, the range and the events; everything visible is yours.
- **`triggerChange` is how a custom content area reports a change back.** The
  reference notes that "The **onChange** event of the original slider can only be
  triggered when the **triggerChange** API is called with valid parameter
  values" - so the inner slider's `onChange` forwarding is not optional plumbing,
  it is the whole data path.
- **`SliderConfiguration.value` is writable**, and the official example assigns it
  before calling `triggerChange` - which is what the refresh button does.
- **`SliderInteraction.SLIDE_ONLY`** prevents tap-to-position; without it the user
  could tap the track and land on the answer without dragging.
- **The verdict fires on `SliderChangeMode.End` only.** The refresh button passes
  `SliderChangeMode.Click`, which is why resetting does not count as an attempt.
- **`isPeople` is never set back to `false` on a failed drag** - only the close
  button clears it - so a previously successful verification survives a later
  failed one.
- **Devices.** Per `module.json5`; the feature itself is device-agnostic.

## Pitfalls

- **The gap and the accepted band are constants, which is incorrect for a
  challenge.** `select1` sits at `left: 90` and the band is always `[360, 380]`,
  so the answer is the same on every attempt and on every device. Randomise the
  gap and derive the band from it. (HW-19-0114)
- **The document does not say the verdict is client-side only.** Challenge,
  answer and result are all local, and `isPeople` gates the login flow directly.
  (HW-19-0115)
- **The version section is named 环境准备, not 约束与限制**, and omits the
  HarmonyOS SDK bullet every sibling carries - on the one document in the
  industry whose floor is *not* API 20. (HW-19-0116)
- **The three geometry constants are coupled and undocumented as such.**
  `INITIAL_POSITION + value` must equal the gap's `left` when `value` is in the
  band; edit `left: 90` alone and the puzzle accepts a visibly wrong position.
- **The dialog closes on every release, success or failure.** A user who
  overshoots must reopen the dialog rather than retry in place - the refresh
  button only helps before the release.
- **`MySliderStyle.sliderChangeMode` is stored and never used.** The constructor
  takes `SliderChangeMode.End` and the field is dead; the mode check happens in
  the dialog's own `onChange`.
- **`verificationPrompt` in the dialog is assigned and never rendered** - the
  header uses `$r('app.string.dialog_text1')` directly.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` -
  `Slider`, `SliderStyle`, `SliderChangeMode`, `sliderInteractionMode` /
  `SliderInteraction`, `blockStyle` / `SliderBlockType`, `trackThickness`,
  `contentModifier`, `SliderConfiguration` (including the writable `value` and
  `triggerChange`) and the note that `onChange` fires only when `triggerChange` is
  called with valid values.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` -
  `TextTimerConfiguration.count` (milliseconds), `elapsedTime` (minimum unit of
  the format) and `started`, used by the countdown builder.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-content-modifier.md` -
  the `ContentModifier<T>` interface and `applyContent`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-content-modifier
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-methods-custom-dialog-box -
  `@CustomDialog` and `CustomDialogController`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/slide_verification-0000002348282886
