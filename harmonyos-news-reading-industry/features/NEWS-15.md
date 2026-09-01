---
id: NEWS-15
title: Typewriter effect - per-character Spans revealed by a 50 ms interval, with the scroller chasing the caret
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/15_typewriter_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/typewriter_effect-0000002322974369
sample: huawei_industry_tree/11_news_reading/downloads/TypeWriter.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [setInterval, clearInterval, setTimeout, "@Observed", "@ObjectLink", ForEach, Span, TextController, Scroller, scrollEdge, lineBreakStrategy, textIndent, "resourceManager.getRawFileContentSync", "util.TextDecoder", "@StorageProp", hilog]
permissions: []
min_api: 17
modules: [entry (entry)]
findings: [HW-11-0016, HW-11-0017, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when text has to **arrive rather than appear** - an AI answer
streaming into a reading pane, a chat bubble being "typed", a summary being
generated, a story unfolding in a game. The pattern: pre-split the text into
per-character models with a `visible` flag, then flip one flag every tick and
let ArkUI's observation system re-render the single character that changed.

The reason to model it this way, rather than re-assigning a growing substring
to one `Text`, is **per-character styling**. Once each character is its own
`Span` with its own colour and size, the reveal animation and the rich-text
rendering are the same mechanism: a highlighted keyword, an inline citation
marker, or a red emphasis run costs nothing extra. The sample demonstrates this
with a deliberately silly rule - every 5th character blue, every 7th red - but
the hook is where real styling would attach.

It generalises to any progressive reveal driven by a clock. What it does *not*
generalise to is long documents: this shape creates one component per
character, so a 1,000-character teaser is 1,000 components. Read the Constraints
before pointing it at an article.

## Feature checklist

- The page opens with a title, a section heading, and an empty card with a
  `0/0` counter.
- Tapping 生成 (generate) starts the reveal; the label then becomes 重新生成
  (regenerate).
- Characters appear one at a time at roughly 20 per second.
- Every 5th character is blue; every 7th (that is not also a 5th) is red at
  24 vp; the rest are black at 14 vp.
- The scroll view follows the writing edge, so the newest character stays
  visible.
- A counter under the card reads `<revealed>/<total>`.
- Tapping 重新生成 clears the card and replays from the beginning.
- The timer stops when the text is exhausted **and** when the page is left
  mid-animation.

## Architecture

One `entry` module, one page. There is no model layer, no data store and no
service - the whole feature is 195 lines in `MainPage.ets` plus a 3.4 KB text
file.

```
entry/src/main/ets
├── entryability/EntryAbility.ets      full screen, avoid areas -> AppStorage, light colour mode
├── entrybackupability/EntryBackupAbility.ets
└── pages/MainPage.ets                 SpanStyleStr + StyleStr models, MainPage (@Entry), Child (@ObjectLink)
entry/src/main/resources/rawfile
└── content.txt                        the article, ~1.1k characters of UTF-8
```

The documented tree matches the zip.

**The design decision worth copying** is the `Child` component. `MainPage`
holds `styleStrArr: StyleStr[]`, and the interval mutates *one field of one
element* - `this.styleStrArr[this.index].visible = true`. A plain `ForEach`
building `Span`s inline would not observe that: `@State` on an array tracks the
array reference and the top-level elements, not a property two levels down. By
declaring `StyleStr` `@Observed` and consuming each element through a child
component's `@ObjectLink`, the mutation is observed at the element that owns
it, and exactly one `Child` re-renders per tick:

```typescript
@Component
struct Child {
  @ObjectLink item: StyleStr;

  build() {
    if (this.item.visible) {
      Span(this.item.value)
        .fontSize(this.item.style.fontSize)
        .fontColor(this.item.style.fontColor)
    }
  }
}
```

That `@Observed`/`@ObjectLink` pairing is the transferable lesson, and it is
what makes a 20 Hz reveal affordable at all. Note the child renders *nothing*
when `visible` is false - the not-yet-typed characters are absent from the tree
rather than transparent, which is also why the text reflows as it types instead
of appearing in a pre-reserved box.

## Implementation steps

1. **Define the style model separately from the character.** `SpanStyleStr`
   (colour + size) is shared by reference between all characters that use it;
   `StyleStr` is value + style + `visible`. Mark **both** `@Observed`.
2. **Read the source text synchronously** with
   `context.resourceManager.getRawFileContentSync(fileName)`, then decode the
   `Uint8Array` with `util.TextDecoder.create('utf-8')`.
3. **Split into per-character models in one pass,** assigning the style there.
   Create the two accent `SpanStyleStr` instances *outside* the loop and reuse
   them - one object per style, not one per character.
4. **Render each element through a child component with `@ObjectLink`**, inside
   a `Text('', { controller })` container so the Spans lay out as one paragraph.
5. **Reveal on `setInterval`**: flip `visible`, advance the index, and call
   `scroller.scrollEdge(Edge.Bottom)` on the same tick.
6. **Clear the interval when the index reaches the length** - the `else` branch
   of the reveal.
7. **Clear it again in `aboutToDisappear`** (`HW-11-0016`). Step 6 alone leaks
   the timer whenever the user leaves mid-animation.
8. **On regenerate, cancel the running timer explicitly** before resetting the
   state, instead of relying on a `setTimeout` to outrace it (see Constraints).
9. **State the API level the project actually targets** (`HW-11-0017`): the
   sample's `compatibleSdkVersion` is `5.0.5(17)`, three versions below the
   document's stated minimum.

## Verified snippets

All snippets are from `TypeWriter.zip`. Corrected forms are marked.

**The character model — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
import util from '@ohos.util';

@Observed
class SpanStyleStr {
  fontColor: Color = Color.Black;
  fontSize: number = 14;
}

@Observed
class StyleStr {
  value: string;
  style: SpanStyleStr;
  visible: boolean;

  constructor(value: string, style: SpanStyleStr, visible: boolean) {
    this.value = value;
    this.style = style;
    this.visible = visible;
  }
}
```

**Two classes, not one, because the style is shared and the visibility is
not.** Every blue character points at the *same* `spanBlue` instance, so the
array of 1,100 `StyleStr` objects holds three style objects between them.
`@Observed` on `SpanStyleStr` means a later change to `spanBlue.fontSize` would
repaint every blue character at once - a cheap way to implement a global
font-size control over an already-typed document.

`visible` is a field of `StyleStr` rather than a separate parallel array
because `@ObjectLink` observes an object, and the observed object has to be the
thing that changes.

**Splitting the source text — same file** (as shipped)

```typescript
private readText(path: string) {
  let context = this.getUIContext().getHostContext() as Context;
  // 读取文件内容
  const value = context.resourceManager.getRawFileContentSync(path);
  this.textBuffer = this.uint8ArrayToString(value);

  let spanBlue = new SpanStyleStr();
  spanBlue.fontColor = Color.Blue;

  let spanRed = new SpanStyleStr();
  spanRed.fontColor = Color.Red;
  spanRed.fontSize = 24;

  // 每隔5个字显示一个蓝字，每隔7个字显示一个红字
  for (let i: number = 0; i < this.textBuffer.length; i++) {
    if (i % 5 === 0) {
      this.styleStrArr.push(new StyleStr(this.textBuffer.charAt(i), spanBlue, false));
    } else if (i % 7 === 0) {
      this.styleStrArr.push(new StyleStr(this.textBuffer.charAt(i), spanRed, false));
    } else {
      this.styleStrArr.push(new StyleStr(this.textBuffer.charAt(i), new SpanStyleStr(), false));
    }
  }
}

// Uint8Array转string
private uint8ArrayToString(u8Array: Uint8Array): string {
  let desString = '';
  if (u8Array && u8Array.length > 0) {
    let textDecode = util.TextDecoder.create('utf-8');
    desString = textDecode.decodeToString(u8Array);
  }
  return desString;
}
```

**`getRawFileContentSync` returns bytes, not text** - the decode step is
mandatory, and `util.TextDecoder.create('utf-8')` is the supported way to do
it. The source file here is Chinese prose, so a naive byte-to-char conversion
would produce mojibake; the guard `u8Array && u8Array.length > 0` covers the
empty-file case that would otherwise throw inside the decoder.

Two things to change when adapting this. The `else` branch allocates a fresh
`new SpanStyleStr()` for **every ordinary character** - roughly 900 identical
default-style objects in this sample - where hoisting one shared default
instance out of the loop, exactly as the two accent styles already are, costs
nothing. And `charAt(i)` splits by UTF-16 code unit: any emoji or other
astral-plane character in the source is torn into two halves that render as
two broken Spans. `Array.from(text)` iterates by code point and is the fix if
the content is user-supplied.

**The reveal loop — same file** (corrected, see `HW-11-0016`)

```typescript
intervalId = -1;

private startTypeWriter() {
  this.readText('content.txt');
  this.intervalId = setInterval(() => {
    if (this.index === this.styleStrArr.length && this.index) {
      this.isEnd = true;
    }
    if (this.index < this.styleStrArr.length) {
      this.styleStrArr[this.index].visible = true;
      this.index++;
      this.scroller.scrollEdge(Edge.Bottom);
    } else {
      clearInterval(this.intervalId); // 停止定时器
    }
    // 文本显示速度
  }, 50);
}

// FIX: absent in the sample - a page left mid-animation keeps the 20 Hz timer alive
aboutToDisappear(): void {
  if (this.intervalId !== -1) {
    clearInterval(this.intervalId);
    this.intervalId = -1;
  }
}
```

**The `else` branch is the only stop condition the sample ships**, and it can
only be reached while the component is still mounted and ticking. Leave the page
at character 300 of 1,100 and the interval keeps firing 20 times a second,
setting `visible` on `@Observed` objects belonging to a detached tree and
calling `scrollEdge` on a released `Scroller`, for another 40 seconds. That is
`HW-11-0016`; the document's own step-3 snippet teaches the same unguarded
shape, so a reader copying it inherits the leak.

`scrollEdge(Edge.Bottom)` on every tick is what keeps the caret in view. It is
also 20 scroll commands per second, each one issued whether or not the text has
grown past the fold - acceptable here, and the first thing to throttle if the
reveal is sped up.

`this.index === this.styleStrArr.length && this.index` reads oddly: the second
operand is a truthiness guard that suppresses `isEnd` when the array is empty
(0 === 0). It works, but `this.index > 0` says it. `isEnd` is then never read
anywhere in `build()` - it is dead state in the shipped sample, presumably the
hook where a "done" indicator was meant to go.

**Layout of the typing surface — same file** (as shipped)

```typescript
Scroll(this.scroller) {
  Text('', { controller: this.controller }) {
    ForEach(this.styleStrArr, (item: StyleStr) => {
      Child({ item: item });
    });
  }
  .width('100%')
  .fontSize(16)
  .lineBreakStrategy(LineBreakStrategy.HIGH_QUALITY)
  .textIndent('16vp')
  .textAlign(TextAlign.Start)
  .padding(10)
}
.height(500)
.align(Alignment.Top)
```

**`lineBreakStrategy(LineBreakStrategy.HIGH_QUALITY)` matters more here than in
static text.** The paragraph is re-laid-out on every single character, and a
greedy strategy makes earlier lines re-break as the text grows, producing a
visible jitter in already-typed content. `HIGH_QUALITY` balances the whole
paragraph and keeps the settled lines settled.

The `Text('', {...})` container form is the same idiom as `NEWS-13`: no string
content, all children. `textIndent('16vp')` gives the paragraph its first-line
indent without a leading space that would itself have to be typed out. The
fixed `.height(500)` is the one part not to copy - it pins the card to 500 vp
regardless of screen, where `layoutWeight(1)` would let the card fill whatever
the title and heading leave.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the text comes from
`rawfile` and everything else is local.

`EntryAbility.onCreate` pins light colour mode; the project ships a
`dark/element/color.json` that is therefore unreachable. Note the character
colours in `SpanStyleStr` are the `Color.Blue` / `Color.Red` / `Color.Black`
enums rather than resources, so they would not adapt to dark mode even if it
were enabled - move them to `$r('app.color....')` before shipping.

The user-visible strings (文档展示, 热爱学习, 生成, 重新生成) are all string
resources, but only `base` exists - there is no `en_US` or `zh_CN` variant.

## Constraints

- **The document and the sample disagree on the API level** (`HW-11-0017`). The
  zip's `build-profile.json5` declares `compatibleSdkVersion: "5.0.5(17)"`; the
  document's 约束与限制 demands API 20 / HarmonyOS 6.0.0. Nothing in the code
  needs API 20 - `setInterval`, `@ObjectLink`, `getRawFileContentSync` and
  `LineBreakStrategy` all predate it - so the constraint text is the side that
  is most likely wrong.
- **One component per character.** `content.txt` is 3,446 bytes of UTF-8,
  around 1,100 characters, and each becomes a `StyleStr` plus a `Child` plus a
  `Span`. `ForEach` (not `LazyForEach`) builds them all up front, so the whole
  tree exists from the first tick regardless of how much is visible. This shape
  does not survive a full article; for longer text, batch several characters
  into one Span per style run, or reveal by word.
- **Regenerate races the old timer.** The handler clears the array and index,
  then `setTimeout(..., 50)` before restarting - relying on the still-running
  interval to reach its `else` branch and cancel itself first. Both are 50 ms,
  so the order is not guaranteed: if the `setTimeout` wins, the new
  `intervalId` overwrites the old one, the old callback then clears the *new*
  timer, and the orphaned old timer keeps running. Call
  `clearInterval(this.intervalId)` in the click handler and drop the
  `setTimeout` entirely.
- `readText` is called from `startTypeWriter` on every generate, appending to
  `styleStrArr`; it only works on repeat because the click handler reassigns
  `this.styleStrArr = []` first. Clearing inside `readText` would be safer.
- `charAt` splits surrogate pairs (see the snippet notes) - fine for this
  bundled Chinese text, not for arbitrary input.
- The counter reads `this.index + '/' + this.textBuffer.length`, mixing the
  model index with the *string* length. They agree here only because both are
  UTF-16 counts of the same text.
- `isEnd` is assigned but never read.

## Pitfalls

- **`HW-11-0016`** (B/low, confirmed): the interval is cleared only in the
  "all characters shown" branch and the component defines no
  `aboutToDisappear`, so leaving the page mid-animation leaves a 50 ms timer
  mutating `@Observed` items of a destroyed tree and calling `scrollEdge` on a
  released scroller until the text length is exhausted. The document's step-3
  snippet teaches the same unguarded pattern. **Fix:** add
  `aboutToDisappear(): void { clearInterval(this.intervalId); }`.
- **`HW-11-0017`** (E/low, confirmed): the 约束与限制 section requires API
  Version 20 / HarmonyOS 6.0.0 while the zip's `build-profile.json5` targets
  `compatibleSdkVersion: "5.0.5(17)"` - the only such mismatch found in a sweep
  of 46 doc/zip pairs across four industries. Readers on 5.0.5 are told they
  cannot run a sample configured for exactly their version. **Fix:** align the
  constraint text with `5.0.5(17)`, or bump the sample to `6.0.0(20)`.

## References

- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval`, `clearInterval`, `setTimeout`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `Text` as a container, `lineBreakStrategy`, `textIndent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-span.md` - `Span` styling
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-span
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContentSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `NEWS-13` - the same `Text('')`-plus-Spans container idiom, used for highlight ranges instead of a reveal
- `huawei_industry_tree/11_news_reading/docs/15_typewriter_effect.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/typewriter_effect-0000002322974369
