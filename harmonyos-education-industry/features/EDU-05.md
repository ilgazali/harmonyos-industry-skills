---
id: EDU-05
title: Expand and collapse a biography - inline "more" label placed by measuring the text
industry: 04_education
doc: huawei_industry_tree/04_education/docs/05_spread_all_text.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
sample: huawei_industry_tree/04_education/downloads/SpreadAllText.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Text, maxLines, textOverflow, MeasureUtils, measureText, measureTextSize, StyledString, subStyledString, "display.getDefaultDisplaySync", Stack, visibility, "@StorageProp", px2vp]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0027, HW-04-0028, HW-04-0029, HW-04-0030, HW-04-0031, HW-04-0032, HW-04-0033, HW-04-0034, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **long block of prose that must collapse to N lines with an
inline "expand" affordance sitting at the end of the last visible line** - a
teacher biography, a course description, a review, a syllabus note.

The important thing to know before you start: **`maxLines` alone does not give
you this.** `maxLines` + `textOverflow(TextOverflow.Ellipsis)` gets you an
ellipsis, but nothing tappable, and nothing you can style. Putting a label
*inside* the flow at the cut point requires knowing where the cut point is, and
that means measuring.

If an ellipsis is enough, use `maxLines` + `textOverflow` and skip this card
entirely. Take this one only when the affordance must be inline.

## Feature checklist

- The biography shows two lines when collapsed.
- 展开 (expand) sits at the end of the second line, in the accent colour, on an
  opaque background so it covers the text it replaces.
- Tapping it reveals the whole text and the label becomes 收起 (collapse).
- If the text is short enough to fit in two lines, no label appears at all.
- The label is positioned so it never overlaps a partially visible glyph.

## Architecture

One `entry` module, one page, one constants file. There is no view model - the
biography is a string constant.

```
entry/src/main/ets
├── constants/StyleConstants.ets         MAX_LINE, INTRODUCTION, IMPRESSION_ARR, TEACH_INFO_ARR
├── entryability/EntryAbility.ets
└── pages/TeacherIntroductionPage.ets    @Entry - the whole page
```

The documented tree matches the zip.

**The page has two halves and the document only describes one of them.**

- The *visible* half is a `Stack({ alignContent: Alignment.BottomEnd })` holding
  the `Text` and, on top of it at the bottom-right, the expand label.
- The *invisible* half is a measurement pass in `aboutToAppear` that decides
  **how many lines the text needs** (`lineNum`) and **where to cut it**
  (`secondLineEndIndex`) so that the label has room at the end of line two.

Five fields carry the result of that pass:

| Field | Meaning |
| --- | --- |
| `lineNum` | how many lines the full text occupies; the label is hidden when `<= 2` |
| `secondLineStartIndex` | character index where line 2 begins |
| `secondLineEndIndex` | last character of line 2 that still leaves room for `...展开` |
| `labelWidth` | measured width of the `...展开` affordance |
| `atLast` | whether the tail fits on its line; drives a -12 vp nudge when expanded |

**The collapsed body is a hand-cut substring, not the full text under a
`maxLines` cap:**

```typescript
Text(this.lineNum <= 2 ? this.introduction
   : this.isSpread ? this.introduction
   : this.styledString.subStyledString(0, this.secondLineEndIndex + 1).getString())
```

That is the whole design, and it is why the measurement exists. The document
(`HW-04-0030`) presents only the `maxLines` toggle.

## Implementation steps

1. **Decide whether you need this at all.** `maxLines(2)` +
   `textOverflow({ overflow: TextOverflow.Ellipsis })` is one line of code and
   covers most cases. Continue only if the affordance must be inline.
2. **Measure the label first.** `measureText({ textContent: '...展开',
   fontSize: 10 })` - pass the font size **as a number**, so it is interpreted in
   the same unit the `Text` renders in (`HW-04-0028`, `HW-04-0029`).
3. **Compute the available width from the component, not the screen.** The
   sample uses `px2vp(display.width) - 64`, which hard-codes the sum of the page
   and card padding and is wrong in a resized or folded window. Prefer
   `onAreaChange` on the `Text`'s parent, or `measureTextSize` with an explicit
   `constraintWidth`.
4. **Walk the text to find the line breaks**, extending the substring one
   character at a time until the measured width exceeds the available width;
   that character index is where the next line starts. **Advance by at least one
   character per pass** or the loop cannot terminate (`HW-04-0031`), and step by
   Unicode code point, not by string index (`HW-04-0032`).
5. **Find the cut point on line 2** by repeating the walk from
   `secondLineStartIndex` against `textWidth - labelWidth`, so the reserved space
   for the label is excluded.
6. **Render the truncated string in a `Stack`** with the label aligned
   `BottomEnd`, on an opaque background so it masks whatever sits under it.
7. **Hide the label when `lineNum <= 2`** with
   `.visibility(Visibility.None)` - not by omitting it, so the layout does not
   reflow.
8. **On tap, swap both the label text and the body.** Bind `maxLines`
   conditionally rather than passing `-1` (`HW-04-0027`).

## Verified snippets

All snippets are from `SpreadAllText.zip`. Corrected forms are marked.

**The measurement pass — `entry/src/main/ets/pages/TeacherIntroductionPage.ets`** (corrected, see `HW-04-0028`, `HW-04-0029`, `HW-04-0031`)

```typescript
import { display, MeasureUtils } from '@kit.ArkUI';

private uiContext: UIContext = this.getUIContext();
private measureUtils: MeasureUtils = this.getUIContext().getMeasureUtils();
private display = display.getDefaultDisplaySync();

private labelWidth: number = this.uiContext.px2vp(this.measureUtils.measureText({
  textContent: '...展开',
  fontSize: 10                       // FIX: sample passes '10p', not a valid unit
}));
private textWidth: number = 0;
private lineNum: number = 1;
private startIndex: number = 0;
private secondLineStartIndex: number = -1;
private secondLineEndIndex: number = -1;
private atLast: boolean = true;

aboutToAppear(): void {
  this.textWidth = this.uiContext.px2vp(this.display.width) - 64;   // page 16*2 + card 16*2
  this.startIndex = 0;
  if (!this.introduction) {
    return;
  }

  // Pass 1 - how many lines does the whole text need, and where does line 2 start?
  to:
  while (true) {
    for (let i = 1; i <= this.introduction.length - this.startIndex; i++) {
      if (this.uiContext.px2vp(this.measureUtils.measureText({
        textContent: this.styledString.subStyledString(this.startIndex, i).getString(),
        fontSize: 10                                  // FIX: sample measures '10vp', renders fp
      })) > this.textWidth) {
        this.lineNum++;
        this.startIndex += Math.max(1, i - 1);        // FIX: sample adds i - 1, which is 0 when i === 1
        this.secondLineStartIndex =
          this.secondLineStartIndex === -1 ? this.startIndex : this.secondLineStartIndex;
        break;
      }
      if (i === this.introduction.length - this.startIndex) {
        break to;                                     // the remainder fits - done
      }
    }
  }

  // Pass 2 - where on line 2 must we stop to leave room for the label?
  if (this.lineNum > 2) {
    for (let i = 1; i <= this.introduction.length - this.secondLineStartIndex; i++) {
      if (this.uiContext.px2vp(this.measureUtils.measureText({
        textContent: this.styledString.subStyledString(this.secondLineStartIndex, i).getString(),
        fontSize: 10
      })) > this.textWidth - this.labelWidth) {       // the reserve for ...展开
        this.secondLineEndIndex = this.secondLineStartIndex + i - 2;
        break;
      }
    }
  }

  // Pass 3 - does the tail fill its line? drives the -12 vp nudge when expanded
  if (this.uiContext.px2vp(this.measureUtils.measureText({
    textContent: this.styledString
      .subStyledString(this.startIndex, this.introduction.length - this.startIndex).getString(),
    fontSize: 10
  })) > this.textWidth - this.labelWidth) {
    this.atLast = false;
  }
}
```

**Three passes, and each answers a different question.** Pass 1 finds the line
breaks - it is a hand-written greedy line breaker. Pass 2 re-breaks line 2
against a *narrower* width, `textWidth - labelWidth`, because the label will
occupy the tail. Pass 3 asks whether the final line of the expanded text has
room for 收起 on the same line, and if not the label is nudged down 12 vp.

**`measureText` returns px, always.** The reference is explicit: "Unit: px."
Every call here is wrapped in `px2vp` before being compared against `textWidth`,
which is in vp - drop one wrapper and the comparison is off by the pixel density.

**The unit trap.** `measureText`'s `fontSize` is fp when passed as a number
(since API 12) and takes the written unit when passed as a string. The `Text`
renders with `.fontSize(10)`, a number, so it is fp - and fp scales with the
user's system font-size setting. Measuring at `'10vp'` therefore agrees with the
rendering only at the default scale; turn the system font up and every index
this pass computes is wrong. Pass the number.

**The termination trap.** `while (true)` exits only via `break to`, reached when
the remainder fits on one line. The advance is `startIndex += i - 1`, so if the
*first* character already exceeds `textWidth` the branch fires with `i === 1`,
the advance is zero, and the loop spins on the UI thread forever. `Math.max(1,
i - 1)` is the whole fix.

**The visible half — same file** (corrected, see `HW-04-0027`)

```typescript
Stack({ alignContent: Alignment.BottomEnd }) {
  Text(this.lineNum <= 2 ? this.introduction
     : this.isSpread ? this.introduction
     : this.styledString.subStyledString(0, this.secondLineEndIndex + 1).getString())
    .width('100%')
    .fontSize(10)
    .fontColor(Color.Black)
    .maxLines(this.isSpread ? Number.MAX_SAFE_INTEGER : MAX_LINE)   // FIX: sample uses -1

  Row() {
    Text(this.isSpread ? $r('app.string.collapseText') : $r('app.string.spreadText'))
      .fontSize(10)
      .fontColor('#ff0a59f7')
  }
  .backgroundColor(Color.White)          // opaque - it masks the text underneath
  .justifyContent(FlexAlign.End)
  .visibility(this.lineNum > 2 ? Visibility.Visible : Visibility.None)
  .onClick(() => {
    this.isSpread = !this.isSpread;
  })
  .margin({ bottom: this.atLast ? 0 : this.isSpread ? -12 : 0 })
}
```

**Three things carry this layout.** `alignContent: Alignment.BottomEnd` pins the
label to the bottom-right corner of the `Stack`, which is the end of the last
visible line. The **opaque** `backgroundColor` is what lets it sit on top of the
text rather than beside it - remove it and the underlying glyphs show through.
`Visibility.None` rather than a conditional keeps the node in the tree, so
toggling does not reflow the card.

**`maxLines(-1)` is out of range.** The reference gives `[0, INT32_MAX]` and
documents no sentinel for "unlimited"; not setting the attribute is how you have
no limit. It appears to work, but the document publishes it as the technique
(`HW-04-0027`).

**The constants — `entry/src/main/ets/constants/StyleConstants.ets`** (as shipped)

```typescript
export const MAX_LINE: number = 2;

export const INTRODUCTION: string =
  'XX大学TESOL（对外英语教学）专业硕士，专业排名前5%，英语专业八级，...' +
  '日常教学中擅长使用幽默的风格，在潜移默化中将英语知识点传授给学生，...';
```

`MAX_LINE` is the collapsed line count and is referenced from both the initial
`@State maxLine` and the toggle - the one number worth keeping named.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block.

Note that `entry/src/main/resources/base/profile/backup_config.json` is shipped
with no `extensionAbilities` entry and no `EntryBackupAbility` to reference it
(`HW-04-0034`) - unlike `EDU-03` and `EDU-04`, which ship both.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The available width is derived from the screen, once.**
  `px2vp(display.width) - 64` assumes the page fills the display and hard-codes
  four paddings. In a floating window, a split screen or an unfolded foldable
  it is wrong, and nothing recomputes it - `aboutToAppear` runs once.
- The collapsed line count is fixed at 2 by `MAX_LINE`; the second measurement
  pass is written specifically for line 2 (`secondLineStartIndex`,
  `secondLineEndIndex`) and would need generalising for any other value.
- The measurement is O(n) `measureText` calls per line on the UI thread, at page
  entry. Acceptable for a two-paragraph biography, not for a long article.
- The 展开/收起 label reserves `...展开` worth of space in both states, so the
  cut point is computed for the longer of the two labels.

## Pitfalls

- **`HW-04-0027` — the document teaches `maxLines(-1)`,** outside the documented
  range `[0, INT32_MAX]`. Bind the attribute conditionally instead.
- **`HW-04-0028` — `labelWidth` is measured at `fontSize: '10p'`,** which is not
  a valid length unit, so `measureText` falls back to its 16 default. The space
  reserved for the affordance is measured at the wrong size.
- **`HW-04-0029` — the body is measured in vp and rendered in fp.** Pass the
  font size as a number so both use fp; otherwise every computed index is wrong
  as soon as the user changes the system font size.
- **`HW-04-0030` — the document explains only the `maxLines` toggle.** The
  measurement pass, which is the whole technique, is not mentioned. Read the zip,
  not the write-up.
- **`HW-04-0031` — the line-counting loop cannot terminate** when one character
  is wider than the available width: `startIndex += i - 1` adds zero. Clamp the
  advance to 1.
- **`HW-04-0032` — the text is cut by string index,** which the MeasureUtils
  reference warns splits multi-code-point characters. Fine for the CJK sample,
  not for user-supplied text with emoji.
- **`HW-04-0033` — `aboutToAppear` writes vp into `@StorageProp` fields bound to
  px keys.** `@StorageProp` is one-way from `AppStorage`; the next write to the
  key reverts the field to px and the padding jumps. The other three samples in
  this industry call `px2vp` at the point of use instead.
- **`HW-04-0034` — `backup_config.json` is orphaned:** no `EntryBackupAbility`,
  no `extensionAbilities` entry naming it.
- **Do not drop the label's opaque background.** It is not decoration - it is
  what hides the truncated glyphs the label sits on top of.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `maxLines` and its `[0, INT32_MAX]` range, `textOverflow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureText`, `measureTextSize`, the px return unit, and the code-point truncation warning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/js-apis-measure.md` - `MeasureOptions`, and `fontSize` being fp when numeric since API 12
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-measure
- `documentation/harmonyos-references/02_application-framework/ts-types.md` - `Length` string units
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-types
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-visibility.md` - `Visibility.None` versus removing the node
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-visibility
- `EDU-01`, `EDU-03`, `EDU-04` - the avoid-area boilerplate this page diverges from
