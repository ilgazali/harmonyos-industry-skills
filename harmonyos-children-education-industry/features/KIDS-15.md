---
id: KIDS-15
title: Schulte grid - a timed concentration game on a resizable Grid
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/15_grid_focus_training.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/grid_focus_training-0000002399252313
sample: huawei_industry_tree/08_children_education/downloads/GridFocusTraining.zip
kits: ["@kit.ArkUI", "@kit.CryptoArchitectureKit"]
apis: [Grid, GridItem, columnsTemplate, rowsTemplate, columnsGap, rowsGap, ForEach, TextTimer, TextTimerController, "cryptoFramework.createRandom", generateRandomSync, "@ObservedV2", "@Trace", "@State", "@Link", "@Watch", Stack]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0110, HW-08-0111, HW-08-0112, HW-08-0113, HW-08-0114, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for a **grid of shuffled cells that must be tapped in order** -
a Schulte table here, and equally a memory game, a number-line exercise, a
sequencing puzzle.

Two things are worth taking:

- **The grid size is one number driving one template string.** A single
  `specificationsNumber` selects `'1fr 1fr 1fr'`, `'1fr 1fr 1fr 1fr'` or five,
  and the same string is passed to both `columnsTemplate` and `rowsTemplate`,
  so a square grid of any size is one assignment.
- **Ordering is enforced by a single counter, not by per-tile state.**
  `searchNumber` is the number the child must click next; a tap that does not
  match it does nothing. There is no "which tiles are done" collection.

**Five findings, and the first is a compile-time error**: the item class is
decorated with state management V2 (`@ObservedV2` / `@Trace`) and held in V1
containers (`@State` / `@Link`), which the guide states is not allowed.

## Feature checklist

- A square grid of shuffled numbers, 3x3, 4x4 or 5x5.
- A count-up timer.
- Tapping numbers in ascending order marks them; out-of-order taps are ignored.
- Completing the grid stops the timer.
- Restart reshuffles; changing the size rebuilds the grid.

## Architecture

One `entry` module, one page, one component, one model. 511 lines.

```
entry/src/main/ets
├── components/GridFocusCom.ets   the Grid, the tiles, the tap rule
├── constants/Constants.ets       CommonConstant, MarginPadding
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/GridFocusModel.ets      GridFocusItem, the shuffle, the timer controller
└── pages/Pages.ets               size selector, timer, restart, all the state
```

The documented tree matches the zip.

**The layout is one string used twice:**

```typescript
if (this.specificationsNumber === CommonConstant.FIVE) {
  this.layoutFormat = '1fr 1fr 1fr 1fr 1fr'
} else if (this.specificationsNumber === CommonConstant.FOUR) {
  this.layoutFormat = '1fr 1fr 1fr 1fr'
} else if (this.specificationsNumber === CommonConstant.JUDGE_MESSAGE_IF_THREE) {
  this.layoutFormat = '1fr 1fr 1fr'
}
// ...
.columnsTemplate(this.layoutFormat)
.rowsTemplate(this.layoutFormat)
```

**Setting `rowsTemplate` as well as `columnsTemplate` is what makes the grid
fill its box** rather than sizing rows to content - the cells then divide the
available height evenly, which is what a Schulte table needs.

**The size change is driven by `@Watch`:**

```typescript
@State @Watch('watchSpecificationsNumber') specificationsNumber: number = 5;
```

so picking a new size updates the template and rebuilds the tile array from one
assignment.

**The whole ordering rule is four lines**, and the counter is the only state:

```
searchNumber = 1
tap(tile) ──> tile.textNumber === searchNumber ? searchNumber++ ; tile.ifOnclick = false
                                               : ignore
searchNumber === length + 1 ──> timer.pause()
```

## Implementation steps

1. **Pick one state management system** - V2 throughout, or V1 throughout
   (`HW-08-0110`).
2. **Build 1..n² and shuffle with Fisher-Yates**, bounding the swap index by
   `i` (`HW-08-0111`).
3. **Draw all the random bytes in one call** (`HW-08-0112`).
4. **Derive both templates from one size number.**
5. **Key the `ForEach` on the tile's number** (`HW-08-0114`).
6. **Compare the tapped number against the counter** and advance only on a
   match.
7. **Stop the timer once**, when the counter passes the last tile
   (`HW-08-0113`).

## Verified snippets

All snippets are from `GridFocusTraining.zip`. Corrected forms are marked.

**The tile model — `entry/src/main/ets/model/GridFocusModel.ets`** (corrected, see `HW-08-0110`)

```typescript
// FIX: as shipped this class is @ObservedV2 with @Trace properties, while
// Pages.ets holds it in `@State gridFocusArr` and GridFocusCom.ets in
// `@Link gridFocusArr`, both inside plain @Component structs. The guide:
// "Classes decorated by @ObservedV2 and @Trace cannot be used together with
// @State or other decorators of V1. Otherwise, an error is reported at
// compile time."
//
// Either go V2 all the way - @ComponentV2 + @Local/@Param on both components -
// or drop to V1:
@Observed
export class GridFocusItem {
  textNumber: number;
  ifOnclick: boolean;

  constructor(textNumber: number, ifOnclick: boolean) {
    this.textNumber = textNumber;
    this.ifOnclick = ifOnclick;
  }
}
```

**The choice matters because of what the tap handler does.** It writes a
*property* of an item already in the array - `item.ifOnclick = false` - rather
than reassigning the array. V1 `@State` observes assignment, not nested
property writes; that is exactly the gap `@ObservedV2`/`@Trace` (or V1's
`@Observed`/`@ObjectLink`) exists to close. Mixing the two picks neither.

**Shuffling — same file** (corrected, see `HW-08-0111` and `HW-08-0112`)

```typescript
export function fillObjectWithRandomNumbers(specificationsNumber: number): GridFocusItem[] {
  const gridItemArr: GridFocusItem[] = [];
  const numbers: number[] = [];
  for (let index = 0; index < specificationsNumber * specificationsNumber; index++) {
    numbers[index] = index + 1;
  }

  // FIX: the sample creates a generator per iteration - draw them all at once
  const bytes = cryptoFramework.createRandom().generateRandomSync(numbers.length).data;

  for (let i = numbers.length - 1; i > 0; i--) {
    // FIX: the sample computes
    //   Math.floor(Number(randOutput.data[0] / 255) * (i + 1))
    // A byte of 255 divides to exactly 1.0, so j becomes i + 1 - one past the
    // largest index Fisher-Yates may pick - and the guard `if (j <= numbers.length)`
    // lets it through. On the first iteration that reads numbers[numbers.length],
    // which is undefined.
    const j = bytes[i] % (i + 1);
    const temp = numbers[i];
    numbers[i] = numbers[j];
    numbers[j] = temp;
  }

  for (let index = 0; index < specificationsNumber * specificationsNumber; index++) {
    gridItemArr.push(new GridFocusItem(numbers[index], true));
  }
  return gridItemArr;
}
```

**Fisher-Yates picks `j` in `[0, i]`, inclusive of `i`** - that is what makes
the permutation uniform. Bounding by anything else, including the array length,
is not a stricter check but a different (wrong) algorithm.

**`% (i + 1)` is the cheap correct mapping** for a byte onto `[0, i]` while
`i + 1` divides 256 or is small enough for the bias not to matter; the general
form rejects bytes above `Math.floor(256 / (i + 1)) * (i + 1)`.

**Building the grid — `entry/src/main/ets/components/GridFocusCom.ets`** (corrected, see `HW-08-0113` and `HW-08-0114`)

```typescript
Grid() {
  ForEach(this.gridFocusArr, (item: GridFocusItem) => {
    GridItem() {
      Stack() {
        Row()                                     // the 3D "lip" under each tile
          .position({ bottom: -MarginPadding.FOUR })
          .borderRadius({ bottomRight: MarginPadding.FOUR, bottomLeft: MarginPadding.FOUR })
          .backgroundColor(CommonConstant.STEREOSCOPIC_BACKGROUND_COLOR)
          .width(CommonConstant.FULL_WIDTH)
          .height(CommonConstant.TEN);

        Text(item.textNumber + '')
          .fontSize(this.specificationsNumber === CommonConstant.JUDGE_MESSAGE_IF_THREE ?
            CommonConstant.SEVENTY_TWO : CommonConstant.THIRTY_SIX)
          .backgroundColor(item.ifOnclick ? CommonConstant.CLICK_COLOR_1 : CommonConstant.CLICK_COLOR_2)
          .width(CommonConstant.FULL_WIDTH)
          .height(CommonConstant.FULL_HEIGHT)
          .textAlign(TextAlign.Center)
          .borderRadius(MarginPadding.FOUR);
      };
    }
    .onClick(() => {
      if (this.isPaused) { return; }              // FIX: sample reads `if (!this.isStart)`
      if (this.searchNumber === item.textNumber) {
        this.searchNumber += CommonConstant.ONE;
        item.ifOnclick = false;
        if (this.searchNumber === this.gridFocusArr.length + CommonConstant.ONE) {
          textTimerController.pause();            // FIX: sample also rewinds searchNumber
        }
      }
    });
  }, (item: GridFocusItem) => item.textNumber.toString());   // FIX: sample has no key
}
.columnsGap(MarginPadding.FOUR)
.rowsGap(MarginPadding.EIGHT)
.padding(MarginPadding.SIXTEEN)
.columnsTemplate(this.layoutFormat)
.rowsTemplate(this.layoutFormat);
```

**The offset `Row` behind each tile is the whole 3D effect** - a bar pushed
`-4` below the tile with rounded bottom corners, so the number appears to sit
on a block. Four lines instead of a shadow or an image.

**Font size switches only for the 3x3 grid** - 72 against 36 - because that is
the size where the cells are large enough to carry it. The 4x4 and 5x5 share
one value.

**Restarting — `entry/src/main/ets/pages/Pages.ets`** (as shipped)

```typescript
textTimerController.reset();
// ...
textTimerController.start();
```

The timer is a module-level `TextTimerController` shared between the page and
the tile component, which is what lets a tap deep in the grid pause a timer
the page owns. That is a reasonable use of module scope for a single-instance
control - contrast `KIDS-13`, where the same pattern carries eight mutable
statics.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. No storage, no
network - the grid is generated in memory and nothing is persisted, so scores
do not survive the page.

`Constants.ets` holds both the numeric literals and the colours. Note the
naming: `CommonConstant.JUDGE_MESSAGE_IF_THREE` is the number 3, used as a grid
size - a constant named for a comparison rather than for its value, which makes
the size branches harder to read than the raw numerals would be.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Three fixed sizes.** 3x3, 4x4 and 5x5 are hardcoded as an if/else chain
  over template strings; a 6x6 means another branch.
- **Ascending order only.** The document states it - 仅支持按照从小到大的顺序
  依次点击 - and there is no descending mode, no colour variant and no
  distractor mode, which are the usual Schulte variations.
- **No score, no history, no best time.** The timer stops on completion and
  nothing is recorded.
- **The grid is rebuilt from scratch on every size change and restart**, which
  is also when the whole shuffle runs synchronously in `aboutToAppear`.
- The completion branch can re-enter, because it rewinds the counter to a value
  that still matches the last tile (`HW-08-0113`).

## Pitfalls

- **`HW-08-0110` — `@ObservedV2`/`@Trace` on the item class, `@State`/`@Link` in
  the components.** The guide states this is a compile-time error, and the tap
  handler's `item.ifOnclick = false` is exactly the nested-property write that
  neither system is then observing.
- **`HW-08-0111` — the shuffle divides the byte by 255**, so a maximum byte
  yields `i + 1`, and the guard tests against `numbers.length` rather than `i`,
  admitting it. On the first iteration that reads past the end and writes
  `undefined` into a tile - which renders as `undefined` and can never be
  clicked, so the game becomes unwinnable. About 9% of 5x5 games.
- **`HW-08-0112` — a `createRandom()` generator per loop iteration**, 24 of them
  for a 5x5 grid, synchronously in `aboutToAppear`.
- **`HW-08-0113` — the guard is `if (!this.isStart)`** on a flag initialised
  `true` and spelled `ifStart` in the page and `isStart` in the component; and
  the completion branch rewinds `searchNumber` to a value that still matches the
  last tile, so it re-enters and re-pauses.
- **`HW-08-0114` — the `ForEach` has no key generator**, so the default key is
  derived from the serialised item and changes every time a tile is marked -
  rebuilding the tile instead of updating it.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2`, `@Trace`, and the prohibition on mixing with V1
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - the V1 alternative
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `columnsTemplate`, `rowsTemplate`, `columnsGap`, `rowsGap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - the key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimerController`, `start`, `pause`, `reset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createRandom`, `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - drawing several bytes in one call
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `KIDS-10` - the sliding puzzle, with the same divide-by-255 mistake in its shuffle
- `KIDS-02` - the other `cryptoFramework` consumer, with a third variant of the same byte-handling bug
- `KIDS-05` - the other grid-arithmetic sample
