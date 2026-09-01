---
id: SHOP-09
title: Editable search history - suggestion list, history chips and a delete mode over one TextInput
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/09_search_history.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history-0000002328895045
sample: huawei_industry_tree/16_shopping/downloads/SearchHistory.zip
kits: ["@kit.AbilityKit", "@kit.BasicServicesKit"]
apis: [TextInput, TextInputController, onChange, Stack, hitTestBehavior, HitTestMode, ForEach, Flex, FlexWrap, Tabs, TabContent, tabBar, BarMode, Span, linearGradient, "@StorageProp", onAreaChange, "window.getWindowAvoidArea", "AppStorage.setOrCreate", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0010]
status: verified-with-fixes
---

## When to use

Load this card when you are building **a search entry page** - the screen the
user lands on after tapping the search bar, before any results exist. It is
three things stacked in one page: a live suggestion list while the user types,
a chip cloud of previous searches with an edit/delete mode, and a hot-search
ranking behind tabs.

The pattern is one `@State` string (`currentSearchContent`) driving a single
branch. Non-empty means "show suggestions"; empty means "show history plus hot
search". There is no navigation, no second page, no route: the same page swaps
its middle section. That is the shape almost every shopping, travel and video
app uses, and it generalises to any search box that has something useful to
show before the query is submitted.

The two transferable techniques here are the **three-span highlight** (split
each candidate into before / match / after and colour only the middle span) and
the **delete mode** (one boolean that changes both the chip label and what a tap
on the chip does, instead of adding per-chip delete buttons). Both are cheap and
both survive being lifted into a real app. The history array handling is the
part you should not lift verbatim - see `HW-16-0010`.

## Feature checklist

- Typing in the search bar immediately replaces the page body with a suggestion
  list; clearing it restores history and hot search.
- Each suggestion shows the typed substring in red (`#E84026`) and the rest in
  black, at whatever position the match occurs.
- Pressing the 搜索 (search) button pushes the typed word to the front of the
  history, removes any earlier copy of the same word, and clears the input.
- A trash icon next to 搜索历史 switches the chip cloud into delete mode; every
  chip grows a ` ×` suffix and tightens its horizontal padding.
- In delete mode a tap removes that chip; outside delete mode a tap loads the
  word back into the input.
- A 完成 (done) label replaces the trash icon while in delete mode and exits it.
- Hot search sits under a scrollable tab bar; the top three ranks are numbered
  in gold/silver/bronze and their rows carry a red-to-white gradient.
- The page draws under the status bar and navigation indicator, padded by the
  avoid areas the ability measured.

## Architecture

One `entry` module. Everything visual lives in a single page; the three data
shapes are separate model files and every literal is in `Constants`.

```
entry/src/main/ets
├── common/Constants.ets            numeric literals + the 16-entry SEARCH_DATA corpus
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── model
│  ├── HistoryWordModel.ets         { word } + the 11 seeded history entries
│  ├── HotSearchModel.ets           TabItemModel (5 tabs) + HotDetailItem (12 rows)
│  └── LinkWordModel.ets            the pre-split suggestion: start/mid/end + lightIndex
└── pages/MainPage.ets              @Entry, 372 lines - the whole feature
```

The documented tree matches the zip exactly, file for file.

**The design decision worth copying** is `LinkWordModel`. The suggestion list
does not hold raw strings and re-run a match at render time; the matcher runs
once in `simulatorLinkWord()` and stores each candidate **already split into
three pieces plus an index saying which piece is the match**. Rendering is then
three `Span`s and a ternary on `lightIndex` - no string work inside the
`ForEach`, no measurement, no rich-text parser. Precomputing the split into the
view model is what keeps a per-keystroke list cheap.

The decision worth *avoiding* is putting 372 lines in one `@Entry` struct. The
suggestion list, the history cloud and the hot-search list are three unrelated
blocks that share exactly one state variable; in a real app they belong in three
`@Component`s taking that string as a `@Prop`. The sample already shows the seam
- `hotSearchTabBuilder` and `hotSearchContent` are pulled out as `@Builder`s.

## Implementation steps

1. **Hold the query in one `@State` string** and let it decide the whole body:
   `if (this.showLinkWord) { suggestions } else { history + hot search }`.
   `showLinkWord` is set from `onChange`, never from anywhere else.
2. **Float the icon and the button over the input with a `Stack`,** and put
   `hitTestBehavior(HitTestMode.None)` on the overlay `Row` so the row is
   transparent to touch while its `Button` child still receives clicks.
3. **Precompute the highlight split** in the change handler: `indexOf` the query
   in each candidate, slice into start/mid/end, and record which of the three
   equals the query in `lightIndex`.
4. **Dedupe the history into a new array before unshifting** -
   `filter()`, not `splice()` inside a `forEach()` (`HW-16-0010`).
5. **Reuse one function for insert and delete.** `dealHistoryData(word, isDelete)`
   removes the matching entry either way and only re-inserts when `isDelete` is
   false, so "search an existing word" and "delete a word" share one code path.
6. **Drive delete mode from a single boolean.** `isDeleteState` selects the chip
   label (`item.word` vs `item.word + ' ×'`), the chip's horizontal padding, and
   the branch inside the chip's `onClick`.
7. **Give the hot-search `Tabs` an explicit height.** A `Tabs` inside a `Scroll`
   has no intrinsic height; the sample computes
   `hotDetaiList.length * Constants.TAB_CONTENT_HEIGHT`.
8. **Read the avoid areas from `AppStorage` with `@StorageProp`** and convert
   with `px2vp` at the point of use - the ability stores px.

## Verified snippets

All snippets are from `SearchHistory.zip`. Corrected forms are marked.

**History insert and delete — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-16-0010`)

```typescript
dealHistoryData(wordModel: HistoryWordModel, isDelete: boolean) {
  if (wordModel.word.length !== Constants.ZERO_LENGTH) {
    // FIX: the sample splices inside forEach over the same array
    this.historyWords = this.historyWords.filter(item => item.word !== wordModel.word);
    if (!isDelete) {
      this.historyWords.unshift(new HistoryWordModel(wordModel.word));
    }
  }
}
```

The shipped version is `this.historyWords.forEach((item, index) => { if (item.word === wordModel.word) { this.historyWords.splice(index, 1); } })`.
Mutating the array `forEach` is walking shifts every later element one slot
left, and `forEach` has already advanced its cursor - so the element directly
after any removal is never visited. With the eleven distinct seeded words the
bug is invisible; feed the list from disk or from a server and any pair of
adjacent duplicates survives.

Reassigning the whole array rather than mutating it also matters for `@State`:
`filter()` produces a new array reference, which is the change ArkUI observes
most reliably. `unshift` on the fresh array is then observed too, because
`@State` on an `Array` tracks the built-in mutating methods.

**The three-span highlight — same file** (as shipped, trimmed)

```typescript
simulatorLinkWord() {
  this.linkWordList = [];
  Constants.SEARCH_DATA.forEach((value: string) => {
    let startStr: string = '';
    let midStr: string = '';
    let endStr: string = '';
    let hIndex: number = Constants.MINUS_ONE;

    let position = value.indexOf(this.currentSearchContent);
    if (position !== Constants.MINUS_ONE) {
      if (position === Constants.ZERO_INDEX) {
        startStr = value.toString().slice(Constants.ZERO_INDEX, this.currentSearchContent.length);
      } else {
        startStr = value.slice(Constants.ZERO_INDEX, position);
      }
      if (startStr.length < value.length) {
        position = value.slice(startStr.length).indexOf(this.currentSearchContent);
        if (position === Constants.MINUS_ONE) {
          midStr = value.slice(startStr.length);
        } else {
          midStr = value.slice(startStr.length, this.currentSearchContent.length + startStr.length);
        }
        if (startStr.length + midStr.length < value.length) {
          endStr = value.slice(startStr.length + midStr.length);
        }
      }
      if (startStr === this.currentSearchContent) {
        hIndex = Constants.ZERO_INDEX;
      } else if (midStr === this.currentSearchContent) {
        hIndex = Constants.ONE_INDEX;
      } else if (endStr === this.currentSearchContent) {
        hIndex = Constants.TWO_INDEX;
      }
      this.linkWordList.push(new LinkWordModel(Constants.NORMAL_COLOR,
        Constants.SELECT_COLOR, hIndex, startStr, midStr, endStr));
    }
  });
}
```

**The whole matcher exists to answer one question: which of the three pieces is
the match.** When the query is at position 0 the match becomes `startStr` and
`lightIndex` is 0; when it is in the middle, `startStr` is the prefix and the
match lands in `midStr`; when it runs to the end it lands in `endStr`. The list
then renders three `Span`s guarded by length checks and picks
`item.lightColor` for whichever index matches:

```typescript
Span(item.midWord)
  .fontColor(item.lightIndex === Constants.ONE_INDEX ? item.lightColor : item.normalColor);
```

Two things are worth knowing before copying it. The corpus
(`Constants.SEARCH_DATA`) is 16 hardcoded 手机 (mobile phone) strings, so every
query that starts with anything else yields an empty list and the page shows a
blank area rather than a "no suggestions" state. And the matcher only handles
the *first* occurrence: a candidate containing the query twice highlights one of
them. For a real implementation, replacing the whole function with a single
`indexOf` and two slices is both shorter and equivalent.

**The floating search button — same file** (as shipped, trimmed)

```typescript
Stack() {
  TextInput({
    placeholder: $r('app.string.search_placeholder'),
    controller: this.controller,
    text: this.currentSearchContent
  })
    .padding({
      left: $r('app.integer.search_content_padding_left'),
      right: $r('app.integer.search_content_padding_right')
    })
    .onChange((currentContent) => {
      this.currentSearchContent = currentContent;
      if (this.currentSearchContent.length !== Constants.ZERO_LENGTH) {
        this.showLinkWord = true;
        this.simulatorLinkWord();
      } else {
        this.showLinkWord = false;
      }
    });
  Row() {
    Image($r('app.media.ic_search'))
      .width($r('app.integer.width_sixteen'));
    Button($r('app.string.search'))
      .backgroundColor($r('app.color.search_background_color'))
      .onClick(() => {
        if (this.currentSearchContent.length !== Constants.ZERO_LENGTH) {
          this.showLinkWord = false;
          this.dealHistoryData(new HistoryWordModel(this.currentSearchContent), false);
          this.currentSearchContent = '';
        }
      });
  }.width(Constants.FULL_PERCENT)
  .hitTestBehavior(HitTestMode.None)
  .justifyContent(FlexAlign.SpaceBetween)
  .padding({ left: $r('app.integer.padding_ten'), right: $r('app.integer.padding_two') });
}.alignContent(Alignment.Start)
```

**`hitTestBehavior(HitTestMode.None)` is the load-bearing line.** The overlay
`Row` spans the full width of the input, so without it every tap in the middle
of the field would hit the transparent row instead of the `TextInput` and the
keyboard would never open. `HitTestMode.None` makes the node itself invisible to
hit testing while leaving its children in the test - so the `Button` at the
right end still gets its clicks and the rest of the row passes taps through to
the input underneath. The two `padding` values on the `TextInput` are what
reserve the space the icon and the button sit in.

Note the submit path: it clears `currentSearchContent`, which fires `onChange`
with an empty string, which sets `showLinkWord = false` - so the explicit
`this.showLinkWord = false` in the handler is belt and braces, and the page
returns to the history view by the same route the user would get by clearing the
field by hand.

**Delete mode over the chip cloud — same file** (as shipped, trimmed)

```typescript
Flex({ direction: FlexDirection.Row, wrap: FlexWrap.Wrap }) {
  ForEach(this.historyWords, (item: HistoryWordModel) => {
    Text(this.isDeleteState === false ? item.word : item.word + ' ×')
      .margin($r('app.integer.history_content_margin'))
      .backgroundColor($r('app.color.background_color'))
      .padding({
        left: this.isDeleteState === false ? $r('app.integer.padding_fifteen') :
          $r('app.integer.padding_nine'),
        right: this.isDeleteState === false ? $r('app.integer.padding_fifteen') :
          $r('app.integer.padding_nine'),
        top: $r('app.integer.padding_nine'),
        bottom: $r('app.integer.padding_nine')
      })
      .borderRadius($r('app.integer.history_content_radius'))
      .textOverflow({ overflow: TextOverflow.Ellipsis })
      .onClick(() => {
        if (this.isDeleteState) {
          this.dealHistoryData(item, true);
        } else {
          this.currentSearchContent = item.word;
        }
      });
  });
}.width(Constants.FULL_PERCENT)
```

**The delete affordance is a suffix, not a component.** Appending `' ×'` to the
label instead of composing a `Row` with a close icon keeps the chip a single
`Text`, which keeps `Flex`+`FlexWrap.Wrap` reflow trivial and costs nothing when
the mode is off. The horizontal padding shrinks from 15 to 9 in delete mode to
partly absorb the extra glyph, so chips do not jump a row when the mode toggles.

`ForEach` here has **no key generator**, so ArkUI falls back to index-based keys.
Combined with `unshift`, that means every chip's key changes when a word is
added and the whole cloud is rebuilt. For eleven `Text` nodes this is
irrelevant; for a chip cloud backed by real history, pass
`(item: HistoryWordModel) => item.word` as the third argument. The sibling
`ForEach`es over the hot-search data in this same file do supply key generators
(`JSON.stringify(item)`), so the omission is an inconsistency rather than a
house style.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. `deviceTypes` is
`phone`, `tablet`, `2in1`.

The ability forces light mode
(`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`) before doing
anything else, which is why the page can hardcode `Color.White` backgrounds
without a dark-mode palette. It then goes full screen with
`setWindowLayoutFullScreen(true)`, reads `TYPE_SYSTEM` and
`TYPE_NAVIGATION_INDICATOR` avoid areas into `AppStorage` as `topRectHeight` /
`bottomRectHeight`, and subscribes to `avoidAreaChange` to keep them current.
The page consumes them as `@StorageProp` and converts at the point of use:

```typescript
.padding({ top: this.uiContext.px2vp(this.topRectHeight), bottom: this.uiContext.px2vp(this.bottomRectHeight) })
```

Note that the `avoidAreaChange` subscription is never released in
`onWindowStageDestroy` - the same boilerplate omission that recurs across this
corpus.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Suggestions come from a 16-string hardcoded array, not a service. The function
  is named `simulatorLinkWord` for that reason; wire it to a debounced request
  before shipping, because it currently runs on every keystroke.
- History is in-memory only. Nothing is persisted, so the eleven seeded words
  come back on every cold start and the user's own searches do not.
- `currentHistoryHeight` is captured once in `onAreaChange` and then never read
  anywhere - dead state left from a collapse feature that is not in this sample.
  For the collapse behaviour see `SHOP-12`.
- The hot-search rows carry `.height('app.integer.height_forty')` - a bare
  string, not `$r('app.integer.height_forty')`. It parses as an invalid length
  and the row falls back to its content height, so the rank rows are shorter
  than intended.
- All five hot-search tabs render the same `hotSearchContent()` list; the tab
  index selects nothing.

## Pitfalls

- **`HW-16-0010`** (B/low, confirmed): `dealHistoryData` splices the history
  array while `forEach` is iterating it, so the element after any removed entry
  is skipped and consecutive duplicates survive. Fix: rebuild with
  `this.historyWords = this.historyWords.filter(item => item.word !== wordModel.word);`
  then `unshift`.

## References

- `huawei_industry_tree/16_shopping/docs/09_search_history.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history-0000002328895045
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `TextInput`, `onChange`, `TextInputController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and why the key generator matters
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-container-flex.md` - `FlexWrap.Wrap` for the chip cloud
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-flex
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `BarMode.Scrollable`, `tabBar` builders
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-gradient-color.md` - the rank-row `linearGradient`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-gradient-color
- `SHOP-12` - the expand/collapse this history cloud is missing
