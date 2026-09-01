---
id: SHOP-20
title: Carousel placeholder search - a vertical Swiper over the Search box, submitted as the query
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/20_search_recommendation_word_carousel.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_recommendation_word_carousel-0000002348742090
sample: huawei_industry_tree/16_shopping/downloads/RecommendSearch.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Swiper, SwiperController, Search, SearchController, Navigation, NavPathStack, NavDestination, NavDestinationContext, pushPath, pop, onReady, "@Builder", routerMap, "$$", Stack, ForEach, defaultFocus, onSubmit, onChange, "UIContext.getFocusController", clearFocus, visibility, "@StorageProp", "UIContext.px2vp", "window.getWindowAvoidArea", "window.on('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0027, HW-16-0034, HW-16-0035]
status: verified
---

## When to use

Load this card when the search box on a home page should **suggest what to
search for before the user types**, and pressing search with an empty field
should run the suggestion currently on screen. Every large shopping app does
this; so do app stores, food delivery and travel apps.

The pattern has two halves that are easy to get wrong separately. The visual
half is that the rotating word is *not* the `Search` component's `placeholder`
property - a placeholder cannot animate. It is a separate vertical `Swiper` of
`Text` items stacked over the search box inside a `Stack`, positioned to land
where the placeholder would be. The behavioural half is that the currently
displayed word must be readable at the moment of submit, which means binding
the Swiper's index back into the page.

It generalises to any "ambient suggestion that becomes the input on commit" -
rotating hint text in a message composer, a cycling promo code field, a
"try: ..." prompt above a command box. The carrying idea is that the hint is a
real component with real state, not a string attribute.

## Feature checklist

- A home page with a scan icon, a search box, a camera icon and a search
  button.
- Three recommended keywords cycle vertically inside the search box, one every
  2 seconds, looping, with no indicator dots.
- Tapping the search button navigates to a result page and runs the keyword
  currently on screen.
- Tapping the rotating word itself navigates to the result page with that word
  pre-loaded as the placeholder instead of as the query, so the user can type
  over it.
- The result page opens with the keyboard focused when it arrives empty, and
  unfocused when it arrives with a query.
- Submitting an empty field on the result page falls back to the placeholder.
- Results appear only for a recognised keyword; the camera icon inside the
  field hides as soon as the user types.
- A back arrow pops the navigation stack.

## Architecture

One `entry` module, two pages, one parameter class. The result page is reached
through the router map, not through `pages`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full screen, both avoid areas, avoidAreaChange subscription
├── entrybackupability/
├── model/SearchContent.ets         SearchContentClass: searchContent + placeholderContent
└── pages
    ├── RecommendSearch.ets         @Entry: Navigation root, the Stack of Search + Swiper
    └── SearchPage.ets              the NavDestination result page + its @Builder
```

The documented tree matches the zip.

`entry/src/main/resources/base/profile/route_map.json` registers the
destination, and `module.json5` points at it with
`"routerMap": "$profile:route_map"`:

```json5
{ "name": "searchPage",
  "pageSourceFile": "src/main/ets/pages/SearchPage.ets",
  "buildFunction": "SearchPageBuilder" }
```

**The design decision worth copying** is the two-field parameter class. A
single "the search term" string would have been the obvious model, and it would
have collapsed two different intents:

```typescript
export class SearchContentClass {
  searchContent: string = '';        // run this query now
  placeholderContent: string = '';   // offer this word, but let me type
}
```

The search button fills `searchContent`; tapping the rotating word fills
`placeholderContent`. `SearchPage.onReady` reads both and derives its whole
initial state from which one is populated - including whether to take keyboard
focus. Two nullable fields express "committed" versus "suggested" with no flag
and no enum, and the result page needs no knowledge of where it was pushed
from.

## Implementation steps

1. **Put the `Search` and the `Swiper` in a `Stack`** aligned
   `Alignment.Start`, with the Swiper declared second so it draws on top.
2. **Make the Swiper vertical, autoplaying and looping**, with `indicator(false)`
   - dots would give away that it is a carousel and not placeholder text.
3. **Offset the Swiper to the placeholder position** (`offset({ x: 36 })` here,
   clearing the magnifier icon) and give it a width short of the field so the
   camera icon at the right stays visible.
4. **Bind the index with `$$`** so the page can read which word is showing at
   submit time.
5. **Register the result page in `route_map.json`** with a `buildFunction`, and
   export a matching `@Builder` from the destination file.
6. **Push with a typed parameter object**, filling `searchContent` from the
   button and `placeholderContent` from a tap on the word itself.
7. **Read the parameter in `NavDestination.onReady`,** and take the
   `NavPathStack` from the context there rather than constructing one - the
   stack the destination was pushed onto is the one it must pop.
8. **Set `defaultFocus` from whether a query arrived**, so an empty arrival
   opens the keyboard and a committed arrival shows results instead.
9. **Fall back to the placeholder on empty submit**, in both the button handler
   and `onSubmit`.

## Verified snippets

All snippets are from `RecommendSearch.zip`.

**The stacked search box — `entry/src/main/ets/pages/RecommendSearch.ets`** (as shipped)

```typescript
private swiperController: SwiperController = new SwiperController();
@State placeholderList: string[] = ['华为热门机型', '618电器国补', '轻奢椅子'];
index: number = 0;                      // deliberately not @State - see below

Stack({ alignContent: Alignment.Start }) {
  Search({ value: this.changeValue, controller: this.controller })
    .width('100%')
    .height(34)
    .backgroundColor('#e6e8e9')
    .placeholderFont({ size: 14, weight: 400 })
    .textFont({ size: 14, weight: 400 });

  // Automatic loop display of recommended word list
  Swiper(this.swiperController) {
    ForEach(this.placeholderList, (item: string) => {
      Text(item.toString())
        .textAlign(TextAlign.Start)
        .fontSize(16);
    }, (item: string) => item);
  }
  .index($$this.index)                  // two-way: the Swiper writes the current page back
  .vertical(true)
  .autoPlay(true)
  .interval(2000)
  .loop(true)
  .indicator(false)
  .duration(500)
  .height(38)
  .width('80%')
  .offset({ x: 36 })                    // clear the magnifier icon
  .onClick(() => {
    let parm: SearchContentClass = new SearchContentClass();
    parm.placeholderContent = this.placeholderList[this.index];
    this.pageInfo.pushPath({ name: 'searchPage', param: parm });
  });

  Image($r('app.media.camera'))
    .width(30)
    .margin({ left: '82%' });
}
.width('64%');
```

**Five Swiper options make it read as placeholder text rather than a carousel.**
`vertical(true)` gives the upward roll the eye expects from a hint line;
`indicator(false)` removes the dots that would announce a carousel;
`autoPlay` + `interval(2000)` + `loop(true)` keep it cycling with no user
input; and `duration(500)` is slow enough to read as motion, not a cut. The
geometry does the rest: `height(38)` against the `Search`'s `height(34)`, and
`offset({ x: 36 })` to sit where the placeholder would.

**`$$this.index` is the load-bearing line.** The Swiper advances itself, so the
page has no other way to know which word is showing; `$$` is ArkUI's two-way
binding sugar and makes the Swiper write its current page number back. Note
that `index` here is a plain member, not `@State`. That is the reason the
search bar does not re-render every 2 seconds - and it works because `index` is
only ever *read* imperatively, inside click handlers, never in `build()`. It is
a real optimisation and a fragile one: the moment anything in `build()` reads
`this.index`, the missing decorator becomes a stale-render bug with no
diagnostic. If you copy this, either keep the read strictly imperative and
leave a comment saying so, or make it `@State` and accept the repaint.

The `Search` on this page has no `placeholder`, no `onChange` and no
`onSubmit`. It is scenery - a styled box for the Swiper to sit in. Its visible
area outside the Swiper's 80% is still tappable and will focus an input the
page does nothing with; a stricter implementation would replace it with a
styled `Row` and let the whole thing navigate.

**Committing the current word — same file** (as shipped)

```typescript
Button($r('app.string.search'), { stateEffect: true, type: ButtonType.Capsule })
  .width('20%')
  .height(40)
  .margin({ left: 10 })
  .onClick(() => {
    let parm: SearchContentClass = new SearchContentClass();
    parm.searchContent = this.placeholderList[this.index];   // committed, not suggested
    this.pageInfo.pushPath({ name: 'searchPage', param: parm });
  });
```

Compare this with the Swiper's own `onClick` above: identical push, different
field. The button means "search this now"; tapping the word means "put this in
the box for me". Everything downstream - focus, results visibility, whether the
field shows text or grey hint - falls out of that one choice, because
`onReady` derives all of it.

Both handlers read `this.placeholderList[this.index]` at click time rather than
tracking the word in state, which is the correct read: the Swiper may advance
between renders, and the index is authoritative at the instant of the tap.

**Receiving the parameter — `entry/src/main/ets/pages/SearchPage.ets`** (as shipped)

```typescript
@Builder
export function SearchPageBuilder() {
  SearchPage();
}

// ...
NavDestination() { /* ... */ }
.hideBackButton(true)
.onReady((context: NavDestinationContext) => {
  // Receive recommended words transmitted from the homepage
  this.pageInfo = context.pathStack;                                        // the real stack
  this.changeValue = (context.pathInfo.param as SearchContentClass).searchContent;
  this.placeholder = (context.pathInfo.param as SearchContentClass).placeholderContent;
  this.submitValue = this.changeValue;
  if (this.changeValue) {
    this.autoFocus = false;      // arrived with a query: show results, do not open the keyboard
  }
});
```

**`onReady` is the only correct place to do this.** It is the hook that hands
the destination both its parameter and the `NavPathStack` it was pushed onto.
The field `pageInfo` is initialised to `new NavPathStack()` at declaration, but
that instance is a placeholder - popping it would do nothing. Reassigning from
`context.pathStack` is what makes `this.pageInfo.pop({ number: 1 })` on the back
arrow work.

The four assignments are the whole state derivation: a committed query becomes
the field's text *and* the submitted value (so results render immediately), a
suggestion becomes the placeholder only, and focus is taken unless a query
arrived. `hideBackButton(true)` plus a custom back `Image` is what lets the
back control sit inline with the search field instead of in a title bar.

**Results and the empty-field fallback — same file** (as shipped)

```typescript
Search({ value: this.changeValue, placeholder: this.placeholder, controller: this.controller })
  .focusable(true)
  .defaultFocus(this.autoFocus)
  .onSubmit((value: string) => {
    if (this.changeValue === '') {
      this.changeValue = this.placeholder;      // empty submit runs the suggestion
      this.submitValue = this.changeValue;
    } else {
      this.submitValue = value;
    }
  })
  .onChange((value: string) => {
    this.changeValue = value;
  });

Image($r('app.media.camera'))
  .visibility(this.changeValue ? Visibility.None : Visibility.Visible);

List({ space: 6 }) {
  ForEach(this.searchResult1, (item: Resource) => {
    ListItem() { Image(item).width('100%'); };
  }, (index: Resource) => index.id.toString());
}
.scrollBar(BarState.Off)
.visibility((this.placeholderList.includes(this.submitValue)) ? Visibility.Visible : Visibility.None);
```

**`changeValue` and `submitValue` are two different things and the separation
matters.** `changeValue` tracks every keystroke through `onChange`;
`submitValue` only moves on an explicit submit. Results are bound to
`submitValue`, so typing does not tear down the previous result list
mid-keystroke - the classic bug when a single field drives both. The camera
icon is bound to `changeValue` instead, because it *should* disappear the
moment the user starts typing.

The button handler duplicates the empty-field fallback and adds
`this.getUIContext().getFocusController().clearFocus()`, which dismisses the
soft keyboard so the results are not hidden behind it - `onSubmit` gets that
for free from the IME's own search key, which is why it does not repeat the
call.

Results are gated on `placeholderList.includes(submitValue)` and are six static
images. This is a demo backend: only the three recommended words match, and any
other term renders nothing at all - not an empty state, just an absent list.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is `phone`, `tablet`, `2in1`. `module.json5` carries both
`"pages": "$profile:main_pages"` and `"routerMap": "$profile:route_map"`, which
is the correct pairing for a `Navigation`-based app - `main_pages` for
`loadContent`, `route_map` for `pushPath` by name.

`EntryAbility` is the most complete of the four shopping samples reviewed here:
it sets full-screen layout with a `.catch((err: BusinessError) => ...)`, reads
both avoid areas, **and subscribes to `avoidAreaChange`** to keep them current:

```typescript
windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});
```

Both pages consume the values with `@StorageProp` and a typed `0` default,
converting with `px2vp` in the padding. That is the shape to copy. The one gap
is symmetry: `off('avoidAreaChange')` is never called in
`onWindowStageDestroy`, the same omission flagged across other industries'
samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The recommended words are a hardcoded three-entry array, duplicated in both
  `RecommendSearch.ets` and `SearchPage.ets` - the result page needs its own
  copy to decide whether a term is recognised. Two literals that must stay in
  sync; a shared constant belongs in `model/`.
- Search results are six static images shown or hidden wholesale. Nothing is
  filtered, ranked or fetched.
- An unrecognised term produces a blank page with no "no results" message.
- `SearchPage` carries `@Entry` even though it is used as a `NavDestination`
  through `SearchPageBuilder`, and `main_pages.json` lists it alongside
  `RecommendSearch`. Harmless, but it advertises a second entry point that the
  app never uses; a pure destination needs neither.
- Most of the home page is a screenshot: `app.media.function`,
  `app.media.mainPageList` and `app.media.tab` are images standing in for the
  category grid, the feed and the bottom tab bar. Only the search row is real.
- The Swiper's `SwiperController` is constructed and never used - the carousel
  is driven entirely by `autoPlay`.
- The `ForEach` over results names its key parameter `index` while it is a
  `Resource`; it keys on `Resource.id`, which is correct but reads as a bug.

## Pitfalls

- **No findings were filed against this document or sample.** The doc's two
  snippets are syntactically valid and correctly abridged - this doc is not
  among the instances of `HW-16-0013`, the corpus-wide finding on truncated
  excerpts - and the shipped code has no behavioural defect.
- **`index` is a plain member under `$$` two-way binding.** It works only
  because `build()` never reads it. Reading it in `build()` without adding
  `@State` gives a stale value with no warning. This is the one line in the
  sample most likely to break under modification.
- **The home page's `Search` is decorative but focusable.** Its edges outside
  the Swiper's 80% width accept taps and open a keyboard for a field with no
  handlers. Replace it with a styled container, or make the whole `Stack`
  navigate.
- **The keyword list is duplicated across the two pages.** Extract it.
- **`avoidAreaChange` is subscribed and never released** in
  `onWindowStageDestroy`.
- **`pageInfo` is initialised to a throwaway `new NavPathStack()`.** If a
  future edit pops or pushes before `onReady` has reassigned it from
  `context.pathStack`, the call silently does nothing.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `vertical`, `autoPlay`, `interval`, `loop`, `indicator`, `index`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - `Search`, `onSubmit`, `onChange`, `defaultFocus`, `SearchController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `Navigation`, `NavPathStack`, `pushPath`, `hideTitleBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - the router map, `buildFunction`, `NavDestination.onReady` and `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - `$$` two-way binding and what it requires of the bound variable
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `huawei_industry_tree/16_shopping/docs/20_search_recommendation_word_carousel.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_recommendation_word_carousel-0000002348742090
- `SHOP-12` - search history expand/collapse, the other half of this search entry point
