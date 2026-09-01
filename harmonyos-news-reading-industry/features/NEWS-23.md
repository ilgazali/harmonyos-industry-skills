---
id: NEWS-23
title: Per-paragraph comments - an ImageSpan at each paragraph end opens its own half-modal sheet
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/23_novel_read_review.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/novel_read_review-0000002347649374
sample: huawei_industry_tree/11_news_reading/downloads/ParagraphComment.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [bindSheet, SheetSize, ImageSpan, ImageSpanAlignment, Span, LazyForEach, IDataSource, DataChangeListener, cachedCount, onScrollIndex, CustomDialogController, "@CustomDialog", "window.getLastWindow", keyboardHeightChange, "display.getDefaultDisplaySync", "@Provide", "@Consume", TransitionEffect]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-11-0025, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when **every row of a long list needs its own modal** - a
comment thread per paragraph, a note per photo, a detail sheet per line item.
The naive approach, one shared sheet plus a "which row am I showing" variable,
falls apart as soon as the sheet needs to animate from and back to the row.
This sample takes the other route: `bindSheet` is attached to the per-row
component, and the visibility state is a boolean *array* indexed by row.

The anchor is the interesting part. The comment button is not a separate
component next to the paragraph - it is an `ImageSpan` inside the paragraph's
own `Text`, so it flows on the last line after the final character and moves
with reflow. `Span` and `ImageSpan` are the only way to get an interactive
glyph inline with text; both live inside a `Text` and neither is a standalone
component.

It generalises to 边读边评 (comment while reading) in any reader, to inline
footnote markers, to a "translate this line" affordance - anything where the
control belongs to a run of text rather than to a container around it.

## Feature checklist

- A dark reading page with a fixed header: book title, a more-options glyph,
  and a chapter caption underneath.
- Paragraphs scroll in a `List` fed by `LazyForEach` over a lazy data source.
- Each paragraph ends with a small comment icon on the same text baseline.
- Tapping the icon raises a half-height modal sheet (`SheetSize.MEDIUM`) for
  that paragraph only.
- The sheet lists existing comments - avatar, nickname, body, a timestamp row
  with like / reply / more, and a 展开更多 (show more) affordance.
- The sheet's bottom bar is a non-focusable input; tapping it opens a real
  input dialog docked above the keyboard.
- Submitting a non-empty comment appends it to the sheet's list and clears the
  field.
- Dismissing the keyboard closes the input dialog.
- Dismissing the sheet resets that paragraph's flag, so the same paragraph can
  be reopened.

## Architecture

One `entry` module, five source files plus the ability. No model layer - the
"book" is three string resources and the comments are an array of strings.

```
entry/src/main/ets
├── common/Constants.ets            CONFIGURATION / STRINGCONFIGURATION - numeric and string literal maps
├── datasource/BasicDataSource.ets  IDataSource over string[]: add / push / insert / delete / pin
├── entryability/EntryAbility.ets   dark colour mode, white status-bar text
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                 @Entry: TopView above a Stack holding the reading view
└── views
    ├── DialogSheet.ets             the sheet body + the @CustomDialog input bar (362 lines)
    ├── TopView.ets                 the fixed header
    └── UpDownFlipView.ets          the List + LazyForEach + the per-paragraph ImageSpan and bindSheet
```

The documented tree matches the zip exactly.

**The design decision worth copying** is `@State isSheetsShow: boolean[]`, one
flag per paragraph, allocated up front:

```typescript
@State isSheetsShow: boolean[] = new Array(CONFIGURATION.PAGE_FLIP_PAGE_END).fill(false);
```

`bindSheet` reads a boolean and does not write it back. If a single boolean were
shared across rows, every row's sheet would be bound to the same value and all
of them would try to open at once. Indexing by the `LazyForEach` index gives
each anchor its own independent flag, and because the array holds primitives,
assigning one element is an observed change that repaints only that item.

The second choice worth noting is that the sheet content is a **`@Builder`
function, not a component instance**: `DialogBuilder()` is exported from
`DialogSheet.ets` and passed to `bindSheet` by call. That keeps the sheet's
state (`rightData`, the dialog controller, the measured screen width) scoped to
one presentation - a fresh `DialogSheet` is constructed each time a sheet opens,
which is also why the comments typed into one paragraph's sheet do not survive
its dismissal.

## Implementation steps

1. **Build each paragraph as one `Text` containing a `Span` and an
   `ImageSpan`**, so the icon lands after the final character rather than on its
   own line. Give the `ImageSpan` `verticalAlign: ImageSpanAlignment.CENTER` and
   an explicit width/height - spans are not laid out by the box model.
2. **Feed the list through `LazyForEach` over an `IDataSource`** and set
   `cachedCount` so neighbouring paragraphs are built before they scroll in.
3. **Key the `LazyForEach` on something stable** - here the resource name
   string, which is unique per paragraph. Never key on the index.
4. **Allocate the visibility array once**, sized to the paragraph count, and
   index it with the `LazyForEach` index.
5. **Attach `bindSheet` to the `ImageSpan`,** not to the `ListItem` or the
   `Text` - the sheet should be owned by the control that raises it.
6. **Reset the flag in `onDisappear`.** `bindSheet` never writes the boolean
   back, so a swipe-down dismissal would otherwise leave it `true` and the
   paragraph could not be reopened.
7. **Track the current page from `onScrollIndex`** rather than from a scroll
   offset.
8. **In the input dialog, register the keyboard listener in `aboutToAppear` and
   remove it in `aboutToDisappear`** (`HW-11-0025`). Keep the callback in a
   field so `off` can name it.

## Verified snippets

All snippets are from `ParagraphComment.zip`. Corrected forms are marked.

**The paragraph and its sheet - `entry/src/main/ets/views/UpDownFlipView.ets`** (as shipped)

```typescript
@Component
export struct UpDownFlipPage {
  @Link isMenuViewVisible: boolean;
  @Link isCommentVisible: boolean;
  @Link currentPageNum: number;
  private data: BasicDataSource = new BasicDataSource([]);
  @State isSheetsShow: boolean[] = new Array(CONFIGURATION.PAGE_FLIP_PAGE_END).fill(false);

  aboutToAppear(): void {
    for (let i = CONFIGURATION.PAGE_FLIP_PAGE_START; i <= CONFIGURATION.PAGE_FLIP_PAGE_END; i++) {
      this.data.pushItem(STRINGCONFIGURATION.PAGE_FLIP_RESOURCE + i.toString());  // 'app.string.pageflip_content1'...
    }
  }

  build() {
    Column() {
      List({ initialIndex: this.currentPageNum - CONFIGURATION.PAGE_FLIP_PAGE_COUNT }) {
        LazyForEach(this.data, (item: string, index: number) => {
          ListItem() {
            Text() {
              Span($r(item))
                .fontSize($r('app.integer.flippage_text_fontsize'))
                .lineHeight(33)
                .fontColor($r('app.color.pageflip_column_backgroundcolor'))
              ImageSpan($r('app.media.comment'))
                .height(25)
                .width(25)
                .margin({ top: 10 })
                .verticalAlign(ImageSpanAlignment.CENTER)
                .bindSheet(this.isSheetsShow[index], DialogBuilder(), {
                  detents: [SheetSize.MEDIUM],
                  onDisappear: () => {
                    this.isSheetsShow[index] = !this.isSheetsShow[index];   // put the flag back down
                  }
                })
                .onClick(() => {
                  this.isSheetsShow[index] = !this.isSheetsShow[index];
                })
            }
            .padding({ left: 24, right: 20 })
          }
        }, (item: string) => item)                                          // key: the resource name
      }
      .cachedCount(CONFIGURATION.PAGE_FLIP_CACHE_COUNT)
      .scrollBar(BarState.Off)
      .onScrollIndex((firstIndex: number) => {
        this.currentPageNum = firstIndex + CONFIGURATION.PAGE_FLIP_PAGE_COUNT;
      })
    }
    .backgroundColor(Color.Black)
  }
}
```

**Four details carry this.** `$r(item)` resolves a resource *name* held as a
string - the data source stores `'app.string.pageflip_content1'` and the
paragraph text lives in `string.json`, which is how a "book" ships without a
parser. `ImageSpan` must sit inside the `Text` alongside the `Span`; put it
outside and it becomes a block-level `Image` on the next line.
`verticalAlign: ImageSpanAlignment.CENTER` is what keeps the 25 vp glyph
centred on the text's 33 vp line box instead of hanging off the baseline. And
`onDisappear` is mandatory, not defensive: `bindSheet` treats the boolean as an
input only, so a drag-down dismissal leaves it `true` and the `onClick` toggle
would then *close* an already-closed sheet.

The toggle form (`= !this.isSheetsShow[index]`) works because `onDisappear`
only fires from the shown state, but `= false` in `onDisappear` and `= true` in
`onClick` is the form to copy - it cannot desynchronise.

**The keyboard listener - `entry/src/main/ets/views/DialogSheet.ets`** (corrected, see `HW-11-0025`)

```typescript
@CustomDialog
struct CommentDialog {
  @Consume rightData: Array<ResourceStr>;
  @Link textInputText: ResourceStr;
  controller?: CustomDialogController;
  @State marginBottom: number = 0;
  private win: window.Window | undefined = undefined;                 // FIX: keep the window
  private onKeyboardHeightChange = (height: number) => {              // FIX: keep the callback
    if (height) {
      this.marginBottom = -16;
    } else {
      this.controller?.close();          // keyboard dismissed -> close the input bar with it
    }
  };

  aboutToAppear(): void {
    window.getLastWindow(this.getUIContext().getHostContext()).then((win) => {
      this.win = win;
      win.on('keyboardHeightChange', this.onKeyboardHeightChange);
    });
  }

  aboutToDisappear(): void {
    this.win?.off('keyboardHeightChange', this.onKeyboardHeightChange);   // FIX: absent in the sample
  }
}
```

**`keyboardHeightChange` is registered on the window, not on the component,** so
its lifetime is the app's unless something removes it. The shipped code
subscribes in `aboutToAppear` with an inline arrow function and never calls
`off` - and because a fresh `DialogSheet`/`CommentDialog` pair is constructed
every time a paragraph sheet opens, the listeners accumulate one per open, each
still holding a reference to a destroyed component and each still calling
`this.controller?.close()`.

Two things are needed to fix it and both are missing in the sample: a field
holding the resolved `window.Window` (the handle only exists inside the
`then`), and a field holding the callback (`off` matches by reference; passing a
new arrow function removes nothing). Calling `win.off('keyboardHeightChange')`
with no callback removes every listener of that type on that window - acceptable
here, but a blunt instrument in an app that has others.

The height branch is worth reading too: a non-zero height means the keyboard
came up, so the bar shifts by `-16` vp; a zero height means it went away, and
the dialog closes itself rather than sitting over the page with no keyboard.

**Submitting a comment - same file** (as shipped)

```typescript
TextInput({
  text: $$this.textInputText,
  placeholder: $r('app.string.dialogfirst_test'),
  controller: this.textController
})
  .onSubmit(() => {
    if (this.textInputText.toString()?.trim().length !== 0) {
      this.rightData.push(this.textInputText);
    }
    this.textInputText = '';
  })
  .defaultFocus(true)
  .layoutWeight(1)
  .borderRadius(20);

Image($r('app.media.up'))
  .onClick(() => {
    if (this.textInputText.toString()?.trim().length !== 0) {
      this.rightData.push(this.textInputText);
      this.textInputText = '';
    }
    this.controller?.close();
  });
```

**`$$this.textInputText` is a two-way binding**, so the field and the state stay
in step without an `onChange`; `defaultFocus(true)` is what raises the keyboard
the moment the dialog appears, which in turn triggers the listener above. The
new comment is pushed into `rightData`, which the sheet publishes as `@Provide`
and this dialog consumes - the sheet's `ForEach(this.rightData, ...)` then
renders it under the two seeded example comments.

Note the two paths differ: `onSubmit` (the keyboard's send key) clears the field
unconditionally and leaves the dialog open; the arrow icon clears only on
success and always closes. And the `@Provide` in `DialogSheet` is declared
`Array<string>` while this `@Consume` declares `Array<ResourceStr>` - they
compile because `string` is assignable to `ResourceStr`, but the two
declarations should be one type.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`EntryAbility` forces dark mode in `onWindowStageCreate`
(`ConfigurationConstant.ColorMode.COLOR_MODE_DARK`) and sets
`statusBarContentColor: '#ffffff'`, which is what makes the white-on-black
reading surface consistent with the system bar. `Index.ets` then uses
`.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])`
to draw under both bars.

`deviceTypes` includes `wearable` alongside `phone`, `tablet` and `2in1` -
unusual for this industry, and worth checking against `SheetSize.MEDIUM`, which
has little meaning on a watch face.

## Constraints

- API Version 24 Release or later; DevEco Studio 6.1.1 Release or later.
  `compatibleSdkVersion` is `6.1.1(24)`, matching the document - note this is a
  newer baseline than the rest of the industry's samples, which target
  `6.0.0(20)`.
- **Comments are not persisted, and not even shared between openings.** Each
  sheet presentation constructs a new `DialogSheet` with an empty `rightData`,
  so a comment typed into a paragraph disappears when the sheet closes. There is
  no id on a comment, no author, and no per-paragraph store; the two comments
  always shown are the same static builder called twice.
- The "book" is three paragraphs (`PAGE_FLIP_PAGE_START` 1 to
  `PAGE_FLIP_PAGE_END` 3) read from string resources, and `isSheetsShow` is
  sized from the same constant. That coupling is implicit - a data source loaded
  from the network would need the array resized alongside it, or the index would
  run off the end.
- `TopView` positions itself with `@StorageProp('topRectHeight')`, but this
  sample's `EntryAbility` never writes that key, so the value is always the `0`
  default. The header sits under the status bar on a device with a notch.
- `Index.ets` defines `registerEmitter()` but never calls it, and
  `deleteEmitter()` calls `emitter.off(1)` while the only event id in the file
  is `2`. The back-from-fullscreen path is dead code.
- `Index` also declares nine `@State` fields (`isMenuViewVisible`,
  `isVisible`, `isDirectoryViewVisible`, `contentFontsize`, `contentColor`, …)
  of which only three reach a child. The file is a trimmed-down copy of a larger
  reader sample - see `NEWS-07` for the page-flip original.
- `display.getAllDisplays` fills `screenWidth` asynchronously and the sheet's
  widths are computed as `screenWidth - 76`, so the first frame lays out at
  `-76`. Prefer `display.getDefaultDisplaySync()` (already imported and used for
  `displayClass`) for a value available synchronously.

## Pitfalls

- **`HW-11-0025`** (B/low, confirmed): the `keyboardHeightChange` window
  listener registered in `CommentDialog.aboutToAppear` (`DialogSheet.ets:282`)
  is never unregistered - `aboutToDisappear` only nulls the dialog controller.
  Every comment sheet that is opened adds another window-level listener that
  outlives its component and keeps firing into destroyed state. Fix: hold the
  `window.Window` and the callback in fields and call
  `win.off('keyboardHeightChange', cb)` in `aboutToDisappear`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `SheetOptions`, `detents`, `onDisappear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-imagespan.md` - `ImageSpan` and `ImageSpanAlignment`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-imagespan
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `@CustomDialog` and `CustomDialogController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `cachedCount`, `onScrollIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and the key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `huawei_industry_tree/11_news_reading/docs/23_novel_read_review.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/novel_read_review-0000002347649374
- `NEWS-07` - the page-flip reader this sample's reading surface is trimmed from
