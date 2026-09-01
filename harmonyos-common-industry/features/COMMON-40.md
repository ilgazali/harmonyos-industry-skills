---
id: COMMON-40
title: In-app language switch - override the system language with an application preferred language
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/40_language_switch.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/language_switch-0000002381506029
sample: huawei_industry_tree/19_common_technical_solutions/downloads/LanguageSwitch.zip
kits: ["@kit.LocalizationKit", "@kit.ArkUI"]
apis: ["i18n.System.setAppPreferredLanguage", "i18n.System.getAppPreferredLanguage", Navigation, NavPathStack, NavDestination, "NavDestination.onReady", NavDestinationContext, routerMap, "component .safeAreaPadding", "@StorageProp", "UIContext.px2vp"]
permissions: []
min_api: 17
modules: [entry]
findings: [HW-19-0120, HW-19-0121, HW-19-0122, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must let the user **pick its language
independently of the device language** - a common requirement for apps used by
people whose phone is set to a language they do not read well, or for products
that ship in fewer languages than the system offers.

The whole feature is two API calls plus correctly qualified resource
directories. There is no manual string table, no reload logic and no restart
handling to write - the resource system does it.

Note that this document has a **lower** version floor than the rest of the
industry: API 17 / HarmonyOS 5.0.5, matching the sample's
`compatibleSdkVersion: "5.0.5(17)"`.

## Feature checklist

The application must:

- Provide `base/element/string.json` plus one qualified directory per supported
  language (`zh_CN`, `en_US`), with the same keys in each.
- Reference every user-visible string through `$r('app.string.*')` so the
  resource system resolves it.
- Offer a language list and call `setAppPreferredLanguage` with the chosen
  language tag - inside `try/catch` (HW-19-0120).
- Read `getAppPreferredLanguage` when the page appears so the current choice is
  ticked.
- Offer a way back to the system language (HW-19-0121).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `components/MessageTab.ets` | the message tab content |
| `pages/MessagePage.ets`, `pages/MinePage.ets` | the two main tabs |
| `pages/SettingPage.ets` | settings, from which the language page is pushed |
| `pages/ChoicePage.ets` | the language list - the entire feature |
| `resources/base`, `resources/zh_CN`, `resources/en_US`, `resources/dark` | the qualified resource sets |

**Where the work happens: nowhere in the UI.** `ChoicePage` sets a preferred
language and nothing else. Every screen already renders its text through
`$r('app.string.*')`, so the resource manager picks the right qualified
directory. That is why the sample has no code to re-render, re-read or
re-navigate after the switch.

**Two language tags, exactly.** `'zh-Hans'` and `'en-Latn-US'` - written in the
script-and-region form rather than the shorter `'zh'` / `'en'` the reference
example uses. The same two strings appear again in `onReady` to decide which row
gets the tick:

```ts
.onReady((context: NavDestinationContext) => {
  let appPreferredLanguage: string = i18n.System.getAppPreferredLanguage();
  if (appPreferredLanguage === 'en-Latn-US') {
    this.isEnglish = true;
  } else if (appPreferredLanguage === 'zh-Hans') {
    this.isEnglish = false;
  }
  this.pathStack = context.pathStack
})
```

Note the two-branch form with no else: if the returned tag matches neither -
which is what happens before any preference has been set, when the API returns
the system language - `isEnglish` keeps its `false` default and the tick lands on
简体中文 regardless of what the device is actually showing.

**When the change takes effect.** The reference is explicit: "Resources are
loaded in the preferred language when the application is launched", and for the
`default` value "the setting will take effect upon cold starting of the
application". Screens already built may not re-resolve their strings until they
are rebuilt.

## Implementation steps

1. **Create the qualified resource directories.** `base/element/string.json` with
   the default values, then `zh_CN/element/string.json` and
   `en_US/element/string.json` with the **same keys**. The document's step 1 says
   exactly this, and it is the half that does the actual translating.
2. **Use `$r('app.string.*')` everywhere** a user-visible string appears - the
   switch has no effect on string literals in code (the trap COMMON-29 falls
   into, HW-19-0090).
3. **Build the language list** as rows with a tick bound to the current
   selection.
4. **Call `setAppPreferredLanguage(tag)` on tap**, inside `try/catch`, and update
   the selection state only after it returns (HW-19-0120).
5. **Read the current preference on entry** with `getAppPreferredLanguage()` and
   handle the "neither" case - before any preference is set the call reports the
   system language.
6. **Add a follow-the-system row** calling `setAppPreferredLanguage('default')`
   (HW-19-0121).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The two language rows

`LanguageSwitch.zip#LanguageSwitch/entry/src/main/ets/pages/ChoicePage.ets`

```ts
import { i18n } from '@kit.LocalizationKit';

Row() {
  Text($r('app.string.zh_CN'))
    .fontColor('#191919')
    .fontWeight(500)
    .fontSize(16)
    .lineHeight(21)
  Blank()
  if (!this.isEnglish) {
    Image($r('app.media.icon_ok'))
      .width(20)
  }
}
.width('100%')
.justifyContent(FlexAlign.SpaceBetween)
.alignItems(VerticalAlign.Center)
.height(48)
.onClick(() => {
  this.isEnglish = false;
  i18n.System.setAppPreferredLanguage('zh-Hans');    // FIX (HW-19-0120): wrap in try/catch
})

Divider()
  .width('100%')
  .color('#CCCCCC')

Row() {
  Text($r('app.string.en_US'))
    .fontColor('#191919')
    .fontWeight(500)
    .fontSize(16)
    .lineHeight(21)
  Blank()
  if (this.isEnglish) {
    Image($r('app.media.icon_ok'))
      .width(20)
  }
}
.width('100%')
.justifyContent(FlexAlign.SpaceBetween)
.alignItems(VerticalAlign.Center)
.height(48)
.onClick(()=>{
  this.isEnglish = true;
  i18n.System.setAppPreferredLanguage('en-Latn-US');  // FIX (HW-19-0120)
})
```

### Reading the current preference

`LanguageSwitch.zip#LanguageSwitch/entry/src/main/ets/pages/ChoicePage.ets`

```ts
.hideBackButton(true)
.onReady((context: NavDestinationContext) => {
  let appPreferredLanguage: string = i18n.System.getAppPreferredLanguage();
  if (appPreferredLanguage === 'en-Latn-US') {
    this.isEnglish = true;
  } else if (appPreferredLanguage === 'zh-Hans') {
    this.isEnglish = false;
  }
  this.pathStack = context.pathStack
})
```

### The page shell

`LanguageSwitch.zip#LanguageSwitch/entry/src/main/ets/pages/ChoicePage.ets`

```ts
@Builder
export function ChoicePageBuilder() {
  ChoicePage()
}

@Component
struct ChoicePage {
  @StorageProp('topRectHeight')
  topRectHeight: number = 0;
  @StorageProp('bottomRectHeight')
  bottomRectHeight: number = 0;
  @State isEnglish: boolean = false;
  pathStack?: NavPathStack;

  build() {
    NavDestination() {
      Column() {
        Row() {
          Image($r('app.media.back'))
            .width(40)
            .height(40)
            .margin({ right: 8 })
            .onClick(() => {
              this.pathStack?.pop();
            })
          Text($r('app.string.Settings'))
            .fontColor('#191919')
            .fontWeight(700)
            .fontSize(20)
            .lineHeight(27)
        }
        .height(56)
        .width('100%')
        // ... the two language rows ...
      }
      .safeAreaPadding({
        top: this.getUIContext().px2vp(this.topRectHeight),
        bottom: this.getUIContext().px2vp(this.bottomRectHeight)
      })
      .backgroundColor('#F1F3F5')
      .height('100%')
      .width('100%')
    }
  }
}
```

### The qualified resource sets

`LanguageSwitch.zip#LanguageSwitch/entry/src/main/resources/`

```
resources/
├── base/element/string.json     // default values
├── zh_CN/element/string.json    // Simplified Chinese
├── en_US/element/string.json    // English
└── dark/                        // dark-mode colours
```

Every visible label in the sample - the settings title, the two language names,
the tab titles - resolves through `$r('app.string.*')` against these.

## Permissions & config

**No permissions are required** and none are declared - `i18n.System` needs none.

`LanguageSwitch.zip#LanguageSwitch/entry/build-profile.json5` pins
`targetSdkVersion` and `compatibleSdkVersion` to `5.0.5(17)`, the lowest floor of
any sample in this industry, matching the document's 约束与限制 section
(API Version 17 Release, HarmonyOS 5.0.5 Release SDK, DevEco Studio 5.0.5
Release).

The `module.json5` declares the usual `EntryAbility` with the home skill, an
`EntryBackupAbility`, and a `routerMap` for the settings and language
destinations.

## Constraints

- **API level.** API Version 17 Release or later, HarmonyOS 5.0.5 Release SDK or
  later, DevEco Studio 5.0.5 Release or later. `setAppPreferredLanguage` is an
  API 11 interface; `getAppPreferredLanguage` an API 9 one.
- **The effect is at launch.** "Resources are loaded in the preferred language
  when the application is launched", and `'default'` "will take effect upon cold
  starting of the application" - so do not expect every already-built screen to
  re-resolve immediately.
- **The tag must be a valid language ID or `default`**, otherwise the call throws
  890001.
- **Only resource-resolved strings switch.** Anything written as a literal in
  `.ets` stays in whatever language it was typed in.
- **`getAppPreferredLanguage` returns the system language before any preference
  has been set**, so an exact-match comparison against your own two tags will not
  match on first run.
- **Qualified directory names are fixed** - `zh_CN`, `en_US` - and must contain
  the same keys as `base`, or a missing key falls back to `base`.
- **Devices.** Per `module.json5`.

## Pitfalls

- **`setAppPreferredLanguage` is called without `try/catch`, which is
  incorrect.** It is documented to throw 401 and 890001, and the reference's own
  example guards it. Worse, `isEnglish` is assigned *before* the call, so the tick
  moves even when the language did not change. (HW-19-0120)
- **There is no way back to the system language.** `setAppPreferredLanguage('default')`
  is the documented mechanism and neither the document nor the sample offers it,
  so a tap on either row is permanent. (HW-19-0121)
- **The documented tree says `component/`, the sample ships `components/`.**
  The neighbouring document 39 has the same error in the opposite direction.
  (HW-19-0122)
- **The `onReady` comparison has no else branch.** Before any preference is set,
  `getAppPreferredLanguage()` returns the system language, which matches neither
  `'en-Latn-US'` nor `'zh-Hans'`, so 简体中文 is ticked on an English device that
  is actually displaying English.
- **`'zh-Hans'` and `'en-Latn-US'` are written three times each** - twice in the
  setters and twice in the reader. Extract them as constants; a mismatch between
  the setter and the reader shows up only as a wrong tick.
- **This is the only sample in the industry built for API 17.** Do not copy its
  `compatibleSdkVersion` into a project that uses anything from API 18 onwards.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-i18n.md` -
  `System.setAppPreferredLanguage` (the launch-time effect, the `default` value,
  error codes 401 and 890001, and the `try/catch` example) and
  `System.getAppPreferredLanguage`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-i18n
- `documentation/harmonyos-guides/01_getting-started/resource-categories-and-access.md` -
  qualified resource directories and `$r()` resolution.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/resource-categories-and-access
- https://developer.huawei.com/consumer/en/doc/harmonyos-guides/i18n-locale-culture -
  valid language IDs, referenced by the `setAppPreferredLanguage` parameter.
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `NavDestination.onReady` and `NavDestinationContext`, used to read the
  preference when the page appears.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/language_switch-0000002381506029
