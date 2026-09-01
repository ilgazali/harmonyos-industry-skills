---
id: UTIL-13
title: Immersive keyboard - tell the input method to match your page with keyboardAppearance
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/13_input_method_application_immersive_mode.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/input_method_application_immersive_mode-0000002283469114
sample: huawei_industry_tree/15_utilities/downloads/InputMethodApplicationImmersiveMode.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [display, hilog, window, Search, SearchController, keyboardAppearance, KeyboardAppearance, "display.getAllDisplays", "UIContext.px2vp", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", avoidAreaChange, "@StorageProp", "ConfigurationConstant.ColorMode"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0101, HW-15-0105, HW-15-0106, HW-15-0107]
status: verified
---

## When to use

Load this card when your page **draws its own background behind the text box** -
a photo, a gradient, a full-screen weather scene - and the system keyboard
sliding up over it in default chrome breaks the picture. One attribute,
`keyboardAppearance(KeyboardAppearance.IMMERSIVE)`, tells the input method
application that the foreground app is running immersively, and the keyboard
picks a matching skin.

The pattern is not a rendering trick you perform yourself. It is a
**declaration passed down a chain**: your text box states a desired mode, the
input method framework forwards it, and the keyboard application decides the
final look. You cannot force a specific keyboard appearance, and you should not
try - `IMMERSIVE` means "follow the system", and the guide asks keyboards that
receive it to resolve it to `LIGHT_IMMERSIVE` or `DARK_IMMERSIVE` themselves.

It generalises to every text entry point over a custom backdrop: a search bar
on a hero image (this sample), a comment box on a video player, a chat
composer over a wallpaper. The same attribute exists on `TextInput`,
`TextArea` and `Search`, so adopting it is one line per field - but read the
constraint first: **it only takes effect after input method adaptation**, so on
a device whose keyboard has not implemented the protocol nothing changes and
nothing breaks.

## Feature checklist

- A weather-style home page with a full-bleed background image behind
  everything, including the status bar and navigation indicator.
- A search box in the top row, over the image, with a translucent pill behind
  it rather than an opaque field.
- Tapping the search box raises the keyboard in immersive style on devices
  whose input method supports it; on other devices the ordinary keyboard
  appears and the page is unchanged.
- The search box width is computed from the real display width minus the back
  button and the horizontal margins, so it fills the row exactly.
- Two translucent cards below: hot cities with weather and temperature, and an
  air quality ranking with the current city's index.
- Content is inset by the status bar height at the top and the navigation
  indicator height at the bottom, tracked live.

## Architecture

One `entry` module, one page, two tiny models and a constants file. There is no
network layer and no state beyond three numbers - this is a styling sample.

```
entry/src/main/ets
├── common/Constants.ets            numeric/percent literals + the two static data arrays
├── entryability/EntryAbility.ets   forces light color mode, full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model
│  ├── AirQualityModel.ets          cityName / provinceName / airQualityIndex
│  └── PopularCityModel.ets         cityName / weather / temperature
└── pages/MainPage.ets              the whole page, 384 lines, four @Builder blocks
```

The documented tree matches the zip exactly, including the file comments.

**The design decision worth copying** is how the translucent field is built.
The `Search` component itself is *not* made translucent. Instead each glassy
surface is a `Stack` of two children: an empty `Column` carrying the white
background, the corner radius and `.opacity(0.1)`, and the real content drawn
on top at full opacity:

```typescript
Stack() {
  Column()                       // the frosted plate: white, rounded, 10% opaque
    .width(this.searchWidth)
    .height($r('app.float.width_40'))
    .borderRadius($r('app.float.radius_24'))
    .backgroundColor(Color.White)
    .opacity(Constants.OPACITY_DEFAULT);

  Search({ ... })                // the real control, fully opaque
}
```

`opacity` on a container applies to its whole subtree, so putting it on the
`Search` would fade the placeholder, the icon and the typed text with the
background. Splitting the plate from the content is what keeps the text
readable at 100% while the plate sits at 10%. The same two-layer trick is
repeated for both cards below, which is why the page has three otherwise
pointless empty `Column`s.

The second structural choice is that `searchWidth` is **computed, not
declared**: the page reads the display width, converts px to vp through the
`UIContext`, and subtracts the two 16 vp margins and the 48 vp back button.
That is the right instinct, though it has a cost - see the constraints.

## Implementation steps

1. **Make the window full screen in the ability**, before anything else:
   `windowStage.getMainWindowSync()` then
   `setWindowLayoutFullScreen(true)`. Without this the background image stops
   at the status bar and the immersive keyboard has nothing to be immersive
   against.
2. **Publish the avoid areas into `AppStorage`** (`topRectHeight`,
   `bottomRectHeight`) from `getWindowAvoidArea` for `TYPE_SYSTEM` and
   `TYPE_NAVIGATION_INDICATOR`, and subscribe to `avoidAreaChange` to keep
   them current when the indicator hides or the device rotates.
3. **Read them back with `@StorageProp`** in the page, typed `number` with a
   `0` default, and convert at the point of use with
   `this.getUIContext().px2vp(...)`. The values in `AppStorage` are px; padding
   is vp. Mixing the two is the classic bug this sample avoids.
4. **Stack a translucent plate behind each control** rather than fading the
   control itself.
5. **Add `.keyboardAppearance(KeyboardAppearance.IMMERSIVE)` to the `Search`.**
   The default is `NONE_IMMERSIVE`, so this must be stated explicitly; it is
   the entire feature.
6. **Do not also set the keyboard's colour.** `IMMERSIVE` (value 1) means
   "follow the system" and is resolved by the input method application; an app
   cannot set `IMMERSIVE` downward from the keyboard side, and should not try
   to second-guess the light/dark decision.
7. **Give the `Search` a `SearchController`** if you intend to drive selection
   or caret position later; the sample creates one and holds it, which costs
   nothing and saves a rewrite.

## Verified snippets

All snippets are from `InputMethodApplicationImmersiveMode.zip`.

**The immersive search box — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Builder
backAndSearch(): void {
  Row() {
    Image($r('app.media.left_icon'))
      .width($r('app.float.width_40'))
      .aspectRatio(Constants.ASPECT_RATIO_1)
      .margin({ left: $r('app.float.margin_16'), right: $r('app.float.margin_8') });

    Stack() {
      Column()
        .width(this.searchWidth)
        .height($r('app.float.width_40'))
        .borderRadius($r('app.float.radius_24'))
        .backgroundColor(Color.White)
        .opacity(Constants.OPACITY_DEFAULT);

      Search({ placeholder: $r('app.string.search_city'), controller: this.controller })
        .width(this.searchWidth)
        .searchIcon({ color: $r('app.color.search_icon_color') })
        .placeholderFont({ size: $r('app.float.font_size_16'), weight: Constants.FONT_WEIGHT_DEFAULT })
        .placeholderColor($r('app.color.font_color_white'))
        .fontColor($r('app.color.font_color_white'))
        .textFont({ size: $r('app.float.font_size_16'), weight: Constants.FONT_WEIGHT_DEFAULT })
        .keyboardAppearance(KeyboardAppearance.IMMERSIVE); // 设置搜索框沉浸模式
    };
  }
  .width(Constants.FULL_PERCENT)
  .height($r('app.float.height_40'))
  .margin({ top: $r('app.float.margin_8'), bottom: $r('app.float.margin_8') });
}
```

**Four attributes carry the design, and only one of them is the feature.**
`keyboardAppearance(KeyboardAppearance.IMMERSIVE)` is the declaration passed to
the input method framework; everything else exists so that the box looks
immersive *before* the keyboard appears. `placeholderColor` and `fontColor` are
both forced to a white resource because the field sits on a photograph and
would otherwise inherit dark defaults; `searchIcon({ color })` is the only way
to recolour the built-in magnifier, which is not covered by `fontColor`.

The trailing comment 设置搜索框沉浸模式 ("set the search box to immersive
mode") is the sample's only annotation, and it is on the right line. Note the
attribute is available from API 15 and defaults to `NONE_IMMERSIVE`, so an
existing text box gains nothing until you add it.

**Deriving the search width — same file** (as shipped)

```typescript
@State screenWidth: number = 0;
@State searchWidth: number = 0;
@State uiContext: UIContext = this.getUIContext();
private controller: SearchController = new SearchController();

aboutToAppear(): void {
  display.getAllDisplays((err, data) => {
    this.screenWidth = data[0].width;
    // 计算搜索框的长度
    this.searchWidth =
      this.uiContext.px2vp(this.screenWidth) - Constants.DOUBLE_MARGIN_16 - Constants.BACK_BUTTON_AND_RIGHT_MARGIN;
  });

  for (let i = 0; i < Constants.GUIDE_DOTS_LENGTH; i++) {
    if (i === Constants.LIST_INDEX0) {
      this.guideDots.push($r('app.media.ellipse1'));
    } else {
      this.guideDots.push($r('app.media.ellipse2'));
    }
  }
}
```

`display.getAllDisplays` returns widths in **px**, and every layout dimension
in ArkUI is **vp**, so the `px2vp` conversion is mandatory - without it the box
would be roughly three times too wide on a typical phone. `DOUBLE_MARGIN_16` is
32 (the two 16 vp page margins) and `BACK_BUTTON_AND_RIGHT_MARGIN` is 48 (the
40 vp icon plus its 8 vp margin); naming those two derived numbers instead of
writing `- 80` is what makes the arithmetic auditable.

The callback form is the weak point: `err` is never inspected and
`data[0].width` is read unconditionally, so a failed call throws inside the
callback rather than falling back. `searchWidth` also stays `0` for the frames
between the first paint and the callback, which is visible as a brief collapse
of the search row. An `await`ed `display.getDefaultDisplaySync()` read, or a
sensible non-zero default, removes both problems.

**Full screen and avoid areas — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
windowStage.loadContent('pages/MainPage', (err) => {
  if (err.code) {
    hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
    return;
  }

  let windowClass: window.Window = windowStage.getMainWindowSync();   // 获取应用主窗口
  // 1. 设置窗口全屏
  windowClass.setWindowLayoutFullScreen(true).then(() => {
    hilog.info(DOMAIN, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag',
      'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
  });

  // 2. 获取布局避让遮挡的区域
  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;                            // 状态栏
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  // 3. 注册监听函数，动态获取避让区域数据
  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });
});
```

**This is the boilerplate that makes the background image full-bleed**, and it
is a prerequisite for the feature rather than part of it: an immersive keyboard
under a page that still respects the system insets looks wrong from both ends.
Two things are worth copying - reading the avoid areas *once* at startup so the
first frame is already correct, and then subscribing to `avoidAreaChange` so a
rotation or a gesture-bar change updates them. Reading only once leaves a stale
inset; subscribing only leaves the first frame wrong.

`windowClass.on('avoidAreaChange', ...)` is never paired with an `off` in
`onWindowStageDestroy`. That is the same unreleased-subscription boilerplate
seen across this industry's samples; it is harmless for a single-ability demo
and wrong in a real app.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`module.json5` lists `deviceTypes: ["phone", "tablet", "2in1"]` with a single
`EntryAbility` and an `EntryBackupAbility`. Nothing about the immersive
keyboard needs a manifest entry - it is purely a component attribute.

`EntryAbility.onCreate` pins the application to light mode:

```typescript
this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
```

That matters for this feature specifically. `KeyboardAppearance.IMMERSIVE`
means "follow the system", and a well-behaved keyboard resolves it to
`LIGHT_IMMERSIVE` or `DARK_IMMERSIVE` from the *system* colour mode - which the
app has just decoupled from its own. On a device in dark mode you can therefore
get a dark keyboard under a light-pinned page. If you copy the colour-mode pin,
consider `LIGHT_IMMERSIVE` / `DARK_IMMERSIVE` explicitly instead of `IMMERSIVE`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `keyboardAppearance` itself is
  available from API 15.
- **The attribute only takes effect after input method adaptation.** The
  reference is explicit: "This setting takes effect only after input method
  adaptation." A keyboard that has not implemented the protocol shows its
  normal appearance; there is no fallback to write and no error to handle.
- An application can express `IMMERSIVE`, `LIGHT_IMMERSIVE`,
  `NONE_IMMERSIVE` and so on, but it cannot dictate the final rendering - the
  input method application makes that call.
- The sample is a static mock: `POPULAR_CITIES` and `AIR_QUALITY` are three
  hardcoded entries each in `Constants.ets`, the search box has no `onSubmit`,
  the back arrow and the air-quality chevron have no `onClick`, and the five
  guide dots are decorative `Image`s with no swiper behind them.
- Both `List`s are laid out inside fixed-height translucent cards
  (`height_178`, `height_223`) sized for exactly three rows. Adding a fourth
  city silently clips.
- `searchWidth` derives from `display.getAllDisplays()[0].width`, the physical
  display, not the window. In a floating or split window on a 2in1 the search
  box will be sized for the whole screen.

## Pitfalls

**No defects were recorded against this document or sample during review** -
the frontmatter `findings` list is empty and the status is `verified`. The
points below are adoption caveats, not review findings:

- **The immersive declaration and the pinned colour mode disagree.** The page
  forces `COLOR_MODE_LIGHT` while asking the keyboard to follow the system.
  Prefer an explicit `LIGHT_IMMERSIVE` when the page's own theme is fixed.
- **`display.getAllDisplays` errors are ignored** and `searchWidth` is `0`
  until the callback lands, so the search row collapses for the first frames.
- **`avoidAreaChange` is never unsubscribed** in `onWindowStageDestroy`.
- **The white plate is drawn with `opacity(0.1)` rather than a blur.** On a
  busy photograph a 10% white wash is thin; `backgroundBlurStyle` would give a
  more readable frosted surface at the same cost.
- **Copying this page wholesale brings three empty `Column`s** whose only job
  is to be a translucent backdrop. That is deliberate here, but if your surface
  does not need a different opacity from its content, one container with a
  `rgba()` background colour is simpler.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - `keyboardAppearance`, `searchIcon`, `SearchController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `documentation/harmonyos-references/02_application-framework/ts-text-common.md` - the `KeyboardAppearance` enum and its five values
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-text-common
- `documentation/harmonyos-guides/03_application-framework/inputmethod-immersive-mode-guide.md` - the app -> framework -> keyboard chain, and what the keyboard side must do
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/inputmethod-immersive-mode-guide
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `avoidAreaChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `huawei_industry_tree/15_utilities/docs/13_input_method_application_immersive_mode.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/input_method_application_immersive_mode-0000002283469114
- `UTIL-15` - the other text-entry card in this industry, where the same `TextInput` family is used for a form
