---
id: NEWS-05
title: In-app font size - a stepped Slider persisted in preferences, with system font scaling switched off
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
sample: huawei_industry_tree/11_news_reading/downloads/SetAppFontSize.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, preferences, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0033, HW-11-0034, HW-11-0035, HW-11-0036]
status: verified
---

## When to use

Load this card when the app has **long-form text the user reads for minutes at
a time** - a news article, a book chapter, a terms page - and you want the
reader's own size control rather than the system one. The pattern is three
parts that have to agree: a five-step `Slider` whose values *are* the font
sizes in fp, a preferences key holding the chosen size, and an app-level
`configuration` profile that detaches the app from the system font scale.

The third part is the one most implementations forget, and it is what makes
the other two honest. If the app still follows the system scale, the reader
sets 18 in your slider, the system multiplies by 1.75, and the setting no
longer means what the preview showed. Opting out with
`"fontSizeScale": "nonFollowSystem"` makes the in-app number the only
authority.

It generalises to any per-user display preference that must survive a restart
and be visible immediately: line height, theme, reading width. The shape is
always the same - one number in `AppStorage`-adjacent state, one preferences
key, one live preview that renders with the same state the real content uses.

## Feature checklist

- The article page renders title, timestamp and body at sizes derived from one
  number.
- A 更多 (more) sheet from the title bar opens a 字体设置 (text setting) page.
- That page shows a live preview using the same three text roles as the
  article, above a slider.
- The slider has exactly five stops (14, 16, 18, 20, 22 fp), drawn with tick
  marks and labelled 小 / 标准 / 大 / 特大 / 超大.
- The label matching the current size is drawn larger and bolder than the rest.
- Dragging the slider changes the preview on the same frame and writes the new
  size to preferences on every step.
- Popping back to the article carries the new size with it; killing and
  relaunching the app restores it.
- Changing the *system* font size in Settings has no effect on this app.

## Architecture

One `entry` module, one `@Entry` page plus one routed `NavDestination`, and a
preferences wrapper.

```
entry/src/main/ets
├── constants/CommonConstants.ets        the five sizes and the slider min/max/step
├── constants/StyleConstants.ets         '100%' / '95%' literals kept out of layout
├── database/PreferencesUtil.ets         create / seed / read / write the font-size key
├── entryability/EntryAbility.ets        creates the preferences store, seeds the default, avoid areas -> AppStorage
├── model/DataModel.ets                  TEXTLIST (5 labels) + SCALESLIST (5 tick marks)
├── pages/NewsRead.ets                   @Entry, @Provide changeFontSize, the NavPathStack
├── pages/SetFontSizePage.ets            NavDestination: back arrow + SliderLayout
├── utils/GlobalContext.ets              a Map singleton holding the Preferences instance
├── utils/Logger.ets
├── view/MoreFunctionCustomDialog.ets    the bottom sheet that pushes SetFontSizePage
├── view/News.ets                        the article body, sized from the prop
├── view/SliderLayout.ets                preview + tick marks + labels + Slider
└── view/TitleBar.ets                    title + the more button that opens the sheet
```

The documented tree matches the zip exactly, file for file.

**The design decision worth copying** is that the font size is a single
`@Provide` on the root page and every consumer reaches it by a *different*
binding chosen to match what it needs:

```typescript
@Provide changeFontSize: number = CommonConstants.SET_SIZE_NORMAL;   // NewsRead
@Prop  changeFontSize: number = 0;                                   // News (read-only body)
@Link  changeFontSize: number;                                       // SliderLayout (writes it)
@Consume changeFontSize: number;                                     // SetFontSizePage
```

`News` renders the article and never changes the size, so it takes `@Prop` - a
one-way copy. `SliderLayout` is the only component that writes, so it takes
`@Link`. `SetFontSizePage` sits between them in the navigation stack, not the
component tree, so it takes `@Consume`. Picking the weakest binding that still
works is what keeps a feature with four consumers from becoming four sources
of truth.

The one structural cost worth knowing about: `SetFontSizePage` is reached
through a `NavPathStack` and a `route_map.json` entry, not through `pages`, so
the size must also travel back through `pop(this.changeFontSize)` and be
parsed out of `popInfo.result` in the sheet's `onPop`. That is belt-and-braces
on top of `@Provide`/`@Consume`, and it is where the `parseInt` on line 67 of
`MoreFunctionCustomDialog.ets` comes from.

## Implementation steps

1. **Switch the app out of system font scaling** in
   `AppScope/resources/base/profile/configuration.json`, and point
   `app.json5` at it with `"configuration": "$profile:configuration"`.
   Without the second line the profile is inert.
2. **Define the sizes once** in `CommonConstants` and make the slider's
   `min`/`max`/`step` the same numbers, so the slider value *is* a font size
   and no mapping table is needed.
3. **Create the preferences store in `onCreate`**, before any page can read it,
   and seed the default with a `hasSync` guard so a relaunch does not
   overwrite the user's choice.
4. **Hold the store in a singleton**, not in each component - `GlobalContext`
   is a `Map<string, Object>` behind a private constructor, and
   `getFontPreferences()` casts the entry back to `preferences.Preferences`.
5. **Declare `changeFontSize` as `@Provide` on the `@Entry` page** and give
   each consumer the weakest binding it can use (`@Prop`, `@Link`,
   `@Consume`).
6. **Re-read the preference in `onPageShow`**, not `aboutToAppear`, so
   returning from the settings page refreshes the article even if the
   component was never destroyed.
7. **Write on every `onChange` step.** With a five-stop stepped slider that is
   at most five writes per drag, so there is nothing to debounce.
8. **Draw the tick marks yourself** with `Line` primitives at fixed
   coordinates and let `showSteps(true)` supply the slider's own step dots.
9. **Route the settings page through `route_map.json`** with a `@Builder`
   entry function, and pop the chosen size back as the `NavPathStack` result.

## Verified snippets

All snippets are from `SetAppFontSize.zip`. Nothing here is corrected - the
review found no defects in this sample.

**Opting out of system font scaling — `AppScope/resources/base/profile/configuration.json`** (as shipped)

```json
{
  "configuration": {
    "fontSizeScale": "nonFollowSystem",
    "fontSizeMaxScale": "3.2"
  }
}
```

referenced from `AppScope/app.json5`:

```json5
{
  "app": {
    "bundleName": "com.example.setappfontsize",
    "icon": "$media:app_icon",
    "label": "$string:app_name",
    "configuration": "$profile:configuration"
  }
}
```

**This four-line file is the whole "屏蔽系统字体大小设置" half of the feature.**
`fontSizeScale` takes `followSystem` or `nonFollowSystem`, and the platform
default is already `nonFollowSystem` - so the value here is a deliberate
restatement rather than a change, which is worth knowing when you copy it into
an app whose profile currently says `followSystem`.

`fontSizeMaxScale` is present but inert: the configuration reference states
that if `fontSizeScale` is `nonFollowSystem`, this attribute does not take
effect. It only matters in the opposite design, where you *do* follow the
system but want to cap the multiplier so a 3.2x setting does not shred your
layout. Leaving it at `3.2` next to `nonFollowSystem` is harmless but says
nothing.

**Slider stops that are literally font sizes — `entry/src/main/ets/constants/CommonConstants.ets`** (as shipped)

```typescript
export default class CommonConstants {
  static readonly SET_SIZE_SMALL: number = 14;         // 小号
  static readonly SET_SIZE_NORMAL: number = 16;        // 正常（默认）
  static readonly SET_SIZE_LARGE: number = 18;         // 大号
  static readonly SET_SIZE_EXTRA_LARGE: number = 20;   // 加大号
  static readonly SET_SIZE_HUGE: number = 22;          // 超大号
  static readonly SET_SLIDER_MIN: number = 14;         // 滑块最小值
  static readonly SET_SLIDER_MAX: number = 22;         // 滑块最大值
  static readonly SET_SLIDER_STEP: number = 2;         // 步长
}
```

**Min, max and step are chosen so that the slider's own value domain is the
font-size domain.** 14 to 22 in steps of 2 gives exactly five stops, one per
named size, so `onChange` can assign `value` straight into `changeFontSize`
with no lookup and no rounding. The named constants are then only used for
*labelling* - `TEXTLIST` pairs each with its display name, and the label
comparison `this.changeFontSize === item.size` is an exact equality that is
safe precisely because the slider can only land on those five integers.

If you widen this to a continuous slider, that equality check silently stops
matching and the emphasised label disappears. Keep the slider stepped, or
replace the comparison with a range test.

**The slider and its persistence — `entry/src/main/ets/view/SliderLayout.ets`** (as shipped)

```typescript
Slider({
  value: this.changeFontSize === CommonConstants.SET_SIZE_HUGE
    ? CommonConstants.SET_SLIDER_MAX : this.changeFontSize,
  min: CommonConstants.SET_SLIDER_MIN,   // 滑动最小值
  max: CommonConstants.SET_SLIDER_MAX,   // 滑动最大值
  step: CommonConstants.SET_SLIDER_STEP, // 步长
  style: SliderStyle.OutSet
})
  .stepColor(Color.Yellow)
  .trackColor('#22000000')
  .selectedColor('#22000000')
  .showSteps(true)
  .trackThickness(1)
  .stepSize(15)
  .blockStyle({ type: SliderBlockType.IMAGE, image: $r('app.media.block') })
  .blockSize({ width: 35, height: 35 })
  .onChange((value: number) => {
    if (this.changeFontSize === 0) {
      this.changeFontSize = PreferencesUtil.getChangeFontSize();
      return;
    }
    this.changeFontSize = value;
    PreferencesUtil.saveChangeFontSize(this.changeFontSize);
    Logger.info(TAG, 'Get the value of changeFontSize: ' + this.changeFontSize);
  })
```

**Three attributes carry the design.** `showSteps(true)` with
`trackThickness(1)` and matching `trackColor`/`selectedColor` turns the slider
into a bare rule with dots - there is no coloured "filled" portion, because on
a five-position size picker the amount to the left of the thumb means nothing.
`blockStyle({ type: SliderBlockType.IMAGE, ... })` replaces the default handle
with the app's own 35x35 asset, which is what makes it read as a font-picker
grip rather than a volume control.

**The `=== 0` guard on entry is a re-seed, not a validation.**
`PreferencesUtil.getChangeFontSize()` uses `0` as its `getSync` default, so `0`
is the sentinel for "the store has never been written". If the state is still
that sentinel when the first `onChange` fires, the handler restores the stored
value and returns *without* applying the drag - one dropped gesture in
exchange for never writing a `0` into the preference and never rendering
`fontSize(0)` in the article.

Note also that the `value:` expression maps `SET_SIZE_HUGE` (22) onto
`SET_SLIDER_MAX` (22). They are the same number in this sample, so the ternary
is a no-op today; it exists so that raising `SET_SIZE_HUGE` past the slider
maximum would clamp the thumb rather than push it off the track.

**Seeding without clobbering — `entry/src/main/ets/database/PreferencesUtil.ets`** (as shipped)

```typescript
const PREFERENCES_NAME = 'myPreferences';
const KEY_APP_FONT_SIZE = 'appFontSize';

createFontPreferences(context: Context) {
  let options: preferences.Options = { name: PREFERENCES_NAME };
  let fontPreferences: preferences.Preferences = preferences.getPreferencesSync(context, options);
  GlobalContext.getContext().setObject('getFontPreferences', fontPreferences);
}

saveDefaultFontSize(fontSize: number) {
  try {
    let fontPreferences: preferences.Preferences = this.getFontPreferences();
    // 检查首选项实例中是否存在字体size属性关键字，没有则创建
    let isExist: boolean = fontPreferences.hasSync(KEY_APP_FONT_SIZE);
    if (!isExist) {
      fontPreferences.putSync(KEY_APP_FONT_SIZE, fontSize);
      fontPreferences.flushSync();
    }
  } catch (err) {
    Logger.error(TAG, 'Get the preferences failed, err: ' + err);
  }
}

getChangeFontSize(): number {
  let defaultFontSize: number = 0;
  let fontPreferences: preferences.Preferences = this.getFontPreferences();
  let fontSize: number = fontPreferences.getSync(KEY_APP_FONT_SIZE, defaultFontSize) as number;
  return fontSize;
}
```

called from the ability before any UI exists:

```typescript
onCreate(want: Want): void {
  GlobalContext.getContext().setObject('abilityWant', want);
  PreferencesUtil.createFontPreferences(this.context);
  PreferencesUtil.saveDefaultFontSize(CommonConstants.SET_SIZE_NORMAL);   // 设置字体默认大小
}
```

**`hasSync` before `putSync` is the difference between a default and an
overwrite.** `saveDefaultFontSize` runs on every cold start; without the guard
it would reset the user's 22 back to 16 each launch. The guard makes the write
first-run-only, which is the correct semantics for "default".

`putSync` + `flushSync` is the synchronous pair: `putSync` updates the
in-memory cache and `flushSync` persists it. Both are cheap here because the
value is a single number written at most five times per drag; for larger or
hotter data the async `put`/`flush` are the right call. Note that the whole
module exports `new PreferencesUtil()` as its default, so every importer
shares one instance - the store lives in `GlobalContext`, not in the class, so
even that is incidental.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`module.json5` sets `"orientation": "portrait"` and `"deviceTypes": ["phone"]`
- narrower than the sibling samples in this industry, and consistent with a
layout that pins the tick marks at absolute coordinates.

The routed settings page is declared in
`entry/src/main/resources/base/profile/route_map.json`:

```json
{
  "routerMap": [
    {
      "name": "SetFontSizePage",
      "pageSourceFile": "src/main/ets/pages/SetFontSizePage.ets",
      "buildFunction": "SetFontSizePageBuilder"
    }
  ]
}
```

`main_pages.json` lists only `pages/NewsRead`. `SetFontSizePage` is reached
solely through `pageInfo.pushPath({ name: 'SetFontSizePage', ... })`, which is
why it needs the exported `@Builder SetFontSizePageBuilder()` wrapper.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The tick marks are absolute coordinates.** `SCALESLIST` in `DataModel.ets`
  holds five `Line` positions with hardcoded `horizontal` values from -56 to
  256 inside a 355 vp column, and `TEXTLIST` carries a per-label `width` (55,
  94, 70, 80, 75) tuned to make the five labels line up with them. Change the
  container width, the font, or the label strings and the ruler drifts out of
  register with the slider. This is the part of the sample not to copy
  verbatim.
- Sizes are applied as **offsets from one number**: title is
  `changeFontSize + 2`, timestamp is `changeFontSize - 4`, body is
  `changeFontSize`. At the smallest step the timestamp is therefore 10 fp.
  Anything below the 14 fp minimum would need its own floor.
- The document's step-2 snippet ends `onChange` with an extra
  `this.changeFontSize = PreferencesUtil.getChangeFontSize()` that the shipped
  `SliderLayout.ets` does not have. It is a harmless read-back of the value
  just written, but it means doc and zip are not line-identical here; the zip
  is the version to copy.
- The 点赞 (like) and 夜间模式 (night mode) entries in the more sheet are
  toasts reading 仅作展示 (display only). Only 字体设置 is wired up.
- `News.ets` carries a stray `@Entry` decorator although it is only ever used
  as a child component of `NewsRead`. Harmless, but it makes the file look
  like a routable page when it is not.

## Pitfalls

No defects were found in this document or sample during review. The frontmatter
`findings` list is empty and the status is `verified`.

Two things reviewed and cleared, worth naming so they are not re-investigated:

- The `if (this.changeFontSize === 0) { ...; return; }` early return in
  `onChange` looks like a swallowed event, but it is the deliberate re-seed
  described above and cannot leave the state at `0`.
- `saveDefaultFontSize` being called on every `onCreate` looks like it would
  reset the user's choice; the `hasSync` guard prevents it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderOptions`, `showSteps`, `blockStyle`, `SliderBlockType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `hasSync`, `putSync`, `getSync`, `flushSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/01_getting-started/app-configuration-file.md` - the `configuration` tag, `fontSizeScale` and why `fontSizeMaxScale` is inert under `nonFollowSystem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-configuration-file
- `huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
