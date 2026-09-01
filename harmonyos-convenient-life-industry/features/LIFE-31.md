---
id: LIFE-31
title: Convenient-life industry FAQ redirect - and the defect families that recur across all 31 documents
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/31_practice-convenient-life-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_2-0000002263237450
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: [HW-02-0266, HW-02-0267]
status: verified-with-fixes
---

## When to use

This document has **no sample project**. It is an eleven-line page whose whole
body is a redirect:

> 行业常见问题内容已迁移至HarmonyOS FAQ，请点击[此处](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone)前往。
> ("The industry FAQ content has been migrated to the HarmonyOS FAQ; please
> click here to go there.")

So there is no feature to implement. What this card is for is the **other** job
the last document of an industry should do: telling you which of the thirty
feature cards to load, and what goes wrong often enough in this industry that
you should check for it before you check for anything else.

Load `LIFE-02` for the scenario catalogue - which document covers which
feature. Load this card when you are about to review or write convenient-life
code and want the failure list first.

## The redirect itself

Two things are wrong with the page, and both are about the word "industry" in
its title.

1. **HW-02-0266 - the destination has no industry anchor.** The link goes to
   `harmonyos-faqs/faq-phone`, the general phone FAQ index, with no fragment
   and nothing naming convenient life. The page is the last numbered document of
   the industry and is titled 行业常见问题 ("Industry FAQ"), so it is where a
   reader goes for this industry's known issues - and it hands them a general
   index with no way to tell which entries were the industry's. It also carries
   none of the sections every other document here provides: no 场景介绍, no
   实现思路, no 约束与限制, no 工程目录, no 参考文档, no 代码下载.

2. **HW-02-0267 - ten industries ship the identical page.** The body is
   character-for-character the same in `01_auto`, `02_convenient_life`,
   `03_sports_health`, `04_education`, `05_office`, `06_public_transport`,
   `07_finance_insurance`, `08_children_education`, `09_tourism` and
   `10_maternity_health`, and every one of them links to the same
   `faq-phone` URL. Only the source URL in the front matter differs. Either the
   migrated FAQ is no longer split by industry - in which case ten identically
   titled industry pages mislead - or it is, and none of the ten links reaches
   the split.

Neither is fixable in code. Treat the page as a dead end and go to the FAQ
directly.

## What the industry actually contains

Thirty documents with samples, reviewed one by one. Grouped by what they teach:

| Group | Cards |
| --- | --- |
| Architecture and catalogue | `LIFE-01` (reference app), `LIFE-02` (scenario catalogue) |
| Calendar and scheduling | `LIFE-08`, `LIFE-12`, `LIFE-13`, `LIFE-18`, `LIFE-24` |
| Map Kit | `LIFE-14` (bindSheet over a map), `LIFE-16` (custom route), `LIFE-20` (rotate and move), `LIFE-25` (commute distance, static map, Petal Maps), `LIFE-26` (nearby search, dial, navigate), `LIFE-29` (draggable coverage circle) |
| Recognition and AI | `LIFE-19` (address text), `LIFE-21` (address image), `LIFE-22` (live ID card scan), `LIFE-23` (CardRecognition control) |
| Security and identity | `LIFE-10` (Asset Store), `LIFE-15` (document type picker), `LIFE-23` |
| Lists, gestures and layout | `LIFE-03`, `LIFE-04`, `LIFE-05`, `LIFE-06`, `LIFE-07`, `LIFE-09`, `LIFE-11`, `LIFE-17`, `LIFE-27` |
| Web and concurrency | `LIFE-28` (javaScriptProxy bridge), `LIFE-30` (taskpool executor) |

## The recurring defect families

These are the patterns that showed up in more than one sample. When reviewing
convenient-life code, check these first - they account for the majority of the
267 findings recorded against this industry.

### 1. `on(...)` without `off(...)`

The single most common defect in the industry. Every one of these registers a
listener that is never unregistered:

- `window.on('avoidAreaChange')` in `onWindowStageCreate`, with an
  `onWindowStageDestroy` that only logs - `LIFE-23`, `LIFE-24`, `LIFE-25`,
  `LIFE-27`, `LIFE-28`.
- `MapEventManager.on(...)` with no `aboutToDisappear` - `LIFE-14`,
  `LIFE-29`.
- `ImageReceiver.on('imageArrival')` unregistered only on the success path -
  `LIFE-22`.

A frequent aggravating detail: the window handle is a **local** inside
`onWindowStageCreate` or inside a `getMainWindow` callback, so the unsubscribe
cannot even be added later without first storing it on the ability. `LIFE-26`
shows the alternative that cannot leak - `.expandSafeArea([SafeAreaType.SYSTEM],
[SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])` instead of the whole avoid-area dance.

### 2. Avoid areas read before the immersive layout applies

`setWindowLayoutFullScreen` returns `Promise<void>` because the layout is not in
effect when the call returns - and the immersive layout is precisely what turns
the system bars into avoid areas. Reading them on the next line races the change
that makes them correct. Wrong in `LIFE-23`, `LIFE-24`, `LIFE-25`, `LIFE-27`;
**right** in `LIFE-28`, which is the one to copy.

A variant: the promise dropped entirely, with neither `then` nor `catch`
(`LIFE-24`), or a `then` with no `catch` (`LIFE-28`).

### 3. `ForEach` / `LazyForEach` key generators that lie about their parameter

Written as `(item: string) => item` or `(index: number) => JSON.stringify(index)`
over an array of objects. The first parameter of a key generator is the **item**,
so the annotation is false at runtime and the compiler cannot catch it - and the
key handed to the framework is an object, not the unique string the contract
requires. `LIFE-26`, `LIFE-27`, `LIFE-30`. The guidance is blunt: "Duplicate key
values will cause rendering issues", and "use a unique id property from the
object data as the key". The data almost always has one - `site.siteId`,
`FileModel.path`, `category.name`.

### 4. State-decorator omissions and type mismatches

- An undecorated field driving a rendered style: `LIFE-24` (`selectedIndex`
  choosing a background), `LIFE-27` (`tabId` bound with `$$` and read for the
  tab-bar highlight - `$$` only triggers updates for decorated variables).
- `@Link` or `@Consume` declared without the `undefined` its source has:
  `LIFE-24` (`@Link selectedParam: SelectedTimeParam` against a `@State` that is
  `| undefined`), `LIFE-27` (`@Consume uiContext: UIContext` in three components
  against a `@Provide` that is `| undefined`). Both guides say the types must
  match, and both samples then guard against the value they declared impossible.
- `@Provide` with no `@Consume` anywhere: `LIFE-23`.

### 5. Unreleased image resources

`ImageSource`, `PixelMap` and `image.Image` all need releasing when the
application handles them itself. Missed in `LIFE-21` (fd + ImageSource +
previous PixelMap per selection), `LIFE-22` (a full-resolution PixelMap per
preview frame, plus `nextImage.release()` skipped on two early returns) and
`LIFE-30` (two ImageSources and a full-size PixelMap per thumbnail). The one
exception worth remembering: a `PixelMap` handed to an `Image` component is
managed by it and must **not** be released manually - which is why `LIFE-25`'s
static map has no leak.

### 6. Unchecked indexing of results the reference says can be empty

`routes[0].steps[0]` (`LIFE-25`), `geoAddress[0]` (`LIFE-26`),
`contacts[0].phoneNumbers?.[0].phoneNumber!` (`LIFE-28`),
`cardDataSource[0]` (`LIFE-23`). In each case the reference explicitly documents
the empty case - "If no routes are available, an empty array will be returned",
`phoneNumbers` marked optional - and in each case the surrounding `try` turns
the resulting TypeError into a mislabelled log line.

### 7. Silent failure paths

The industry's most consistent user-facing weakness:

- A `canIUse` guard with no `else`, leaving the user on a dead page:
  `LIFE-23`, `LIFE-28`.
- An operation that resolves `false` and a caller that only reacts to `true`:
  `LIFE-24`.
- An empty `catch`: `LIFE-26`.
- A `then` with no `catch` on a cross-application handoff: `LIFE-25`,
  `LIFE-26`.
- A button whose handler does nothing when a precondition fails:
  `LIFE-24`, `LIFE-26`.

### 8. Placeholder content left in shipped samples

Two samples display literal placeholder text where the recognised or searched
data belongs: `LIFE-23` (`Text('***')` and `Text('123456789877654321')` on the
real-name confirmation page) and `LIFE-26` (four commented-out bindings replaced
by `'xxxxxxxx'`). In both cases the data is computed correctly and simply never
rendered - the highest-severity findings in the industry.

### 9. Logging defects

- A `Logger` whose format string declares `'%{public}s, %{public}s'` while every
  method passes the rest parameter as one array: `LIFE-22`, `LIFE-24`,
  `LIFE-30`.
- Arguments in the wrong slots, putting the message where the tag belongs -
  which is subject to the documented 31-byte tag truncation: `LIFE-26`, and
  `HW-02-0009` earlier in the industry.
- Personal data logged as `{public}`: `LIFE-22` (national ID number and the
  raw OCR text), `LIFE-23` (the whole `cardInfo`, six lines below a comment
  forbidding exactly that).

### 10. Documents that diverge from their own samples

Forty-one Category E findings, most of them one of these shapes:

- Snippets that do not compile: a `try` with no `catch` and an undeclared
  variable (`LIFE-26`), a method call missing `this.` (`LIFE-29`), identifiers
  misspelled against the ZIP (`LIFE-27`: `swiperctr` for `swiperCtr`,
  `categorys` for `categories`).
- Snippets that drop the guard the ZIP has: `LIFE-23` omits both the
  deprecation comment and the `canIUse` wrapper; `LIFE-28` reproduces an
  unguarded double index the API reference's own example guards.
- Constraints sections missing a real restriction: `LIFE-22` (entity extraction
  is not supported on the emulator), `LIFE-24` (which calendar permissions the
  kit needs), `LIFE-28` (the SDK version behind its API 24 requirement).
- A documented behaviour the code inverts: `LIFE-30`, where the document
  promises the queue discards the oldest tasks and the executor discards the
  newest.

## Constraints

- **This document has no sample project.** `industry_status.py` reports it as
  the only NO ZIP entry in the industry, and it was reviewed on the
  documentation and consistency axes only.
- **The redirect target is outside the industry tree**, so its content could not
  be verified from the local corpus - the findings above are about the link and
  the page, not about the FAQ's contents.
- The defect families above are drawn from the 267 findings recorded against
  this industry's 31 documents; each family entry names the cards where the
  pattern was confirmed with file and line evidence.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_2-0000002263237450
- Redirect target (HarmonyOS FAQ, phone):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faq-phone
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- ForEach (key generation rules):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- LazyForEach (key uniqueness, incremental notifications):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- @Link (type must match the data source):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- @Provide and @Consume (type-matching rule):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- Two-way sync with `$$`:
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- Image decoding (when to release ImageSource and PixelMap):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- hilog (`{public}` vs `{private}`, format-argument mapping, tag length):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
