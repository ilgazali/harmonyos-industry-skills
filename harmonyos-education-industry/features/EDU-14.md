---
id: EDU-14
title: Course reading progress - per-chapter scroll position and a course-wide completion bar
industry: 04_education
doc: huawei_industry_tree/04_education/docs/14_course_progress_display.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
sample: huawei_industry_tree/04_education/downloads/CourseProgressDisplay.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit", "@kit.LocalizationKit"]
apis: [Scroll, Scroller, currentOffset, onDidScroll, onReachEnd, onSizeChange, SizeOptions, Text, Span, TextController, textIndent, LineBreakStrategy, Progress, "@Provide", "@Consume", "@StorageProp", Navigation, NavPathStack, NavDestination, pushPathByName, getParamByName, "resourceManager.getRawFileContent", "util.TextDecoder", PromptAction]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0099, HW-04-0100, HW-04-0101, HW-04-0102, HW-04-0103, HW-04-0104, HW-04-0105, HW-04-0106, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for **reading progress over a text course**: a live "you are at
character N of M" readout inside a chapter, plus a course-level percentage that
advances as chapters are finished.

The technique is two scroll events answering two different questions:

- **`onDidScroll`** → *how far through this chapter am I right now?* Read
  `scroller.currentOffset().yOffset`, divide by the measured text height, scale
  to the character count.
- **`onReachEnd`** → *is this chapter done?* Add it to the read set and
  recompute the course percentage.

And one measurement that makes both possible: **`onSizeChange` on the `Text`**
gives you the rendered height of the whole chapter, which is the denominator
neither scroll event provides.

The state plumbing is `@Provide`/`@Consume` from the course page down into the
reader, so the reader writes progress that the contents list and the progress bar
read - no callbacks, no navigation results.

## Feature checklist

- A contents list of chapters, each with a read/unread tick.
- A course progress bar showing the percentage of chapters completed.
- Tapping a chapter opens a reading page; the last-opened chapter is remembered.
- The reading page shows a live `N / M` character counter that tracks scrolling.
- Reaching the bottom marks the chapter read, shows a 已完成 toast, and advances
  the course bar.

## Architecture

One `entry` module, two pages, two components.

```
entry/src/main/ets
├── components
│   ├── ContentsListItem.ets     one chapter row: tick + title, pushes ReadPage
│   └── Title.ets                shared title bar
├── constants/CommonConstants.ets  SCROLL_HEIGHT 640, READ_PAGE, sizes
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages                        (documented as page/ - HW-04-0105)
    ├── MainPage.ets             @Entry - owns all the @Provide state
    └── ReadPage.ets             the reader and both progress calculations
```

**`MainPage` is the state owner and everything else consumes.** Five
`@Provide`s, and the direction of travel is what makes the design work:

| Provided | Written by | Read by |
| --- | --- | --- |
| `pathStack: NavPathStack` | `MainPage` | both, to navigate |
| `contents: string[]` | `MainPage` | `ContentsListItem`, `ReadPage` |
| `readChapterArray: string[]` | **`ReadPage`** | `ContentsListItem` (tick), `MainPage` |
| `ProgressValue: number` | **`ReadPage`** | `MainPage` (the bar) |
| `lastedLearnIndex: number` | **`ReadPage`** | `MainPage` (continue button) |

The reader is a `NavDestination` several levels below the page that displays the
progress, and `@Provide`/`@Consume` is what lets it write upward without a
result callback or a shared singleton. That is the pattern to take from this
card.

**Three numbers drive the in-chapter counter:**

- `textHeight` - the rendered height of the whole chapter, from `onSizeChange`
- `SCROLL_HEIGHT` - the viewport height, a constant (`HW-04-0100`)
- `textBuffer.length` - the character count, used only as the scale

`rollingProgress = floor((yOffset + viewport) / textHeight * characterCount)`.
It maps *pixels read* onto *characters*, which is an approximation - it assumes
uniform text density - but it is the only way to get a character counter without
laying out the text yourself.

## Implementation steps

1. **Declare the shared state as `@Provide` on the course page** and `@Consume`
   it in the reader and the list rows. Name the keys explicitly
   (`@Provide('contents')`) so a rename cannot silently break the binding.
2. **Load the chapter text** with `resourceManager.getRawFileContent` and decode
   with `util.TextDecoder`. **Check the error argument** - the sample discards it
   (`HW-04-0102`).
3. **Size the reading pane responsively** and capture its measured height, rather
   than hard-coding 640 vp and dividing by the constant (`HW-04-0100`).
4. **Measure the text in `onSizeChange`.** `newValue.height` is the full rendered
   height of the `Text`, including the part scrolled out of view - which is
   exactly the denominator you need and which no scroll event reports.
5. **Compute the in-chapter position in `onDidScroll`** from
   `scroller.currentOffset().yOffset` plus the viewport height, so the counter
   reflects the *last visible* character rather than the first.
6. **Mark the chapter complete in `onReachEnd`**, guarding the *whole* handler
   against repeats, not just the array push (`HW-04-0099`).
7. **Identify chapters by index, not by title** (`HW-04-0104`).
8. **Read the navigation parameter from `NavDestinationContext.pathInfo.param`**,
   not `getParamByName` (`HW-04-0103`).

## Verified snippets

All snippets are from `CourseProgressDisplay.zip`. Corrected forms are marked.

**The reader's consumed state — `entry/src/main/ets/pages/ReadPage.ets`** (as shipped)

```typescript
@Component
struct ReadPage {
  @StorageProp('topRectHeight') topRectHeight: number = 0
  @Consume('lastedLearnIndex') lastedLearnIndex: number
  @Consume('pathStack') pathStack: NavPathStack
  @Consume('readChapterArray') readChapterArray: string[]
  @Consume('contents') contents: string[]
  @Consume('ProgressValue') progressValue: number
  @State rollingProgress: number = 0      // characters read in this chapter
  @State textBuffer: string = ''          // the raw chapter text
  @State textHeight: number = 1           // rendered height, measured
  @State currentText: string = ''         // the text actually displayed
  private scroller: Scroller = new Scroller()
}
```

**`@Consume` is two-way**, which is the point: `readChapterArray`,
`progressValue` and `lastedLearnIndex` are written here and observed by the
course page. `textHeight` starts at **1**, not 0 - a deliberate guard, since it
is the denominator of both ratios and is read before `onSizeChange` first fires.

**Measuring the chapter — same file** (corrected, see `HW-04-0100`, `HW-04-0101`)

```typescript
Scroll(this.scroller) {
  // FIX: sample writes Text('', { controller: this.controller }) with a controller
  //      that is never used, and an empty string value the Span child overrides
  Text() {
    Span(this.currentText.replace(/\n/g, '\n       '))   // fake the paragraph indent
  }
    .align(Alignment.Top)
    .width(CommonConstants.FULL_WIDTH)
    .fontSize(CommonConstants.NORMAL_FONTSIZE)
    .textIndent(CommonConstants.TEXT_INDENT)
    .lineBreakStrategy(LineBreakStrategy.HIGH_QUALITY)
    .onSizeChange((oldValue: SizeOptions, newValue: SizeOptions) => {
      this.textHeight = newValue.height as number       // the FULL rendered height
      const ratio = Math.min(this.viewportHeight / this.textHeight, 1)
      this.rollingProgress = Math.floor(ratio * this.textBuffer.length)
    })
}
  .height(CommonConstants.SCROLL_HEIGHT)     // FIX: 640 vp fixed - measure this instead
  .align(Alignment.Top)
  .scrollBar(BarState.Off)
  .onDidScroll(() => {
    const yOffset = this.scroller.currentOffset().yOffset
    const ratio = Math.min((yOffset + this.viewportHeight) / this.textHeight, 1)
    this.rollingProgress = Math.floor(ratio * this.textBuffer.length)
  })
```

**`onSizeChange` on the child, not the `Scroll`, is the whole trick.** The
`Scroll` is a fixed-height viewport, so its size tells you nothing; the `Text`
inside it lays out to its full content height, and `onSizeChange` reports that.
Without it there is no denominator - `currentOffset()` gives you a position with
nothing to divide by.

**`yOffset + viewport`, not `yOffset`.** The counter is meant to say how much has
been *read*, so it must measure to the bottom of the visible window. Using
`yOffset` alone would report 0 on a chapter that fits entirely on screen.

**The `Math.max(ratio, 0)` in the shipped code is a dead guard** - both numerator
and denominator are non-negative, so it can never fire. The bound that actually
needs guarding is the upper one; `Scroll`'s default `EdgeEffect.None` is what
keeps `ratio` at exactly 1 today, and setting a spring effect would push the
counter past the chapter length.

**`replace(/\n/g, '\n       ')` is a workaround** for `textIndent` applying only
to the first line of a `Span`; seven spaces after each newline fake the indent on
subsequent paragraphs.

**Marking the chapter complete — same file** (corrected, see `HW-04-0099`, `HW-04-0104`)

```typescript
.onReachEnd(() => {
  // FIX: the sample toasts unconditionally and guards only the array push,
  //      so scrolling up and back down re-announces completion every time
  if (this.readChapterArray.includes(this.chapterIndex)) {
    return
  }
  this.readChapterArray.push(this.chapterIndex)          // FIX: sample pushes the TITLE
  this.progressValue = Math.floor((this.readChapterArray.length / this.contents.length) * 100)
  this.promptAction.showToast({
    message: $r('app.string.finished'),
    textColor: $r('app.color.toast_text_color')
  })
})
```

**`onReachEnd` means "is at the end position", not "just scrolled to the end".**
A chapter whose rendered text is shorter than the 640 vp pane is at its end
position from the moment it lays out, so it is marked read, toasted and counted
without the student scrolling at all. Requiring `rollingProgress` to have reached
`textBuffer.length`, or simply requiring a prior scroll event, closes that.

**Identify chapters by index.** `contents` is a list of titles, and course
outlines repeat titles - 练习, 小结, 复习 recur per unit. Two chapters with the
same title are one entry in the read set, so both tick together and the
percentage never reaches 100.

**Loading the chapter — same file** (corrected, see `HW-04-0102`)

```typescript
private readText(path: string) {
  const context = this.getUIContext().getHostContext() as Context
  context.resourceManager.getRawFileContent(path, (err, value) => {
    if (err) {                                   // FIX: sample names this _err and ignores it
      hilog.error(0x0000, 'ReadPage', `getRawFileContent failed: ${err.code} ${err.message}`)
      this.loadFailed = true
      return
    }
    this.textBuffer = this.uint8ArrayToString(value)
    this.currentText = this.textBuffer
  })
}

private uint8ArrayToString(u8Array: Uint8Array): string {
  if (!u8Array || u8Array.length === 0) {
    return ''
  }
  return util.TextDecoder.create('utf-8').decodeToString(u8Array)
}
```

**`getRawFileContent`'s callback reports failure in its first argument** - the
call itself does not throw. Discarding it turns a missing `content.txt` into a
blank pane with a `0/0` counter, which `onReachEnd` then immediately marks
complete.

**Navigating in — `entry/src/main/ets/components/ContentsListItem.ets`** (as shipped)

```typescript
Image(this.readChapterArray.includes(this.contents[this.itemIndex])
        ? $r('app.media.selected') : $r('app.media.unselected'))
// ...
.onClick(() => {
  this.pathStack.pushPathByName(CommonConstants.READ_PAGE, this.itemIndex)
})
```

and the reader's side (corrected, see `HW-04-0103`):

```typescript
// FIX: the sample does
//   this.params = this.pathStack.getParamByName(CommonConstants.READ_PAGE) as number[]
//   this.chapterIndex = this.params[0]
// getParamByName returns Array<unknown> - one entry per ReadPage on the stack.
.onReady((context: NavDestinationContext) => {
  this.chapterIndex = context.pathInfo.param as number
  this.lastedLearnIndex = this.chapterIndex
  this.readText('content.txt')
})
```

Writing `lastedLearnIndex` here is what makes the course page's "continue where
you left off" button work - the reader publishes upward through `@Consume` the
moment it opens.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block; the
chapter text is a bundled rawfile (`entry/src/main/resources/rawfile/content.txt`).

Routing is by `route_map.json`, with `readPageBuilder` exported from
`ReadPage.ets`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The reading pane is a fixed 640 vp** and the same constant is the numerator
  of both progress ratios (`HW-04-0100`), so layout and arithmetic cannot be
  changed independently.
- **Every chapter renders the same file.** `readText('content.txt')` ignores
  `chapterIndex`; the contents list, the tick marks and the percentage are real,
  the per-chapter content is not.
- The character counter is a pixel-to-character approximation that assumes
  uniform text density; a chapter with an image or a long code block will report
  a position that does not match the visible text.
- Progress is in-memory `@Provide` state only - nothing is persisted, so the read
  set and the percentage reset on every launch. Compare `EDU-13`, which persists
  the equivalent state in `preferences`.
- `textIndent` applies to the first line only, hence the newline-replacement
  workaround for paragraph indents.

## Pitfalls

- **`HW-04-0099` — `onReachEnd` toasts every time and completes a short chapter
  on sight.** The duplicate guard covers the array push but not the toast, and a
  chapter shorter than the viewport is at the end position before it is read.
- **`HW-04-0100` — `SCROLL_HEIGHT = 640` is both the pane height and the
  progress numerator,** so the layout is not responsive and cannot be made
  responsive without changing what the counter means.
- **`HW-04-0101` — a `TextController` is constructed, bound and never used,** on
  a `Text` that is also given an inert empty-string value alongside its `Span`.
- **`HW-04-0102` — `getRawFileContent`'s error argument is discarded,** so a
  failed read is a silent blank chapter that immediately counts as complete.
- **`HW-04-0103` — `getParamByName(...) as number[]` returns every matching
  page's parameter,** and the cast misdescribes the `Array<unknown>` return. Same
  defect as `EDU-09`'s `HW-04-0058`.
- **`HW-04-0104` — the read set stores chapter titles,** so repeated titles
  collide and the course can never reach 100 %.
- **`HW-04-0105` — the documented directory is `page/`; the zip ships `pages/`,**
  which is what `main_pages.json` and the route map address.
- **`HW-04-0106` — an unused `@State index11` is left in the reader.**
- **`Math.max(ratio, 0)` guards the wrong side.** Both operands are non-negative;
  it is the upper bound that needs clamping if the edge effect is ever enabled.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-scrollable-common.md` - `onDidScroll`, `onReachEnd` and what "reaches the end position" means
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scrollable-common
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroller.currentOffset`, `edgeEffect` defaulting to `EdgeEffect.None`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-size-change-event.md` - `onSizeChange` and `SizeOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-size-change-event
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - two-way binding from a descendant back to an ancestor
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `Text` with `Span` children, `textIndent`, `TextController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContent` and its error-first callback
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `getParamByName` versus `NavDestinationContext.pathInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `EDU-09` - the same `getParamByName` misuse
- `EDU-13` - the same kind of per-item progress state, persisted in `preferences` rather than held in memory
