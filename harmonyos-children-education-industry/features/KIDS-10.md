---
id: KIDS-10
title: Sliding picture puzzle - one image tiled by background offset, shuffled by legal moves
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/10_pictures_huarong_road.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pictures_huarong_road-0000002367694177
sample: huawei_industry_tree/08_children_education/downloads/PicturesHuarongRoad.zip
kits: ["@kit.ArkUI", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: [backgroundImage, backgroundImagePosition, visibility, Flex, FlexWrap, ForEach, "cryptoFramework.createRandom", "Random.generateRandom", "@StorageLink", "@StorageProp", "@Link", "@Prop", "@Provide", "@Watch", bindSheet, "UIContext.px2vp", NavPathStack]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0076, HW-08-0077, HW-08-0078, HW-08-0079, HW-08-0080, HW-08-0081, HW-08-0082, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for a **sliding-tile puzzle**, or more generally for **cutting
one image into a grid of independently positioned tiles without slicing any
files**.

Two ideas here are worth taking and neither is obvious:

- **Tiles are one image seen through many windows.** Every tile is a `Flex` of
  the same size carrying the *whole* picture as its background, offset by
  `backgroundImagePosition` so that a different part of it shows through. No
  image processing, no `PixelMap`, no cropping - a 3x3 puzzle is nine frames
  and nine offsets.
- **Shuffling by replaying legal moves.** Rather than permuting the tiles and
  then testing solvability, the shuffle starts from the solved board and makes
  100 random *legal* slides. Every reachable state is by construction solvable,
  which removes the parity check a naive shuffle needs.

**Seven findings, and one is serious:** the random index divides by 255 where
it needs 256, so roughly a third of games start unshuffled or throw.

## Feature checklist

- A picture, a row count and a column count, chosen before the game starts.
- The picture is cut into `rows x cols` tiles; the last one is the gap.
- The board is shuffled by simulated legal moves.
- Tapping a tile adjacent to the gap slides it in.
- When every tile is back in place, "WIN" appears.
- The picture can be swapped from a sheet.

## Architecture

One `entry` module, two pages, three components, one model, one constants file.

```
entry/src/main/ets
├── component
│   ├── Initialize.ets     the pre-game preview
│   ├── SetPiece.ets       one tile: frame, offset, tap handler, win test
│   └── Start.ets          the board: init, shuffle, move rules
├── constants/Constants.ets      StyleConstants  (doc says CalendarConstants)
├── entryability/EntryAbility.ets
├── entrybackupability/
├── models/Pieces.ets            FigMessage, Block  (doc says model/)
└── pages
    ├── AddPicture.ets     the picture chooser
    └── MainPage.ets       config, sheet, Start/Initialize switch
```

Two names in the documented tree do not match the zip (`HW-08-0082`).

**The tile model carries both where it is and where it belongs:**

```typescript
export interface Block {
  id: number,        // fixed: position in the solved board
  x: number,         // current background offset - what this frame SHOWS
  y: number,
  targetX: number,   // the offset this frame SHOULD show
  targetY: number,
  w: number, h: number,
  imgUrl: Resource,
  isVisible: boolean // false on exactly one tile: the gap
}
```

**Blocks never move in the array.** A slide swaps `x`, `y` and `isVisible`
between two entries and leaves `id`, `targetX` and `targetY` alone. So array
position *k* is board position *k* permanently, `targetX/targetY` is the tile
that belongs there, and `x/y` is the tile currently shown there - which makes
the win test a per-element comparison with no permutation logic:

```typescript
isCorrect = (block: Block) => this.isEqual(block.x, block.targetX)
                           && this.isEqual(block.y, block.targetY);
```

That is the cleanest part of the sample. Storing "what is here" separately
from "what belongs here" on the same object is why nothing has to track a
mapping.

**`Start` is created only while `isStart` is true** (`if (this.isStart) { Start({...}) }`
in `MainPage`), so its field initialisers pick up the current `rows`/`cols`
every time a game begins. That is why `blockWidth` being computed in a field
initialiser is safe here.

## Implementation steps

1. **Build the solved board**: `id = i * cols + j`, offsets
   `x = -j * blockWidth`, `y = -i * blockHeight`, and hide the last tile.
2. **Deep-copy it** as `originalBlocks` before shuffling.
3. **Shuffle by replaying legal moves** from the gap - and draw all the random
   bytes in one call (`HW-08-0077`), with a correct index (`HW-08-0076`).
4. **Render each tile as a frame with a background offset.**
5. **On tap, test adjacency by array index**, then swap `x`, `y` and
   `isVisible`.
6. **Test the win by comparing each tile's offset with its target.**
7. **Key the `ForEach` by `id`** and let the tile observe its own data
   (`HW-08-0078`, `HW-08-0080`).

## Verified snippets

All snippets are from `PicturesHuarongRoad.zip`. Corrected forms are marked.

**Cutting the picture — `entry/src/main/ets/component/Start.ets`** (as shipped)

```typescript
private blockWidth: number = this.gameConfig.w / this.gameConfig.cols;
private blockHeight: number = this.gameConfig.h / this.gameConfig.rows;

initBlocksArray: Function = () => {
  for (let i = 0; i < this.gameConfig.rows; i++) {
    for (let j = 0; j < this.gameConfig.cols; j++) {
      this.blocks.push({
        id: i * this.gameConfig.cols + j,
        x: -j * this.blockWidth,          // negative: slide the image LEFT to
        y: -i * this.blockHeight,         // bring column j into the frame
        targetX: -j * this.blockWidth,
        targetY: -i * this.blockHeight,
        w: this.blockWidth,
        h: this.blockHeight,
        imgUrl: this.gameConfig.imgUrl,   // every tile: the SAME whole image
        isVisible: (i === this.gameConfig.rows - 1 && j === this.gameConfig.cols - 1 ? false : true)
      });
    }
  }
};
```

**The offsets are negative and that is the whole trick.** A background is
positioned by moving the image, not the window, so showing column `j` means
shifting the image `j` tile-widths to the left.

**The gap is the last tile, hidden rather than absent.** Keeping it in the
array means the move rules and the win test treat it like any other tile, and
`findIndex(block => !block.isVisible)` is all that is needed to locate it.

**One tile — `entry/src/main/ets/component/SetPiece.ets`** (corrected, see `HW-08-0080`)

```typescript
// FIX: the sample uses `@Prop block: Block` - a one-way copy - while the tap
// handler writes to the parent's array, so the tile it draws is stale and the
// refresh has to be forced. Make Block an @Observed class and bind it here:
@ObjectLink block: Block;

Flex()
  .width(this.block.w + 'px')
  .height(this.block.h + 'px')
  .borderWidth('1px')
  .borderColor(Color.White)
  .backgroundImage(this.block.imgUrl)
  .backgroundImagePosition({
    x: this.uiContext.px2vp(this.block.x),      // px model -> vp attribute
    y: this.uiContext.px2vp(this.block.y)
  })
  .visibility(this.block.isVisible ? Visibility.Visible : Visibility.Hidden)
```

**The `'px'` suffix on width/height and `px2vp` on the offset are both
deliberate.** The model is in image pixels, `width` accepts a `'…px'` string,
and `backgroundImagePosition` takes vp - so one of the two needs converting and
the sample converts the right one.

**`Visibility.Hidden`, not `Visibility.None`.** Hidden keeps the layout slot,
so the gap holds its place in the wrapping `Flex`; `None` would collapse it and
the remaining tiles would reflow.

**Sliding a tile — same file** (as shipped)

```typescript
.onClick(() => {
  let inVisibleBlock = this.blocks.find((b) => !b.isVisible);
  if (inVisibleBlock === undefined) { return; }

  let inVisibleIndex = this.blocks.findIndex(b => b.id === inVisibleBlock!.id);
  if (this.isAdjacent(this.idx, inVisibleIndex)) {
    let tempX = this.blocks[this.idx].x;
    let tempY = this.blocks[this.idx].y;
    let tempVisible = this.blocks[this.idx].isVisible;

    this.blocks[this.idx].x = inVisibleBlock.x;      // only these three fields
    this.blocks[this.idx].y = inVisibleBlock.y;      // ever move between tiles
    this.blocks[this.idx].isVisible = inVisibleBlock.isVisible;

    inVisibleBlock.x = tempX;
    inVisibleBlock.y = tempY;
    inVisibleBlock.isVisible = tempVisible;
  }

  let wrongs = this.blocks.filter((item) => !this.isCorrect(item));
  if (wrongs.length === 0) { this.isShow = true; }
});

isAdjacent: Function = (id1: number, id2: number): boolean => {
  let cols = this.gameConfig.cols;
  let row1 = Math.floor(id1 / cols);
  let row2 = Math.floor(id2 / cols);
  let col1 = id1 - cols * row1;
  let col2 = id2 - cols * row2;
  return (row1 === row2 && Math.abs(col1 - col2) === 1) ||
         (col1 === col2 && Math.abs(row1 - row2) === 1);
};
```

**Adjacency is tested on array indices, not on ids** - `this.idx` is the tile's
board position - which is correct precisely because tiles never move in the
array. The row/column decomposition also stops `Math.abs(id1 - id2) === 1` from
wrongly linking the end of one row to the start of the next.

**`isEqual` floors before comparing** because `blockWidth` is a division and
the offsets are accumulated floats; exact `===` on those would fail on a board
whose width is not divisible by its column count.

**Shuffling — `entry/src/main/ets/component/Start.ets`** (corrected, see `HW-08-0076` and `HW-08-0077`)

```typescript
shuffle = async () => {
  const currentBlocks: Array<Block> = JSON.parse(JSON.stringify(this.originalBlocks));

  // FIX: the sample creates a generator and awaits one byte inside the loop,
  // 100 times over. Draw them all at once.
  const rand = cryptoFramework.createRandom();
  const bytes = (await rand.generateRandom(100)).data;

  for (let i = 0; i < 100; i++) {
    const emptyIndex = currentBlocks.findIndex(block => !block.isVisible);
    const possibleMoves: number[] = this.getValidMoves(emptyIndex);
    if (possibleMoves.length > 0) {
      // FIX: the sample computes Math.floor((byte / 255) * (maxIndex + 1)),
      // which returns maxIndex + 1 when the byte is 255 - one past the end.
      const move = possibleMoves[bytes[i] % possibleMoves.length];
      this.swapBlocks(currentBlocks, emptyIndex, move);
    }
  }

  this.blocks = currentBlocks;
};

getValidMoves = (emptyIndex: number): Array<number> => {
  const moves: Array<number> = [];
  const row = Math.floor(emptyIndex / this.gameConfig.cols);
  const col = emptyIndex % this.gameConfig.cols;
  if (row > 0) { moves.push(emptyIndex - this.gameConfig.cols); }                        // up
  if (row < this.gameConfig.rows - 1) { moves.push(emptyIndex + this.gameConfig.cols); } // down
  if (col > 0) { moves.push(emptyIndex - 1); }                                           // left
  if (col < this.gameConfig.cols - 1) { moves.push(emptyIndex + 1); }                    // right
  return moves;
};
```

**Shuffling by legal moves is the right algorithm** and the reason to read this
sample. Half of all tile permutations of a sliding puzzle are unreachable, so
a random permutation gives an unsolvable board half the time; replaying moves
from the solved state cannot leave the reachable set. It also means no parity
computation and no retry loop.

**The bounds are checked per direction, not by array range** - `row > 0` for
up, `col > 0` for left - which is what stops the gap wrapping from the left
edge of one row to the right edge of the row above.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The pictures are
bundled resources named `_0`, `_1`, … and selected by index:

```typescript
@StorageLink('Url') @Watch('onCountUpdated') url: number = 0;

onCountUpdated(): void {
  this.gameConfig.imgUrl = $r(`app.media._${this.url}`);
  this.isInitial = !this.isInitial;
}
```

Building a resource reference by string interpolation works but silently
resolves to nothing if the index has no matching file - there is no guard.

Shared state goes through `AppStorage`: `GameConfig` holds the board
configuration and `Url` the picture index, so `MainPage` and `AddPicture` stay
in step without a route parameter.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Every picture must be exactly 1100 x 1100 pixels.** `w` and `h` are fixed in
  the config and never updated when the picture changes (`HW-08-0079`).
- **The gap is always the bottom-right tile** at the start, and the win test
  requires it back there.
- **No move counter, no timer, no undo, no hint** - and no persistence, so a
  game in progress is lost when the page goes away.
- **The board is not animated.** A slide is an instantaneous offset swap;
  there is no `animateTo` anywhere in the sample.
- **Rows and columns are entered as text** and parsed with `parseInt` with no
  validation, so a non-numeric or absurd value reaches the grid arithmetic.
- The whole board rebuilds on every tap, because the array is replaced with a
  spread copy and a refresh boolean is toggled (`HW-08-0078`, `HW-08-0080`).

## Pitfalls

- **`HW-08-0076` — the random move index divides by 255 instead of 256.** A
  byte of 255 selects `maxIndex + 1`, `possibleMoves[…]` is `undefined`,
  `undefined !== null` passes the guard, and `swapBlocks` dereferences
  `blocks[undefined]`. Over 100 iterations that is about a 32% chance per game;
  when it fires the assignment after the loop never runs and the child is shown
  an already-solved puzzle.
- **`HW-08-0077` — the shuffle awaits 100 separate one-byte crypto calls**
  while the solved picture is on screen, and the promise is neither awaited nor
  caught by the caller.
- **`HW-08-0078` — `ForEach(this.isShuffle ? this.blocks : this.blocks, …)`**
  has identical branches and no key generator. The refresh works only through
  the incidental dependency on `isShuffle`, so simplifying the ternary breaks
  the board.
- **`HW-08-0079` — the image size is hardcoded to 1100 x 1100** while the
  picture is user-selectable and only `imgUrl` is updated, so any other size is
  tiled wrongly with no error.
- **`HW-08-0080` — `SetPiece` renders from a `@Prop` copy** while writing to
  the parent's array, so two brute-force triggers (`[...this.blocks]` and
  toggling `isShuffle`) rebuild every tile on each move. `Block` is an
  interface, so nothing on it can be observed.
- **`HW-08-0081` — `getValidMoves` is declared with one parameter and called
  with two**, hidden by the field being typed `Function`; and `@Observed` on
  `FigMessage` never applies, because the only value of that type is an object
  literal and no `@ObjectLink` exists anywhere.
- **`HW-08-0082` — the documented tree names `constants/CalendarConstants.ets`
  and `model/Pieces.ets`;** the zip has `constants/Constants.ets` and
  `models/Pieces.ets`. `CalendarConstants.ets` is the real filename in the
  tangram sample, so the tree was copied between documents.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-background.md` - `backgroundImage`, `backgroundImagePosition`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-background
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - the key generator and the default fallback
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - the pairing that would replace the forced refresh
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createRandom`, `generateRandom`, `DataBlob.data`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - drawing several bytes in one call
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-visibility.md` - `Hidden` versus `None`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-visibility
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageLink`, `@StorageProp`, `@Watch`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `KIDS-06` - the other piece-arrangement game, and the source of the `CalendarConstants` filename
- `KIDS-15` - the Schulte grid, the industry's other index-arithmetic-on-a-grid sample
- `KIDS-02` - the other `cryptoFramework` random-number consumer, with a different byte-handling bug
