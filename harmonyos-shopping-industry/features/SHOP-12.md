---
id: SHOP-12
title: Expand/collapse a wrapped history block - cap a Flex with constraintSize and clip, not with a height animation
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/12_search_history_expand_and_collapse.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history_expand_and_collapse-0000002343696461
sample: huawei_industry_tree/16_shopping/downloads/SearchHistoryFolding.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Flex, FlexWrap, constraintSize, clip, visibility, Visibility, Search, searchButton, searchIcon, ForEach, "@Prop", "@State", aboutToAppear, Blank, Stack, expandSafeArea, SafeAreaType, SafeAreaEdge, hilog, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0013, HW-16-0027]
status: verified
---

## When to use

Load this card when **a block of wrapped chips must be capped at N rows with an
expand/collapse toggle** - the "search history / 展开" control at the top of
every shopping and travel search page, and the same shape as "show more tags",
"show more filters", "show all sizes".

The technique is worth knowing because the obvious approach is wrong. Chips have
different widths, they wrap, and the number of rows depends on the container
width - so you cannot compute a height from the item count, and setting
`.height()` on the container would either clip the last row mid-glyph or force
empty space. What this sample does instead is leave the `Flex` free to lay itself
out at its natural height and constrain only the *visible* box:
`constraintSize({ maxHeight })` plus `clip(true)`. Collapsed, `maxHeight` is
`rows x rowHeight`; expanded, it is `'100%'`. The layout never changes; only how
much of it you can see does.

That separation - real layout underneath, a clipping window on top - is the
transferable idea. It applies to a truncated description block, a collapsed log
pane, a "read more" article teaser. The parts of this sample that are *not*
worth transferring are the two heuristics it uses to decide the cap and the
button visibility, both of which assume a fixed chip size and a fixed three
columns; both are called out below.

**This card also carries `HW-16-0013`, a systematic defect in the published
documentation** that affects roughly forty snippets across six industries. Read
the section below before copying code out of any doc in this corpus.

## Feature checklist

- A search field at the top with a red 搜索 button and the system search icon.
- Below it, 搜索历史 with a 展开 / 收起 (expand / collapse) toggle and a trash
  icon.
- The history chips wrap freely and are clipped to two rows while collapsed.
- Tapping 展开 reveals every row and flips the label to 收起 and the chevron to
  its up-pointing asset; tapping again re-collapses.
- The toggle is present only when the history is long enough to overflow two
  rows; otherwise it occupies its space but is not drawn.
- A second block, 想搜 (want to search), with a refresh affordance and its own
  wrapped chip list, always fully expanded within a fixed 27% of the page.
- The page draws under the status bar and navigation indicator.

## Architecture

One `entry` module, one page file, three structs in it.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/SearchPage.ets       @Entry SearchPage + TextItem + TextItem2 - 202 lines
    ├── TextItem               a history chip: grey pill, 40 high, radius 20
    ├── TextItem2              a suggestion row: white, 50% wide, 40 high
    └── SearchPage             the search field, both blocks, the toggle
```

The documented tree matches the zip - though the document's tree lists only
`SearchPage.ets` and does not mention that it contains three components.

**The design decision worth copying** is that `heightControl` is typed
`number | string`, and that the expanded value is `'100%'` rather than a
computed number:

```typescript
@State heightControl: number | string = 0;
@State maxHeight: number = 0;      // 最大行数对应的最大高度
private maxNumber: number = 2;     // 最大行数
```

Collapsed, the cap is a concrete `number` (rows x 50). Expanded, it is the
string `'100%'` - a percentage of the parent, which for a `Column` that is
itself `height('100%')` means "as tall as you like". So one state variable
carries both a hard pixel cap and "no cap", and the toggle is a single ternary
between them. Constraining a *maximum* rather than setting a height is what lets
the `Flex` keep its own natural layout in both states, which is why there is no
reflow and no re-measure when the toggle is pressed.

The decision worth avoiding is deriving both the cap and the button visibility
from arithmetic instead of from measurement. `maxHeight = maxNumber * 50` assumes
every chip is exactly 50 tall (it is: `TextItem` is `height(40)` with
`margin({ top: 10 })` - but change either and the cap silently clips a row in
half), and `lineCount = count / 3` assumes exactly three chips per row regardless
of chip width or screen width. `SHOP-09`'s history block already carries the
`onAreaChange` handler that would measure this properly; wiring the two together
is the real fix.

## Implementation steps

1. **Type the cap `number | string`** so one state variable can hold both a
   pixel cap and `'100%'`.
2. **Compute the collapsed cap in `aboutToAppear`** as
   `maxNumber * rowHeight`, and seed `heightControl` from it so the first frame
   is already collapsed - never collapse after the first paint.
3. **Constrain, do not size.** Put `constraintSize({ maxHeight: this.heightControl })`
   and `clip(true)` on the `Flex`; never set `.height()` on it.
4. **Decide whether the toggle is needed at all** from the data, and hide it with
   `visibility` rather than an `if`, so the row above the chips does not reflow
   when the toggle appears.
5. **Flip three things together** in the toggle's `onClick`: the cap, the label
   (展开 / 收起) and the chevron asset. All three are `@State`, so one handler
   updates them in one pass.
6. **Compare against `maxHeight`, not against a boolean.** The handler tests
   `this.heightControl === this.maxHeight` to decide the direction, so no
   separate `isExpanded` flag can drift out of sync with the cap.
7. **Wrap with `Flex({ wrap: FlexWrap.Wrap })`,** not a `Grid` - chip widths are
   content-driven and a grid would force a column template.
8. **Take the code from the zip, not from the document** (`HW-16-0013`) - the
   doc's step-2 snippet does not parse.

## Verified snippets

All snippets are from `SearchHistoryFolding.zip`.

**The cap and the overflow test — `entry/src/main/ets/pages/SearchPage.ets`** (as shipped)

```typescript
private flexWidth: string = '100%';
@State heightControl: number | string = 0;
@State maxHeight: number = 0;      // 最大行数对应的最大高度
private maxNumber: number = 2;     // 最大行数
@State buttonText: string = '展开 ';
@State isShowButton: boolean = false;
@State imageRes: Resource = $r('app.media.ic_public_chevron_down_base_ic_public_chevron_down');

aboutToAppear(): void {
  this.maxHeight = this.maxNumber * 50;
  this.heightControl = this.maxHeight;
  if (this.allData.length > 0) {
    if (this.maxNumber > 0) {
      let count = this.allData.length;
      let lineCount = count / 3;
      if (lineCount > this.maxNumber) {
        this.isShowButton = true;
      }
    }
  }
}
```

**`heightControl` is seeded before the first layout pass, which is the whole
reason the block never flashes open.** `aboutToAppear` runs before `build()`, so
the `Flex` is measured with the cap already in place; deferring this to
`onAppear` or to an `onAreaChange` would paint one uncollapsed frame first.

The two literals are the weak points. `50` is `TextItem`'s `height(40)` plus its
`margin({ top: 10 })`, restated here with no link to the component - the two
numbers must be changed together and nothing enforces it. `count / 3` is the
guess at chips-per-row; with thirteen entries it gives 4.33 > 2 and the button
appears, which happens to be correct, but a locale with longer words fits two per
row and the estimate is wrong in the direction that hides the button on content
that overflows. Measuring the `Flex` with `onAreaChange` and comparing its
reported height against `maxHeight` needs no assumptions at all.

**The toggle — same file, the complete handler** (as shipped)

```typescript
Row() {
  Text(this.buttonText)
    .fontColor('#99000000');
  Image(this.imageRes)
    .width(17);
}
.onClick(() => {
  if (this.heightControl === this.maxHeight) {
    this.heightControl = '100%';
    this.buttonText = '收起 ';
    this.imageRes = $r('app.media.ic_public_chevron_down');
  } else {
    this.heightControl = this.maxHeight;
    this.buttonText = '展开 ';
    this.imageRes = $r('app.media.ic_public_chevron_down_base_ic_public_chevron_down');
  }
})
.width(80)
.height(30)
.visibility(this.isShowButton ? Visibility.Visible : Visibility.Hidden)
.backgroundColor(Color.White)
.padding(5);
```

**This is the snippet the document mangles** - compare it with what the doc
prints at lines 38-53, which opens the `onClick`, writes the `if`/`else`, and
then jumps straight into a `Flex(...)` without ever closing the arrow function or
the `onClick(` call. The braces do not balance and the excerpt cannot compile
(`HW-16-0013`).

Two choices here are deliberate. The direction test compares `heightControl`
against `maxHeight` rather than reading a separate `isExpanded` boolean, so there
is exactly one source of truth for the state and no way for a label to disagree
with a cap. And the button is hidden with `Visibility.Hidden` rather than
removed with an `if`, so it keeps its 80x30 box in the header `Row` - the 搜索历史
label and the trash icon stay in the same place whether or not the history
overflows. Use `Visibility.None` if you would rather reclaim the space.

**The clipped Flex — same file** (as shipped)

```typescript
Flex({ wrap: FlexWrap.Wrap }) {
  ForEach(
    this.allData,
    (item: string) => {
      TextItem({ message: item, fontSizeData: this.fontSizeData });
    }
  );
}
.constraintSize({ maxHeight: this.heightControl })
.width(this.flexWidth)
.clip(true);
```

**Three attributes, and each one is load-bearing.** `FlexWrap.Wrap` lets the
chips flow onto as many rows as they need - the `Flex` always lays out its full
content, in both states. `constraintSize({ maxHeight })` bounds the box the
parent gives it without touching that internal layout, which is why toggling is
instantaneous and why no chip is ever re-measured. `clip(true)` is what makes the
cap visible instead of merely nominal: without it the overflowing rows are drawn
outside the constrained box and the collapse does nothing.

There is no animation here, and adding one is not free: `constraintSize` is a
layout constraint, so an `animateTo` around the toggle animates a re-layout of
every chip rather than a cheap transform. If a smooth collapse matters, animate a
`height` on a wrapper `Column` around this `Flex` and leave the `Flex` itself
unconstrained.

Note the type error in this snippet as shipped: `allData` is declared
`Resource[]`, the `ForEach` callback types `item` as `string`, and `TextItem`
declares `private message: string`. A `Resource` object is handed to a
`string`-typed field and then to `Text()`. It renders, because `Text` accepts a
`ResourceStr`, but the declared types are a lie the compiler is only letting
through because `ForEach`'s callback parameter is not checked against the array's
element type. Type the callback `(item: Resource)` and `message: ResourceStr`.

## `HW-16-0013` — the snippets in these documents do not compile

This card is the anchor for a defect that is not specific to this feature.

The published architecture guides abridge their code samples, and the abridgement
step **removes structurally necessary code rather than only eliding bodies**. The
result is snippets that are cut mid-construct and do not parse. Confirmed
instances in this industry:

| Doc | What is broken |
| --- | --- |
| `01_practice-purchase-app-architecture-v1.md` | `CustomScan` snippet, stray `}` |
| `04_sign.md` | `try` block with no `catch` |
| `05_get_coupons.md` | `Promise<string>` function with no `return`; `await`s dropped |
| `07_scratch_effect.md` | `getMediaContent( id: resource.id)` - a pseudo named-argument that is not TypeScript |
| `10_scroll_celling_demo.md` | `Tabs`/`ForEach` snippet, unclosed (see `SHOP-10`) |
| `12_search_history_expand_and_collapse.md` | `onClick` handler never closed - **this doc** |
| `14_auto_height_list.md` | unclosed `if` inside `onAreaChange` |
| `23_address_manager.md` | `await` inside a non-`async` callback |

The same pipeline produces the same defect elsewhere: `11_news_reading` docs 04
and 21, four docs in `18_photography`, eleven snippets across eight docs in
`13_media_entertainment`, five in `14_social_communication`, and ten across six
docs in `15_utilities`. Roughly forty instances across six industries.

**In every case checked, the source in the zip is valid.** Only the excerpt is
mangled. So the rule for anyone working from these guides:

1. **Download the sample zip and read the real file.** The doc's 工程目录 tree
   gives you the path; the snippet's header comment sometimes does not (in
   `SHOP-10` it names a file that does not exist - `HW-16-0011`).
2. **Treat a doc snippet as a pointer, not as source.** Use it to find the
   function and to see which attributes the authors considered load-bearing, then
   copy from the zip.
3. **If a snippet is all you have, expect missing closers.** The recurring
   omissions are: the closing `}` `)` of a handler, the `catch` of a `try`, the
   `async` modifier, and the `return` of a non-void function.
4. **Do not paste a doc snippet into a build to "see what breaks".** The failure
   is a parse error at a brace count, which points at the wrong line and tells
   you nothing about the API.

The cards in this pack quote the zip source exclusively for this reason, and mark
every snippet as shipped or corrected.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. `deviceTypes` is
`phone`, `tablet`, `2in1`.

The page reaches under the system bars declaratively:

```typescript
.height('100%')
.backgroundColor('#F1F3F5')
.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM]);
```

This is the same approach as `SHOP-10` and the better one - no ability-side
`AppStorage` plumbing and no `avoidAreaChange` listener to leak. Note though that
the outermost `Column` expands under the status bar but the inner white `Column`
carries only `padding({ top: 10 })`, so the search field sits close to the status
bar on devices with a tall one.

Both chip components take `@Prop fontSizeData: number`, and `TextItem2` ignores
it in favour of a hardcoded `.fontSize(16)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The 50 vp row height is a literal that must be kept in step with `TextItem`'s
  `height(40)` + `margin({ top: 10 })` by hand.
- `count / 3` assumes three chips per row. On a wider window, or with longer
  strings, the toggle can be shown when it is not needed or hidden when it is.
- The expanded cap `'100%'` is a percentage of the **parent**, not of the
  content. The parent `Column` is `height('100%')` of the page, so with enough
  history the expanded block still clips - at the page height rather than at two
  rows. There is no scrolling anywhere on this page.
- The `Search` component's `searchButton('搜索', ...)` label is a hardcoded
  Chinese literal, not a resource, unlike the chip strings which are all
  `$r('app.string.all_dataN')`.
- The trash icon and the 刷新 (refresh) icon are decorative - neither carries an
  `onClick`. Clearing history is not implemented.
- The 想搜 block is fixed at `height('27%')` of the page and holds a `Stack` whose
  second child is an empty `Column().height('100%').width('100%')` overlaying the
  chips, which swallows taps on them.
- `allData` and `allData2` are `Resource[]` typed as `string` at every use site.

## Pitfalls

- **`HW-16-0013`** (E/medium, confirmed): Systematic - the abridged snippets in
  these documents are cut mid-construct and no longer parse. This doc's step-2
  snippet opens `.onClick(() => { if (...) {...} else {...}` and jumps straight
  into `Flex({...})` without ever closing the handler; roughly forty further
  instances span six industries, and in every case the zip source is valid while
  the excerpt is not. Fix: regenerate excerpts with brace-balanced elision
  (keeping `catch`/`async`/`return` skeletons); until then, take all code from
  the sample zip and treat the doc snippet only as a pointer to the right file
  and function. Full account in the section above.

## References

- `huawei_industry_tree/16_shopping/docs/12_search_history_expand_and_collapse.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history_expand_and_collapse-0000002343696461
- `documentation/harmonyos-references/02_application-framework/ts-container-flex.md` - `Flex` and `FlexWrap.Wrap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-flex
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-size.md` - `constraintSize` and `ConstraintSizeOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-size
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sharp-clipping.md` - `clip`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sharp-clipping
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - `Search`, `searchButton`, `searchIcon`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-area-change-event.md` - `onAreaChange`, the measured alternative to `count / 3`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-area-change-event
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and its callback typing
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `SHOP-09` - the same history chip cloud, with the delete mode this page lacks
- `SHOP-10` - the other shopping doc whose snippet is unbalanced (`HW-16-0011`, `HW-16-0013`)
