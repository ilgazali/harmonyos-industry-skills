---
id: NEWS-17
title: Regex keyword highlight - escape the query, split the article, render matched runs as coloured Spans
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/17_regular_highlight.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/regular_highlight-0000002328562941
sample: huawei_industry_tree/11_news_reading/downloads/RegularHighlightDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [RegExp, "RegExp.exec", "String.replace", TextInput, cancelButton, Text, Span, ForEach, "resourceManager.getStringByNameSync", "UIContext.getHostContext"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0019, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when you have **one long body of text and a live query box over
it**, and every occurrence of the query has to change colour as the user types.
Article search, in-page find, chat search, a glossary term picked out of a
paragraph - all the same shape.

The pattern has three parts and each one is a separate decision: escape the
user's input before it reaches `RegExp`, turn the match positions into a list of
`{ text, isMatch }` runs, and render that list as `Span`s inside a single `Text`.
Only the third part is ArkUI-specific. The first two are the ones people get
wrong.

It generalises past highlighting. Any "annotate substrings of a paragraph"
feature - a spellcheck squiggle, a mention chip, a redaction bar - is the same
run-splitting problem with a different `Span` decoration at the end. What you
must not do is build the highlight out of concatenated `Text` components in a
`Row`: that breaks line wrapping, because the layout engine can no longer break
a line inside your highlighted word. Composing `Span`s inside one `Text` keeps
the whole paragraph on one text layout pass.

## Feature checklist

- The article body loads from a string resource on entry and renders as one
  justified paragraph.
- A rounded `TextInput` above the article accepts the query, with a permanently
  visible clear button.
- Every occurrence of the query in the article turns blue and bold as soon as
  the character is typed - no submit button.
- Clearing the query returns the article to plain text with no flicker of a
  half-highlighted state.
- Typing regex metacharacters (`.`, `*`, `(`, `[`, `\`) matches them literally
  instead of throwing or matching everything.
- A trailing byline `Span` stays attached after the article body regardless of
  how the body was split.

## Architecture

One `entry` module, one page. There is no model layer and no search service -
the whole feature is one pure function plus a `ForEach`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets          program entry, nothing feature-specific
├── entrybackupability/EntryBackupAbility.ets
└── pages/Index.ets                        @Entry: the input, processSource(), the Span list
```

The documented tree does **not** match the zip: the doc writes
`EntryBackAbility.ets` where the zip ships `EntryBackupAbility.ets`
(`HW-11-0019`). `module.json5` references
`./ets/entrybackupability/EntryBackupAbility.ets`, so the zip is right and the
doc's tree is the truncated one.

**The design decision worth copying** is that `processSource(mainStr, subStr)`
is a *pure function returning data*, not a builder that emits components. It
takes two strings and hands back `formatString[]`, an array of
`{ char, isAim }`. The UI layer never touches indices; it only asks "is this run
highlighted". That split is what makes the search logic testable without a
device, and it is why swapping the highlight style - underline instead of
colour, a background chip instead of bold - costs one line in `build()` and
nothing in the algorithm.

The decision **worth avoiding** in the same file is calling that function from
inside `build()`. `processSource` is invoked as the `ForEach` data source, so it
re-runs on every re-render of the page, over the whole article, for every
keystroke. On a short demo string that is free; on a real article it is a scan
per frame. Compute it into an `@State formatString[]` in the `TextInput`'s
`onChange` handler instead.

## Implementation steps

1. **Load the body once** in `aboutToAppear` via
   `context.resourceManager.getStringByNameSync('Text')`. The synchronous
   variant is fine here because it is one small resource read before first
   paint.
2. **Escape the query before building the RegExp.** `escapeRegExp` replaces
   every metacharacter with a backslash-prefixed copy. Without it, a user typing
   `(` throws `SyntaxError` inside `RegExp` and a user typing `.` highlights
   every character in the article.
3. **Guard the empty case first.** With an empty `subStr` the escaped pattern is
   the empty regex, `exec` matches a zero-length string at every position and
   never advances `lastIndex` - the `while` loop never terminates. The sample's
   `if (mainStr.length === 0 || subStr.length === 0)` early return is what keeps
   the page alive while the input is empty, which is most of the time.
4. **Collect match boundaries, not match contents.** Each successful `exec`
   pushes `matchResult.index` and `globalMatchRegex.lastIndex - 1` - the first
   and last character positions of that occurrence.
5. **Walk the article once**, accumulating non-matching characters into a
   `cache` string and flushing it whenever a boundary index is reached, then
   pushing the matched run and jumping the cursor past it with
   `index = index + subStr.length - 1`.
6. **Render as `Span`s inside one `Text`,** branching on `isAim` for the colour
   and weight. Keep the trailing byline `Span` outside the `ForEach` so it
   survives every re-split.
7. **Fix the tree entry** if you are copying the doc's layout:
   `EntryBackupAbility.ets`, not `EntryBackAbility.ets` (`HW-11-0019`).

## Verified snippets

All snippets are from `RegularHighlightDemo.zip`,
`entry/src/main/ets/pages/Index.ets`.

**The escape and the match scan** (as shipped)

```typescript
class formatString {
  // 解析后的标签内容数组，char 是要显示的内容，isAim 为是否高亮标志
  char: string = '';
  isAim: boolean = false;
}

// 转义特殊字符的函数
escapeRegExp(subStr: string): string {
  return subStr.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

processSource(mainStr: string, subStr: string): formatString[] {
  let result: formatString[] = [];
  if (mainStr.length === 0 || subStr.length === 0) {
    // 如果主串或子串为空，则返回空结果
    result = [{ char: mainStr, isAim: false }];
    return result;
  }

  // 对子串进行转义处理
  let escapedSubStr = this.escapeRegExp(subStr);
  // 构建正则表达式
  let globalMatchRegex = RegExp(escapedSubStr, 'g');
  let indexList: number[] = [];
  let matchResult: RegExpExecArray | null = null;
  // 记录所有匹配位置的索引
  while ((matchResult = globalMatchRegex.exec(mainStr)) !== null) {
    indexList.push(matchResult.index, globalMatchRegex.lastIndex - 1);
  }
  // ...
}
```

**Two lines carry the whole safety story.** `replace(/[.*+?^${}()|[\]\\]/g,
'\\$&')` is the standard escape: the character class lists every regex
metacharacter, `$&` in the replacement means "the text that matched", so each
metacharacter becomes itself preceded by a backslash. The user's `1.5` then
matches the literal string `1.5` rather than `1` + any character + `5`. Skip
this and the feature is a live regex console attached to your article - typing
`(` is a crash, typing `*` is a `SyntaxError`, typing `.` highlights everything.

The empty-string guard on the second line is not defensive padding, it is the
termination condition. A global regex with an empty pattern matches at every
position with zero length, `lastIndex` never advances, and `exec` returns a
truthy result forever. Because the query box starts empty and returns to empty
whenever the user clears it, this branch is the common path, not the edge case.

Note the returned shape: `[{ char: mainStr, isAim: false }]` - one unhighlighted
run covering the whole article, not an empty array. The renderer therefore has
exactly one code path.

**Splitting the article into runs** (as shipped)

```typescript
let cache = '';
for (let index = 0; index < mainStr.length; index++) {
  if (!indexList.includes(index)) {
    cache = cache + mainStr[index];
    if (index === mainStr.length - 1) {
      result.push({ char: cache, isAim: false });
    }
  } else {
    if (cache.length > 0) {
      result.push({ char: cache, isAim: false });
      cache = '';
    }
    result.push({ char: subStr, isAim: true });
    index = index + subStr.length - 1;
  }
}
return result;
```

**`cache` is the plain-text accumulator and the jump is the match consumer.**
Non-matching characters pile into `cache` until either a boundary index is hit -
flush the accumulated run, emit the match - or the article ends, which is what
the `index === mainStr.length - 1` check catches. Without that check a query
whose last occurrence is not at the end of the article silently drops the tail:
the loop exits with a full `cache` and nobody pushes it.

`index = index + subStr.length - 1` followed by the loop's own `index++` moves
the cursor exactly one past the last character of the match, which is also the
second boundary the scan recorded. That is why pushing *both* the start and end
index works: the end index is always consumed by the jump, never re-entered.

Two costs are worth knowing before you copy this into production.
`indexList.includes(index)` is a linear scan inside a linear loop, so the split
is O(article length x match count) - use a `Set`, or walk the sorted boundary
list with a pointer. And the emitted run is `subStr`, the *query*, not the
matched slice of the article. Those are identical only because the pattern is
fully escaped and case-sensitive; add an `'i'` flag for case-insensitive search
and the highlighted text will silently be re-cased to whatever the user typed.
Push `mainStr.substring(index, index + subStr.length)` instead.

**Rendering the runs** (as shipped)

```typescript
TextInput({ text: this.subStrInfo, placeholder: $r('app.string.Input') })
  .borderRadius(24)
  .cancelButton({
    style: CancelButtonStyle.CONSTANT,
    icon: { size: 32, src: $r('app.media.cancel') }
  })
  .onChange((value: string) => {
    this.subStrInfo = value;
  });

// ...
Text() {
  ForEach(this.processSource(this.mainStrInfo, this.subStrInfo), (item: formatString) => {
    if (item.isAim) {
      Span(item.char)
        .fontWeight(700)
        .fontSize(16)
        .fontColor('#0A59F7');   // 对目标内容进行高亮处理
    } else {
      Span(item.char)
        .fontWeight(400)
        .fontSize(16);
    }
  });
  Span($r('app.string.Worker'))
    .fontSize(16)
    .fontColor('#66000000');
}
```

**`ForEach` emitting `Span`s inside a bare `Text()` is the load-bearing choice.**
`Text` with a child block treats its `Span` children as a single inline run, so
the paragraph wraps across the highlighted words exactly as it would if they
were plain text. Build the same thing out of `Text` components in a `Row` and
each highlight becomes an unbreakable layout box; a match near a line end pushes
the whole word onto the next line and the paragraph develops ragged holes.

`CancelButtonStyle.CONSTANT` keeps the clear icon visible even when the field is
empty, which is the right call for a search box that also acts as the highlight
switch - the affordance for "turn highlighting off" should not itself be
conditional on there being something to turn off.

The `ForEach` here has **no key generator**, so ArkUI falls back to its default
keying and re-creates the whole `Span` list on every keystroke. For a paragraph
that is acceptable; for a long article, combine the pre-computed `@State` array
from the Architecture note with an explicit key over the run's start offset.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `deviceTypes` is
`phone`, `tablet`, `2in1`.

The article body, the placeholder, the section headings and the trailing byline
are all string resources, and `resources` carries `base`, `en_US` and `zh_CN`.
The body is read by *name* (`getStringByNameSync('Text')`) rather than by
`$r('app.string.Text')`, because the algorithm needs a plain `string` to index
into - a `Resource` has no `.length`. That is the correct reason to reach for
`resourceManager` instead of `$r`, and the only place in the sample where it is
justified.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Matching is **case-sensitive** and has no diacritic folding. For a Latin-script
  article this is usually not what a reader expects from a find box.
- Search is over one in-memory string. There is no paging, no scroll-to-match and
  no match counter; the article is short enough that every match is on screen.
- `private uiContext = this.getUIContext()` is evaluated as a member
  initialiser, before the component is attached. It happens to work here because
  it is only dereferenced later in `aboutToAppear`, but resolving the UI context
  inside the lifecycle callback is the safer form.
- `processSource` runs on the UI thread inside `build()`. See the Architecture
  note before using this on anything longer than a few paragraphs.

## Pitfalls

- **`HW-11-0019`** (E/low, confirmed): the document's 工程目录 lists
  `EntryBackAbility.ets`; the zip and `module.json5` both use
  `EntryBackupAbility.ets` - "Backup" truncated to "Back". Fix: correct the tree
  entry to `EntryBackupAbility.ets`.

## References

- `huawei_industry_tree/11_news_reading/docs/17_regular_highlight.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/regular_highlight-0000002328562941
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-span.md` - `Span` inside `Text`, and why the inline run wraps
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-span
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `Text` with child components
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `cancelButton` and `CancelButtonStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` keying and re-render behaviour
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
