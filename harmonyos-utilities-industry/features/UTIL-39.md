---
id: UTIL-39
title: Four-up meeting grid - chunk a participant list into Swiper pages driven by LazyForEach
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/39_swiper_conference_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/swiper_conference_page-0000002538460457
sample: huawei_industry_tree/15_utilities/downloads/SwiperConferencePage.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Swiper, SwiperController, LazyForEach, IDataSource, DataChangeListener, Flex, FlexWrap, Stack, TextTimer, TextTimerController, DotIndicator, hilog, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0082, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to show **a list of equal tiles, paginated by
swipe rather than scrolled** - a video-conference gallery view, a classroom
roster, a camera-wall, a dashboard of device cards. The technique is one line
of arithmetic wrapped in a data source: chunk the flat list into arrays of N,
hand the array-of-arrays to `LazyForEach` inside a `Swiper`, and let each page
lay its N items out with a wrapping `Flex`.

Two ideas generalise past the conference framing. The first is that a page in a
`Swiper` does not have to be homogeneous: index 0 here is a **speaker view**
built from an empty chunk pushed in front of the real data, and every later
index is a grid. Pushing a sentinel element and branching on `index === 0` is a
cheaper way to get a heterogeneous pager than nesting two `Swiper`s. The second
is the `LazyDataSource<T>` file itself - a complete, reusable `IDataSource`
implementation with `notifyDataAdd` / `notifyDataDelete` / `notifyDatasetChange`
wired up - which is worth lifting into any project that uses `LazyForEach`.

**Do not copy the tap handler** (`HW-15-0082`), and do not copy the key
generator: both are covered below.

## Feature checklist

- A status bar across the top: 小红的会议 (Xiaohong's meeting) with a pull-down
  chevron, a running `TextTimer` in HH:mm:ss, zoom-out and speaker icons, and a
  red 离开 (leave) button.
- The body is a swipeable pager with a dot indicator, looping disabled.
- Page 0 is the speaker view: one large avatar filling the page and one small
  bordered thumbnail pinned to the top right.
- Tapping page 0 swaps who is large and who is small.
- Pages 1..n each show up to four participant tiles in a 2x2 grid.
- Each tile shows an avatar - a photo for 小明, otherwise a blue circle holding
  the second character of the name - with the name underneath and a muted-mic
  chip anchored bottom-left.
- Tapping a tile removes that participant and repaginates.
- The bottom bar shows five controls and a live 与会者(n) participant count.

## Architecture

One `entry` module, one page, one data-source util and one tiny model file.

```
entry/src/main/ets
├── Mode/ObservedArray.ets       @Observed class ObservedArray<T> extends Array<T>
├── Utils/LazyDataSource.ets     BasicDataSource<T> + LazyDataSource<T>: the reusable IDataSource
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/MeetingSwiper.ets      @Entry: ItemParam, the whole UI, 389 lines
```

The documented 工程目录 does not match the zip: it names the directories
`model` and `utils` where the zip ships `Mode` and `Utils`, both capitalised
differently. On a case-sensitive build the two import paths in
`MeetingSwiper.ets` (`'../Utils/LazyDataSource'`) resolve and the document's
tree does not describe the shipped project.

**The design decision worth copying** is that the pagination lives entirely in
`resetDataArr()` and nowhere else. `this.list` is the single source of truth - a
flat `ItemParam[]` - and the `LazyDataSource` is a *derived* view rebuilt from
scratch after every mutation. There is no incremental index maths, no
"which page did this participant move to", no partial-update bookkeeping. Any
change to the list is one call away from a correct UI:

```typescript
this.list.splice(index, 1);
this.resetDataArr();
```

The cost is that `pushDataPositionArray` ends in `notifyDataReload()`, which
rebuilds every page rather than the one that changed. For a conference roster
of tens of people that is free. If the list ran to hundreds you would want
`notifyDataDelete(pageIndex)` for the page that lost someone - which is exactly
what `LazyDataSource.deleteData` already provides.

## Implementation steps

1. **Model a participant as a class, not an interface** - `ItemParam` has a
   `name` and an optional `image: Resource`, and `LazyForEach` needs a real
   constructor to type the callback parameter.
2. **Chunk the flat list** with a `for (let i = 0; i < list.length; i += 4)` and
   `slice(i, i + 4)`; the last chunk is naturally short and the wrapping `Flex`
   handles it without padding.
3. **Push an empty chunk at index 0** as the speaker-view sentinel, then splice
   the real chunks in behind it with `pushDataPositionArray(1, listArr)`.
4. **Clear before you push.** `resetDataArr` is called on every mutation and
   `dataArr.clear()` is what keeps it idempotent.
5. **Branch the `LazyForEach` body on `index === 0`** to render the speaker view
   or the grid; the item type is the same `ItemParam[]` in both arms.
6. **Key by identity, not by `JSON.stringify`** - the sample stringifies both
   the page array and the participant, which collides on duplicate names and is
   flagged by the ArkUI linter rule
   `@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator`.
7. **Lay each page out with `Flex({ wrap: FlexWrap.Wrap })`** over fixed
   164x280 vp tiles rather than a `Grid`; nothing here needs scroll or lazy
   cells inside a page.
8. **Delete the participant that was tapped, by index** - the sample always
   splices the last element (`HW-15-0082`) - and derive the footer count from
   `list.length` in one branch instead of two.

## Verified snippets

All snippets are from `SwiperConferencePage.zip`. Corrected forms are marked.

**The pagination — `entry/src/main/ets/pages/MeetingSwiper.ets`** (as shipped)

```typescript
class ItemParam {
  name: string = '';
  image?: Resource;

  constructor(name: string, image?: Resource) {
    this.name = name;
    this.image = image;
  }
}

@State private dataArr: LazyDataSource<ItemParam[]> = new LazyDataSource(); // 处理后的数据
@State list: ItemParam[] = []; // 初始数据

resetDataArr() {
  // 将数据按四个拆分
  let listArr: ItemParam[][] = [];
  for (let i = 0; i < this.list.length; i += 4) {
    listArr.push(this.list.slice(i, i + 4));
  }
  // 清空数据
  this.dataArr.clear();
  // 添加第一屏会议界面
  this.dataArr.pushData([]);
  // 添加与会人
  this.dataArr.pushDataPositionArray(1, listArr);
}
```

**The empty array pushed first is the whole trick.** `LazyForEach` iterates
`ItemParam[][]`, so page 0 receives `[]` and its render branch ignores the item
entirely - it draws the speaker view from `this.list` and `this.change`
instead. Without the sentinel you would need either a second component outside
the `Swiper` (which loses the swipe gesture between speaker view and grid) or an
index offset threaded through every access.

`clear()` before `pushData` is what makes `resetDataArr` safe to call
repeatedly; it splices the backing array in place rather than reassigning it, so
the `@Observed` `LazyDataSource` instance the `@State` field points at stays the
same object.

**The pager — same file** (as shipped, key generator flagged)

```typescript
Swiper(this.swiperController) {
  LazyForEach(this.dataArr, (item: ItemParam[], index: number) => {
    if (index === 0) {
      Stack() {
        // speaker view: one large avatar + one bordered thumbnail, swapped by this.change
      }
      .width('100%')
      .backgroundColor('#99000000')
      .height('100%')
      .align(Alignment.TopStart)
      .hitTestBehavior(HitTestMode.Transparent)
      .onClick(() => {
        this.change++;
      });
    } else {
      Flex({ wrap: FlexWrap.Wrap, alignContent: 0 }) {
        ForEach(item, (param: ItemParam) => {
          // one 164x280 vp tile
        }, (param: ItemParam) => JSON.stringify(param));   // linter: no stringify in a key generator
      }
      .padding({ left: 20 })
      .size({ width: '100%', height: '90%' })
      .backgroundColor('#99000000');
    }
  }, (item: ItemParam[]) => JSON.stringify(item));         // same problem, one level up
}
.indicator(DotIndicator.dot().bottom(40)) // 设置导航点样式
.loop(false)
.width('100%')
.height('85%');
```

**Three attributes carry the pager.** `loop(false)` is right for a bounded
roster - looping from the last page back to the speaker view would read as a
glitch. `DotIndicator.dot().bottom(40)` puts the page dots clear of the bottom
control bar. And `hitTestBehavior(HitTestMode.Transparent)` on the speaker-view
`Stack` lets the tap that swaps the speaker coexist with the `Swiper`'s own
horizontal drag: the stack handles the click but does not consume the gesture.

**Both key generators stringify their item**, which the ArkUI performance linter
explicitly forbids. Two participants with the same name and no image produce
identical JSON and therefore identical keys, so `LazyForEach` treats them as one
node; and on the outer level, any two pages holding equal participants collide.
It is also a per-frame allocation on the scroll path. Give `ItemParam` a stable
`id` and key on that.

**The tile tap — same file** (corrected, see `HW-15-0082`)

```typescript
ForEach(item, (param: ItemParam, tileIndex: number) => {
  Column() {
    // avatar + name + mic chip
  }
  .onClick(() => {
    // 点击按钮，删除该与会人，更新数据
    const globalIndex = (index - 1) * 4 + tileIndex;   // FIX: sample splices this.list.length - 1
    this.list.splice(globalIndex, 1);
    this.resetDataArr();
  })
  .borderRadius(4)
  .margin(2)
  .width('164vp')
  .height('280vp')
  .backgroundColor('#99858585');
}, (param: ItemParam) => param.name)
```

**The shipped handler is `this.list.splice(this.list.length - 1, 1)`** - it
ignores which tile was tapped and always removes the last participant. Tap
小明's tile and 小何 disappears. The fix needs the page index too, which
`LazyForEach` already supplies as its second callback parameter: page `index` 1
holds list positions 0..3, page 2 holds 4..7, so the global index is
`(index - 1) * 4 + tileIndex`. The `- 1` is the sentinel page's offset - the one
place the empty-chunk trick leaks into the rest of the code.

Because `resetDataArr()` rebuilds the whole data source, nothing else has to
change: the tiles reflow, a page that empties out disappears, and the dot
indicator loses a dot.

**The participant count — same file** (corrected, see `HW-15-0082`)

```typescript
Column() {
  Image($r('app.media.ic_attenders'))
    .width('24vp')
    .aspectRatio(1);
  Text('与会者' + '(' + this.list.length + ')')     // FIX: sample has a second branch
    .margin({ top: 5 })                             //      printing a hardcoded (1) when empty
    .fontSize(10)
    .fontColor(Color.White);
};
```

**The shipped code has two `if` branches over the same value**, one for
`list.length > 0` printing the real count and one for `list.length === 0`
printing a literal `1`. That second branch exists to paper over the speaker
view, which keeps drawing 小红(我) after the list has been emptied - see the
`this.list.length <= 1` arm of page 0, which renders her unconditionally. The
result is a page claiming one attendee while the roster is empty and the grid
pages are gone.

Decide which model is true. If 小红 is a participant, she belongs in `this.list`
and the count is just `list.length`. If she is the local user and separate from
the roster, the count should read `list.length + 1` in every branch, not only
the empty one.

## Permissions & config

**None.** The sample declares no `requestPermissions` and touches no system
resource beyond the window.

`deviceTypes` is `phone`, `tablet`, `2in1`. The layout does not honour that
range: tiles are a fixed `164vp x 280vp` and the speaker-view thumbnail is
positioned with `margin({ left: '274vp', bottom: '645vp' })`, absolute offsets
tuned for one phone screen. On a tablet the thumbnail lands in the wrong place
and the 2x2 grid leaves a large gap.

All strings - 小红的会议, 离开, 解除静音, 视频, 共享, 与会者, 更多 - are
literals in the source rather than string resources, so the sample cannot be
localised without editing the layout.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The participant list is eight hardcoded `ItemParam`s built in `aboutToAppear`;
  there is no network, no join/leave events and no persistence.
- The avatar rule is `param.name === '小明'` - one named participant gets the
  bundled photo and everyone else gets a generated initial. Any real roster
  needs an avatar field on the model instead.
- The initial is `param.name.charAt(1)`, the *second* character, which is right
  for two-character Chinese given names and wrong for anything else.
- `Flex` with `alignContent: 0` uses the numeric literal rather than
  `FlexAlign.Start`; it compiles but defeats the type check.
- The 离开 button, the mic chip and the whole bottom bar are decoration - none
  of them carries a handler.
- `ObservedArray<T> extends Array<T>` exists only to satisfy the
  `pushDataPositionArray` / `appendArrayData` signatures in `LazyDataSource`;
  the page passes plain arrays and never constructs one.

## Pitfalls

- **`HW-15-0082`** (B/low, probable): the tile `onClick` calls
  `this.list.splice(this.list.length - 1, 1)`, so tapping any participant
  deletes the last one in the roster instead of the one tapped - and repeated
  taps walk the list down to 小红, whom the speaker view still renders while the
  footer prints a hardcoded 与会者(1). Fix: splice by the tapped index computed
  from the page index, and make the empty state honest.
- Not a filed finding, but fix it if you copy the file: both `LazyForEach` key
  generators use `JSON.stringify` on their item, which the ArkUI linter rule
  `@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator` forbids -
  duplicate names collide into one node and every frame allocates a string.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `SwiperController`, `loop`, `indicator`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-swiper-components-indicator.md` - `DotIndicator` and its positioning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-swiper-components-indicator
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource`, the listener protocol, key generators
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `LazyForEach` parameters and the `index` callback argument
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` - why the two key generators here are wrong
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-flex-layout.md` - `FlexWrap` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-flex-layout
