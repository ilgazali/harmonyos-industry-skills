---
id: COMMON-25
title: Accessibility screen reading - give every non-text control a spoken label, and group tiles into one stop
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/25_accessibility.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/accessibility-0000002320999085
sample: huawei_industry_tree/19_common_technical_solutions/downloads/Accessibility.zip
kits: ["@kit.ArkUI"]
apis: ["component .accessibilityText", "component .accessibilityGroup", "component .accessibilityLevel", "component .accessibilityDescription", CommonModifier, Navigation, NavPathStack, "NavPathStack.pushPath", routerMap, Tabs, TabContent, Grid, GridItem, "@Provide", "@Consume", "util.format"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0079, HW-19-0080, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must be usable with the **system screen
reader** - the document's framing: 面向视障人士开发的一项功能，可以在点击屏幕时朗读出
对应的内容 ("a feature for visually impaired users that reads out the
corresponding content when the screen is touched").

The concrete problem this solves is that icon-only controls announce nothing. A
`Search` box, a scan icon, an avatar, a tile made of an image plus two lines of
text - the screen reader either says nothing or says the wrong thing unless the
application supplies the label and decides what counts as one stop.

The document names one attribute, `accessibilityText`. The sample uses a second
one that matters just as much - `accessibilityGroup` - and getting the pair right
is the whole technique (HW-19-0080).

## Feature checklist

The application must:

- Give every control without text content an `accessibilityText`.
- Keep screen-reader strings in their own constants file, separate from display
  strings.
- Mark an aggregate (a tile, a tab bar item) with `accessibilityGroup(true)` so
  it is a single stop, and give that group one `accessibilityText` covering all
  of its lines.
- Expand values the reader would mangle - a masked phone number read digit by
  digit rather than as one number.
- Announce selection state where the visual carries it (选中 / 未选中).
- Not set `accessibilityText` on a container whose children should stay
  individually focusable, and not leave a container ungrouped when it should be
  one stop.

## Architecture

Single-module project (`entry` HAP), a small telecom-account app:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `common/Constants.ets` | `CommonConstants` - layout values and display strings |
| `common/AccessibilityConstants.ets` | `AccessibilityConstants` - **only** the strings the screen reader speaks |
| `model/GridInfo.ets` | `GridInfo` and `GRID_COMPONENT`, the home-screen tiles |
| `pages/HomePage.ets` | the home: search row, avatar row, tile grid, and the bottom `Tabs`; owns the `NavPathStack` and `@Provide`s the account data |
| `pages/Charge.ets` | top-up page - the phone-number readout and the amount selection |
| `pages/Cards.ets` | asset query page |
| `pages/Order.ets` | order confirmation page |

**The separation of the two constants classes is the design.** `CommonConstants`
holds what is drawn; `AccessibilityConstants` holds what is spoken:

```ts
export class AccessibilityConstants {
  static readonly SEARCH = '搜索关键词';
  static readonly SCAN = '扫描';
  static readonly AVATAR = '头像';
  static readonly USER = '亲爱的用户';
  static readonly CHECKED = '已选中';
  static readonly UNCHECKED = '未选中';
  static readonly PHONE_FIRSTTHREE = '正在为手机号前三位';
  static readonly ADD = '添加';
}
```

That split is why the document's snippet, which reads both from
`CommonConstants`, does not match the project (HW-19-0079).

**Three shapes of annotation appear in the sample:**

1. **Leaf control with no text** - `Image($r('app.media.scan'))`,
   `Image($r('app.media.user'))`: one `accessibilityText`, nothing else.
2. **Aggregate that should be one stop** - the tab bar item, the amount card in
   `Charge`: `accessibilityGroup(true)` **plus** `accessibilityText`.
3. **Value that must be re-spelled for speech** - the masked phone number in
   `Charge`, expanded with `util.format` into digit-by-digit text so the reader
   does not run the digits together.

## Implementation steps

1. **Create a dedicated accessibility constants class.** Keeping spoken strings
   apart from display strings is what makes them reviewable as a set.
2. **Label every non-text control**: `.accessibilityText(AccessibilityConstants.X)`
   on icons, images and search boxes.
3. **Decide, per container, whether it is one stop or several.**
   - One stop: `.accessibilityGroup(true)` and one `accessibilityText` that
     covers every line the user needs (HW-19-0080).
   - Several stops: no `accessibilityText` on the container; label the children.
4. **Do not set `accessibilityText` on a component that already has the right
   text content** unless you intend to replace it - the reference is explicit
   that "If a component has both text content and accessibility text, only the
   accessibility text is announced."
5. **Re-spell values for speech** where the raw string reads badly, as `Charge`
   does for the phone number.
6. **Announce state** with dedicated strings (`已选中` / `未选中`) rather than
   relying on colour or border.
7. **Test with the screen reader actually on** - the document notes
   使用前需开启手机屏幕朗读功能 ("the phone's screen-reading function must be
   enabled first").

## Verified snippets

All snippets below come from the sample project, not from the document.

### Labelling leaf controls

`Accessibility.zip#Accessibility/entry/src/main/ets/pages/HomePage.ets`

```ts
import { AccessibilityConstants } from '../common/AccessibilityConstants';

Row() {
  Search({ placeholder: CommonConstants.SEARCH })
    .width(CommonConstants.SEARCH_INDEX_WIDTH)
    .accessibilityText(AccessibilityConstants.SEARCH)
  Image($r('app.media.scan'))
    .width(CommonConstants.SCAN_WIDTH)
    .margin({ left: CommonConstants.SCAN_WIDTH_MARGIN_LEFT })
    .accessibilityText(AccessibilityConstants.SCAN)
}

Row() {
  Image($r('app.media.user'))
    .width(CommonConstants.AVATAR_WIDTH)
    .margin({ left: CommonConstants.AVATAR_MARGIN_LEFT })
    .accessibilityText(AccessibilityConstants.AVATAR)
  Text(CommonConstants.USER)
    .accessibilityText(AccessibilityConstants.USER)
    .fontSize(CommonConstants.AVATAR_FONTSIZE)
    .fontWeight(FontWeight.Bold)
    .margin({ left: CommonConstants.AVATAR_TEXT_MARGIN_LEFT })
}
```

Note this is the block the document reproduces - but against `CommonConstants`
rather than `AccessibilityConstants` (HW-19-0079).

### The correct group pattern

`Accessibility.zip#Accessibility/entry/src/main/ets/pages/HomePage.ets`

```ts
@Builder
tabBuilder(title: string, selectedImg: ResourceStr) {
  Column() {
    Image(selectedImg)
      .width(CommonConstants.TAB_IMAGE_WIDTH)
    Text(title)
      .fontSize(CommonConstants.TAB_TITLE_FONTSIZE)
  }
  .margin({ top: CommonConstants.TAB_COLUMN_MARGIN_TOP })
  .accessibilityGroup(true)
  .accessibilityText(title)
  .height(CommonConstants.TAB_COLUMN_HEIGHT)
}
```

`Accessibility.zip#Accessibility/entry/src/main/ets/pages/Charge.ets` uses the
same pair on the amount card:

```ts
}
.margin({ top: CommonConstants.TAB_CHARGE_COLUMN_MARGIN_TOP })
.accessibilityGroup(true)
.accessibilityText(name)
```

### The tile grid (as shipped - see HW-19-0080)

`Accessibility.zip#Accessibility/entry/src/main/ets/pages/HomePage.ets`

```ts
Grid() {
  ForEach(GRID_COMPONENT, (item: GridInfo) => {
    GridItem() {
      Column() {
        Image(item.icon)
          .width(CommonConstants.GRID_ICON_WIDTH)
          .backgroundColor(item.backgroundColor)
          .borderRadius(CommonConstants.GRID_ICON_BORDERRADIUS)
          .padding(CommonConstants.GRID_ICON_PADDING)
          .position({ x: CommonConstants.GRID_ICON_POSITION_X, y: CommonConstants.GRID_ICON_POSITION_Y })
        Column() {
          Text(item.title)
            .fontSize(CommonConstants.GRID_TITLE_FONTSIZE)
            .fontWeight(FontWeight.Bold)
          Text(item.component)
            .fontSize(CommonConstants.GRID_COMPONENT_FONTSIZE)
            .margin({ top: CommonConstants.GRID_COMPONENT_MARGIN_TOP })
            .fontColor(CommonConstants.GRID_COMPONENT_FONTCOLOR)
            .fontWeight(FontWeight.Medium)
        }
        .alignItems(HorizontalAlign.Start)
        .position({ x: CommonConstants.GRID_WORD_POSITION_X, y: CommonConstants.GRID_WORD_POSITION_Y })
      }
      .width(CommonConstants.GRID_ITEM_COLUMN_WIDTH)
      .alignItems(HorizontalAlign.Start)
      .accessibilityText(item.title)      // FIX (HW-19-0080): add .accessibilityGroup(true)
    }
    .onClick(() => {
      if (item.title === CommonConstants.CHARGE) {
        this.pathStack.pushPath({ name: 'charge' })
      } else if (item.title === CommonConstants.CARDS) {
        this.pathStack.pushPath({ name: 'cards' })
      }
    })
    // ... size, background, radius ...
  })
}
.padding(CommonConstants.GRID_PADDING)
.columnsTemplate(CommonConstants.GRID_COLOUMNSTEMPLATE)
```

### Re-spelling a value for speech

`Accessibility.zip#Accessibility/entry/src/main/ets/pages/Charge.ets`

```ts
Text(this.phoneNumber)
  .accessibilityText(util.format('手机号前三位%s%s%s后四位%s%s%s%s',
    this.number.get(this.telephoneNumber[0]),
    this.number.get(this.telephoneNumber[1]),
    this.number.get(this.telephoneNumber[2]),
    this.number.get(this.telephoneNumber[4]),
    this.number.get(this.telephoneNumber[5]),
    // ...
  ))
```

The masked number is `['1', '3', '9', '****', '7', '6', '5', '9']`
(`HomePage.ets`, `@Provide telephoneNumber`), and the map turns each digit into
its spoken form so the reader does not run them together or try to read the
mask.

### The accessibility string table

`Accessibility.zip#Accessibility/entry/src/main/ets/common/AccessibilityConstants.ets`

```ts
export class AccessibilityConstants {
  static readonly SEARCH = '搜索关键词'
  static readonly SCAN='扫描'
  static readonly AVATAR='头像'
  static readonly USER='亲爱的用户'
  static readonly CHECKED='已选中'
  static readonly UNCHECKED='未选中'
  static readonly PHONE_FIRSTTHREE='正在为手机号前三位'
  static readonly ADD='添加'
}
```

## Permissions & config

**No permissions are required** and none are declared - the accessibility
attributes are ordinary universal component attributes, and the screen reader is
a system service the application does not talk to directly.

`Accessibility.zip#Accessibility/entry/src/main/module.json5` declares the usual
single `EntryAbility` with the home skill, an `EntryBackupAbility`, a `routerMap`
for the `charge` / `cards` / `order` destinations, and no `requestPermissions`
block.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `accessibilityText` and
  `accessibilityGroup` are atomic-service APIs since API 11 and widget APIs since
  API 12.
- **The screen reader must be enabled** to observe any of this - the document
  says so explicitly.
- **`accessibilityText` replaces text content, it does not add to it**: "If a
  component has both text content and accessibility text, only the accessibility
  text is announced."
- **`accessibilityGroup(true)` removes the children from the focus order**: "the
  component and all its children are treated as a single selectable unit, and the
  accessibility service will no longer focus on the individual child components."
- **A grouped component with no text of its own concatenates its children's
  text** in depth-first order - and by default uses their *text content*, not
  their accessibility text. To prefer accessibility text, use the API-14
  `accessibilityGroup(isGroup, { accessibilityPreferred: true })` overload.
- **A child can escape the group** by setting `accessibilityLevel` to `"yes"`.
- **`accessibilityGroup` cannot be called inside `attributeModifier`.**
- **All spoken strings in this sample are hardcoded Chinese literals**, not
  resources - fine for a demo, but a localised application must move them into
  `resources` alongside the display strings.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The document's snippet reads the labels from `CommonConstants`, which is
  incorrect.** The sample keeps them in `AccessibilityConstants`, and
  `CommonConstants.SCAN` does not exist at all - only `SCAN_WIDTH` and
  `SCAN_WIDTH_MARGIN_LEFT`. Copying the snippet does not compile against the
  project it links to. (HW-19-0079)
- **The tile container sets `accessibilityText` without
  `accessibilityGroup(true)`, which is incorrect.** The children stay focusable,
  so the title is announced twice - once by the container, once by its own
  `Text` - and the subtitle never appears in the container's announcement,
  because accessibility text replaces content rather than extending it. The tab
  bar and the `Charge` amount card in the same project pair the two correctly.
  (HW-19-0080)
- **`accessibilityText` on a `Text` that already reads correctly is redundant at
  best.** `Text(CommonConstants.USER).accessibilityText(AccessibilityConstants.USER)`
  works only because the two constants happen to hold the same string; if either
  is edited the spoken and displayed text silently diverge.
- **Grouping a container whose children carry only accessibility text loses that
  text.** The default concatenation uses the children's *universal text
  attributes*; children with only accessibility text "won't be used in the merged
  text" unless `accessibilityPreferred` is set.
- **Only 14 accessibility annotations exist across four pages.** Everything else
  - the amounts, the divider rows, the images in `Cards` and `Order` - is
  unlabelled. Treat the sample as a demonstration of the attributes, not as an
  audited accessible application.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-accessibility.md` -
  `accessibilityText` ("If a component lacks text content, you can set the
  accessibility text attribute...", "If a component has both text content and
  accessibility text, only the accessibility text is announced"),
  `accessibilityGroup` (both overloads, the child-focus rule, the concatenation
  rule and `accessibilityPreferred`), `accessibilityLevel` and
  `accessibilityDescription`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-accessibility
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  `ForEach` key generation, relevant to the tile grid.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `Navigation` / `NavPathStack.pushPath` and the `routerMap` profile used for the
  three sub-pages.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/accessibility-0000002320999085
