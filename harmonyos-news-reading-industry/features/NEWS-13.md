---
id: NEWS-13
title: Text highlight marking - a mark list of ranges rebuilt into Spans, persisted in a KV store
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/13_text_marker_ability.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marker_ability-0000002283796046
sample: huawei_industry_tree/11_news_reading/downloads/TextMarkinAbility.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Text, Span, TextController, textBackgroundStyle, onTextSelectionChange, copyOption, ForEach, distributedKVStore, "distributedKVStore.createKVManager", getKVStore, SingleKVStore, hilog, "window.getWindowAvoidArea", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0014, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when a reading surface has to carry **user-owned annotations
over immutable text** - highlighter marks in a news article, a novel, a
journal paper or a contract. The pattern: keep the text as one unchanging
string, keep the annotations as a sorted list of `{start, end, color}`
ranges, and render the pair by slicing the string into `Span`s at the range
boundaries.

The load-bearing idea is that **the marks are the model and the Spans are a
projection**. Nothing in the view holds state; changing a mark means
rewriting the range list, and the `ForEach` inside the `Text` re-slices the
original string on the next frame. That makes persistence trivial - the whole
annotation layer is one `JSON.stringify` of an array of three-field objects -
and it makes undo, export, sync and server-side merge equally easy to bolt on
later.

The technique generalises to anything that decorates a range of text without
editing it: comment anchors, search-hit highlighting, spelling squiggles,
speaker colouring in a transcript. What does **not** generalise is this
sample's merge algorithm, which is wrong in a way that quietly destroys the
user's earlier marks - read `HW-11-0014` before copying `markHighlight`.

## Feature checklist

- The article renders as one selectable `Text` with `copyOption(CopyOptions.InApp)`,
  so the system selection handles appear on a long press.
- Selecting a range and tapping 标记 (mark) paints that exact range yellow.
- Selecting a range and tapping 清除 (clear) removes the highlight from that
  exact range and leaves the rest of any overlapping mark intact.
- Marking a range that already carries the same colour does not split the
  visual highlight into visible seams.
- Marks survive leaving and re-entering the page: the list is written to a
  `SingleKVStore` on `aboutToDisappear` and read back on `aboutToAppear`.
- The page draws under the status bar and navigation indicator, with padding
  derived from the avoid areas.

## Architecture

One `entry` module, four ArkTS files. The whole feature is a page plus a
two-file data store; there is no view layer and no HAR.

```
entry/src/main/ets
├── datastore/DataModel.ets          the HighlightMark interface + TEXT_DATA, the article as a const string
├── datastore/DataStoreManager.ets   StoreManager singleton over distributedKVStore (init / put / get)
├── entryability/EntryAbility.ets    full screen, avoid areas -> AppStorage, forces light colour mode
├── entrybackupability/EntryBackupAbility.ets
└── pages/TextPage.ets               @Entry - the mark list, markHighlight(), and the three builders
```

The documented tree matches the zip exactly; `TextPage.ets` is 210 lines and
`markHighlight` is 60 of them.

**The design decision worth copying** is `HighlightMark` covering the *whole*
document, not only the marked parts. On a first launch with an empty store the
page seeds

```typescript
this.markList = [{ start: 0, end: this.originalText.length, color: Color.Transparent }];
```

so the invariant is "the mark list tiles the text end to end, with no gaps and
no overlaps". The renderer then needs no fallback branch for unmarked text - it
walks the list, and transparent-backgrounded Spans are the unmarked runs. Every
edit is a re-tiling of that partition, and the render loop stays four lines
long. The cost is that the merge function must maintain the invariant on both
sides of every edit, which is exactly where the sample fails.

## Implementation steps

1. **Model the annotation layer as a tiling.** `{start, end, color}`, `start`
   inclusive and `end` exclusive, seeded with one transparent range covering
   the entire string.
2. **Render with `ForEach` over the marks inside a `Text`,** slicing
   `originalText.substring(mark.start, mark.end)` into a `Span` and applying
   `textBackgroundStyle({ color: mark.color, radius: 0 })`.
3. **Capture the selection with `onTextSelectionChange`** on the `Text`, into
   plain (non-`@State`) fields - the selection is an input to the next command,
   not something the view renders.
4. **Enable `copyOption(CopyOptions.InApp)`.** Without it the text is not
   selectable at all and `onTextSelectionChange` never fires.
5. **Trim overlapping marks at the selection boundaries; never absorb them
   into the new range** (`HW-11-0014`). An existing mark that starts before the
   selection contributes a left remnant with **its own** colour; one that ends
   after the selection contributes a right remnant, likewise with its own
   colour.
6. **Reassemble as `leftMarks.concat(currentSelection, rightMarks)`** - because
   the input list is ordered and disjoint, the output is too, and the tiling
   invariant holds without a sort.
7. **Persist the list as JSON** through the KV store on `aboutToDisappear`,
   read it back on `aboutToAppear`, and fall back to the seed range when the
   stored value is empty.
8. **Call `initKVManager(context)` once from the ability's `onCreate`.** The
   shipped sample never calls it, so `kvStore` stays `undefined` and every
   `put`/`get` takes the "KVStore is not initialized" branch (see Constraints).

## Verified snippets

All snippets are from `TextMarkinAbility.zip`. Corrected forms are marked.

**The renderer — `entry/src/main/ets/pages/TextPage.ets`** (as shipped)

```typescript
@Builder
TextViewer() {
  Scroll() {
    Text('', { controller: this.controller }) {
      ForEach(this.markList, (mark: HighlightMark) => {
        Span(this.originalText.substring(mark.start, mark.end))
          .textBackgroundStyle({ color: mark.color, radius: 0 })
          .lineHeight(this.textLineHeight)
          .fontSize(this.textFontSize);
      }, (mark: HighlightMark) => JSON.stringify(mark));
    }
    .padding(24)
    .copyOption(CopyOptions.InApp)
    .onTextSelectionChange((selectionStart: number, selectionEnd: number) => {
      this.selectionStart = selectionStart;
      this.selectionEnd = selectionEnd;
    });
  }
  .scrollBar(BarState.Off)
  .width('90%')
  .layoutWeight(1);
}
```

**Three details carry this.** The `Text('')` with an empty first argument and
a child block is the container form - the string content comes entirely from
the `Span`s, so the mark list is the single source of truth for what is on
screen. `textBackgroundStyle` with `radius: 0` is what makes adjacent Spans of
the same colour read as one continuous highlighter stroke; a non-zero radius
would show a seam at every range boundary, which is why step 5's trimming must
also merge same-colour neighbours rather than leaving them split. And
`copyOption` is not decoration: selection is disabled by default on `Text`, and
without `CopyOptions.InApp` the whole feature is inert.

The key generator `JSON.stringify(mark)` is safe **only** because the tiling
invariant guarantees no two marks share a `start`. It is a stringified-object
key, which the ArkTS lint rule `hp-arkui-no-stringify-lazyforeach-key` warns
about for `LazyForEach`; on a list this small the cost is a full re-slice per
edit, which is what the sample wants anyway.

**The mark/clear command — same file** (corrected, see `HW-11-0014`)

```typescript
markHighlight(highlight: boolean) {
  if (this.selectionStart !== -1 && this.selectionEnd !== -1) {
    let leftMarks: HighlightMark[] = [];   // 区域左侧的文本样式数据
    let rightMarks: HighlightMark[] = [];  // 区域右侧的文本样式数据
    let currentSelection: HighlightMark = {
      start: this.selectionStart,
      end: this.selectionEnd,
      color: highlight ? $r('app.color.mark_color') : Color.Transparent
    };

    this.markList.forEach((mark) => {
      // the part of an existing mark that lies BEFORE the selection
      if (mark.start < currentSelection.start) {
        if (mark.end < currentSelection.start) {
          leftMarks.push({ start: mark.start, end: mark.end, color: mark.color });
        } else {
          // FIX: shipped code does `currentSelection.start = mark.start` when the
          // colours match, swallowing the untouched left half into the new range
          leftMarks.push({ start: mark.start, end: currentSelection.start, color: mark.color });
        }
      }
      // the part of an existing mark that lies AFTER the selection
      if (mark.end > currentSelection.end) {
        if (mark.start > currentSelection.end) {
          rightMarks.push({ start: mark.start, end: mark.end, color: mark.color });
        } else {
          // FIX: shipped code does `currentSelection.end = mark.end` when the
          // colours match, swallowing the untouched right half
          rightMarks.push({ start: currentSelection.end, end: mark.end, color: mark.color });
        }
      }
    });
    this.markList = leftMarks.concat(currentSelection, rightMarks);
  }
}
```

**Read the two `if`s as independent questions, not as an if/else.** A mark that
strictly contains the selection satisfies both, so it contributes a left
remnant *and* a right remnant and the selection is punched out of its middle -
which is precisely how "clear a few words inside a long highlight" is supposed
to behave. The shipped code reaches the same two branches but, when the colours
happen to agree, extends `currentSelection` to the union instead of emitting a
remnant. Two user-visible consequences follow: clearing three words inside a
long yellow mark sets the *whole* union transparent (the entire highlight
disappears), and re-marking part of an existing yellow mark silently re-anchors
the mark's boundaries.

The `else` branches above deliberately drop the sample's same-colour merge. The
merge is not needed for correctness - two adjacent same-colour ranges render as
one stroke because `radius` is 0 - and it is the vehicle for the defect. If you
want the list compacted, do it as a separate pass over the assembled result,
where "adjacent and equal colour" can be checked without touching the selection.

Note that `color` is typed `ResourceColor`, so a highlight stores the
*resource* `$r('app.color.mark_color')` and a clear stores the enum
`Color.Transparent`. The equality test the sample performs on those two
representations (`mark.color === currentSelection.color`) compares a resource
object against a resource object by reference - another reason not to build
behaviour on it.

**The KV store wrapper — `entry/src/main/ets/datastore/DataStoreManager.ets`** (as shipped)

```typescript
initKVManager(context: Context): void {
  if (this.kvManager === undefined) {
    const KV_MANAGER_CONFIG: distributedKVStore.KVManagerConfig = {
      context: context,
      bundleName: 'com.textmarkviewer.demo'
    };
    try {
      this.kvManager = distributedKVStore.createKVManager(KV_MANAGER_CONFIG);
      const OPTIONS: distributedKVStore.Options = {
        createIfMissing: true,
        encrypt: false,
        backup: false,
        autoSync: false,
        kvStoreType: distributedKVStore.KVStoreType.SINGLE_VERSION,
        securityLevel: distributedKVStore.SecurityLevel.S1
      };
      this.kvManager.getKVStore<distributedKVStore.SingleKVStore>('storeId', OPTIONS,
        (err, store: distributedKVStore.SingleKVStore) => {
          if (err) { return; }
          this.kvStore = store;
          // 请确保获取到键值数据库实例后，再进行相关数据操作
        });
    } catch (e) {
      let error = e as BusinessError;
      hilog.error(0xFFFF, TAG, `An unexpected error occurred. Code:${error.code}`);
    }
  }
}
```

**`getKVStore` is asynchronous and the store handle only exists inside its
callback** - which the sample's own trailing comment says out loud. Everything
that follows in `putValue`/`getValue` is guarded by `if (this.kvStore)` for
that reason, and silently logs "KVStore is not initialized" when the guard
fails. That guard is what turns the missing `initKVManager` call into a silent
no-op instead of a crash.

The options are worth understanding as a set: `SINGLE_VERSION` is the
non-distributed store type (no conflict resolution, one writer), `autoSync:
false` keeps the marks on this device, and `securityLevel: S1` is the lowest
tier - appropriate for reading positions, not for anything the user would
consider private. For a purely local annotation list, `preferences` would be
the lighter choice; `distributedKVStore` earns its keep only once `autoSync` is
turned on and the marks are meant to follow the user across devices.

**Seed and persist — `entry/src/main/ets/pages/TextPage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  // 读取数据库，如果不为空，则载入数据
  STORE_MANAGER.getValue('markText').then((str) => {
    if (str) {
      this.markList = JSON.parse(str) as HighlightMark[];
    } else {
      this.markList = [{
        start: 0,
        end: this.originalText.length,
        color: Color.Transparent
      }];
    }
  });
}

aboutToDisappear(): void {
  // 保存高亮标记数据
  STORE_MANAGER.putValue('markText', JSON.stringify(this.markList));
}
```

Persisting only on `aboutToDisappear` is the pragmatic choice for a KV store -
one write per visit instead of one per highlighter stroke - but it is also the
fragile one: `aboutToDisappear` does not run if the process is killed from the
recents list. If the marks matter, write through on every `markHighlight` and
accept the extra `put`s, or write on `onBackground` in the ability.

The `else` branch is the tiling seed described above. It is what lets the
renderer assume "the list always covers the text".

## Permissions & config

**None.** The sample declares no `requestPermissions`; `distributedKVStore`
needs a permission only for cross-device sync (`ohos.permission.DISTRIBUTED_DATASYNC`),
and `autoSync` is `false` here.

`module.json5` declares `phone`, `tablet` and `2in1`. `EntryAbility.onCreate`
pins the app to light mode with
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)` - the project
ships a `dark/element/color.json`, but it is unreachable while that call
stands. The highlighter colour `#88FFF200` is a 53%-alpha yellow, chosen so
black text stays legible through it; if you enable dark mode you need a second
value there.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` in the zip is
  `6.0.0(20)`, matching the document.
- **`initKVManager` is never called anywhere in the zip** (not a separately
  filed finding, but verified by grep across the sample: the only occurrence is
  its own declaration). `kvStore` therefore stays `undefined`, both `putValue`
  and `getValue` take the "not initialized" branch, and marks do not actually
  survive a restart. Call it from `EntryAbility.onCreate` with `this.context`
  before loading the page.
- `TEXT_DATA` is a hardcoded const string in `DataModel.ets`. Because marks are
  integer offsets into it, any change to the article invalidates every stored
  mark. Real content needs the mark list keyed by article id and versioned
  against the text.
- `onTextSelectionChange` reports offsets into the rendered `Text`, which here
  equals the original string only because the Spans concatenate to exactly
  `originalText`. Insert any decoration Span and the offsets stop lining up.
- The layout uses `@StorageProp('topRectHeight')`/`('bottomRectHeight')` fed
  by the ability. The `avoidAreaChange` listener registered in
  `onWindowStageCreate` is never released in `onWindowStageDestroy` - the same
  boilerplate omission that recurs across this industry's samples.
- The toolbar has a `toolbar_color` string resource ("Color") with no
  corresponding control: multi-colour marking is modelled in `HighlightMark`
  but not exposed in the UI.

## Pitfalls

- **`HW-11-0014`** (B/low, probable): the merge algorithm changes text outside
  the user's selection - any existing mark that overlaps the selection is
  absorbed by extending `currentSelection` to the union, so the merged range
  takes the new colour. Highlighting over half of an existing mark repaints the
  untouched half, and clearing a few words inside a long highlight deletes the
  whole highlight. **Fix:** when an overlap is found, push the non-overlapping
  sub-ranges of the existing mark, with its original colour, into
  `leftMarks`/`rightMarks` instead of extending `currentSelection` - the
  corrected `markHighlight` above.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `Text` as a container, `copyOption`, `onTextSelectionChange`, `TextController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-span.md` - `Span` and `textBackgroundStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-span
- `documentation/harmonyos-references/02_application-framework/js-apis-distributedkvstore.md` - `createKVManager`, `getKVStore`, `Options`, `SingleKVStore.put`/`get`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-distributedkvstore
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-kv-store.md` - when a KV store is the right store
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-kv-store
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getWindowAvoidArea` and `avoidAreaChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `huawei_industry_tree/11_news_reading/docs/13_text_marker_ability.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marker_ability-0000002283796046
