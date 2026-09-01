---
id: JOBS-04
title: Job detail to recruiter to company job list - a three-page Navigation flow driven by routerMap
industry: 12_jobs
doc: huawei_industry_tree/12_jobs/docs/04_recruitment_and_job_list_display.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/recruitment_and_job_list_display-0000002456685322
sample: huawei_industry_tree/12_jobs/downloads/RecruitmentInformation.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Navigation, NavPathStack, NavDestination, NavDestinationContext, pushPathByName, pop, onReady, routerMap, List, ListItem, ForEach, "@Prop", "@StorageProp", Divider, Blank, Stack, "window.getWindowAvoidArea", "UIContext.px2vp"]
permissions: []
min_api: 17
modules: [entry]
findings: [HW-12-0004, HW-12-0005]
status: verified-with-fixes
---

## When to use

Load this card when you need a **detail page that branches to two related
pages** and want the routing declared outside the code that navigates. Here:
a job posting, whose recruiter card leads either to the recruiter's profile or
to every job that recruiter has posted. The same three-page shape is a product
page linking to the seller and the seller's catalogue, an article linking to the
author and the author's feed, a listing linking to the agent and their
portfolio.

The transferable part is the **`routerMap` profile plus exported `@Builder`
functions**. Destinations are named in `resources/base/profile/route_map.json`
with the source file and the builder that constructs them; the calling page
only ever says `pushPathByName('JobListPage', null)`. There is no import from
the caller to the destination, which is what lets destinations move to another
module later without touching the pages that push them.

This is the plainest of the three jobs samples — no gestures, no animation, no
permissions. Read it for the routing wiring and for the layout idioms
(`Blank()` to pin a footer button, avoid areas converted at the ability), not
for state management: every page's content is a hardcoded literal.

## Feature checklist

- A job detail page: title, location/experience/education row, skill tags, and
  the 岗位职责 (responsibilities) and 任职要求 (requirements) lists.
- A recruiter card in the middle of the detail page, with avatar, name,
  employer, and a response-time line.
- Tapping the recruiter's avatar opens their profile page.
- Tapping the chevron at the end of the recruiter row opens their company and
  job list page.
- The job list page shows a hero image, the recruiter over it, a company card,
  and a scrollable list of that company's postings with divider rows between
  them.
- Each posting shows title, salary, and three chips (location, experience,
  education).
- Both destination pages have a back arrow that pops the stack.
- The 立即沟通 (chat now) button on the detail page is a demo toast.

## Architecture

One `entry` module, three pages, one constants file that doubles as the data
source. No model layer and no repository.

```
entry/src/main/ets
├── constants/StyleConstants.ets           StyleConstants + RESPONSIBILITIES/REQUIRE + JobType + JOB_LIST
├── entryability/EntryAbility.ets          full-screen window, avoid areas converted to vp -> AppStorage
├── entrybackupability/
└── pages
   ├── MainPage.ets                        @Entry: the Navigation host and the job detail
   ├── JobListPage.ets                     NavDestination: company card + JobList + JobListItem
   └── PersonalIntroductionPage.ets        NavDestination: the recruiter's profile
```

The documented 工程目录 matches the zip's `ets` tree, but it stops at
`entry/src/main/resources` with no detail — and the file that makes this sample
work, `resources/base/profile/route_map.json`, is inside that folder. A reader
following the document alone would wire the destinations by import and never
discover the `routerMap` mechanism the sample is demonstrating.

**The design decision worth copying** is that each destination is a plain
`@Component` plus an exported `@Builder` function in the same file:

```typescript
@Builder
export function jobListPageBuilder() {
  JobListPage()
}
```

`route_map.json` names `JobListPage`, points at the file, and names
`jobListPageBuilder`. The framework loads the module lazily on first push. The
struct itself stays un-exported, so the only public surface of the file is the
builder — nothing else can construct a destination page by accident.

**The design decision worth avoiding** is `StyleConstants.ets`. It holds two
constants, two string arrays of body copy, an interface, and a seven-entry
data table. That is four unrelated concerns in a file whose name promises
styling. Both other pages import their content from it. Split the data out
before you copy this structure.

## Implementation steps

1. **Host the flow in `Navigation(this.pathStack)`** on the entry page, with
   `hideTitleBar(true)` since the sample draws its own back arrows.
2. **Declare the destinations in `route_map.json`** under
   `resources/base/profile/`, each with `name`, `pageSourceFile` and
   `buildFunction`, and point `module.json5` at it with
   `"routerMap": "$profile:route_map"`.
3. **Export a `@Builder` function per destination** whose only body is the
   struct, and keep the struct itself un-exported.
4. **Push by name**: `this.pathStack.pushPathByName('JobListPage', null)`.
   Pass `null` when the destination reads its data from a shared source, as
   here; otherwise pass the payload as the second argument.
5. **Capture the real stack in `onReady`**:
   `.onReady((context: NavDestinationContext) => { this.pathStack = context.pathStack })`.
   The field initialiser `new NavPathStack()` is a placeholder that would pop
   nothing.
6. **Convert avoid areas to vp at the ability** and publish vp into
   `AppStorage`, so every page can add `topRectHeight` to its padding without
   converting.
7. **Pin the footer button with `Blank()`** rather than absolute positioning:
   it absorbs the leftover column height and pushes 立即沟通 to the bottom.
8. **Keep `fontWeight` inside [100, 900] in steps of 100** — the sample ships
   `fontWeight(40)`, which is silently replaced by 400 (`HW-12-0004`).

## Verified snippets

All snippets are from `RecruitmentInformation.zip`. Corrected forms are marked.

**The navigation host and the two branches — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct MainPage {
  pathStack: NavPathStack = new NavPathStack();
  @StorageProp('topRectHeight') topRectHeight: number = 0
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0

  @Builder
  headhunter() {
    Row() {
      Image($r('app.media.small_avatar'))
        .width(50)
        .height(50)
        .onClick(()=>{
          this.pathStack.pushPathByName('PersonalIntroductionPage',null)
        })
      Column({ space: 5 }) {
        Text('张先生')                       // the recruiter's name
        Text('华为技术研究所 招聘者')          // employer + role
        Text('2分钟前回复 今日回复10+次')      // "replied 2 min ago, 10+ replies today"
      }
      .alignItems(HorizontalAlign.Start)

      Blank()

      Image($r('app.media.right'))
        .onClick(()=>{
          this.pathStack.pushPathByName('JobListPage',null)
        })
    }
    .width('100%')
  }

  build() {
    Navigation(this.pathStack) {
      Column() {
        // back arrow, this.titleItem(), Divider, this.headhunter(), Divider, this.jobDetails()
        Blank()
        Button('立即沟通')
          .margin({ bottom: this.bottomRectHeight + 12 })
      }
      .padding({ top: this.topRectHeight + 12, left: 16, right: 16 })
    }
    .hideTitleBar(true)
  }
}
```

**Two tap targets in one row lead to two different destinations**, and neither
knows anything about the page it opens — only its registered name. That is the
whole point of `routerMap`: `MainPage.ets` has no import of `JobListPage.ets`,
so the two files could live in different modules tomorrow.

The `@StorageProp` values are added directly to padding with no `px2vp` call,
because this sample's ability already converted them (see the last snippet).
`Blank()` between the content and the button is what pins the button to the
bottom of a `Column` that is otherwise content-height: it expands to fill
whatever vertical space is left. Adding `bottomRectHeight` to its bottom margin
keeps it clear of the navigation indicator.

**Registering the destinations — `entry/src/main/resources/base/profile/route_map.json`** (as shipped)

```json
{
  "routerMap": [
    {
      "name": "PersonalIntroductionPage",
      "pageSourceFile": "src/main/ets/pages/PersonalIntroductionPage.ets",
      "buildFunction": "personalIntroductionPageBuilder",
      "data": { "description": "this is PersonalIntroductionPage" }
    },
    {
      "name": "JobListPage",
      "pageSourceFile": "src/main/ets/pages/JobListPage.ets",
      "buildFunction": "jobListPageBuilder",
      "data": { "description": "this is JobListPage" }
    }
  ]
}
```

`pageSourceFile` is **module-relative, not `ets`-relative** — it starts at
`src/main/`, which is the single most common thing to get wrong here; a wrong
path fails at push time, not at build time. `buildFunction` must name a
function that is `export`ed and decorated `@Builder`, and it takes no
arguments: any payload passed to `pushPathByName` reaches the destination
through `NavDestinationContext`, not through the builder signature. `data` is a
free-form bag the framework carries along and this sample never reads.

The `module.json5` side is one line: `"routerMap": "$profile:route_map"`.

**A destination and its back arrow — `entry/src/main/ets/pages/JobListPage.ets`** (as shipped)

```typescript
@Builder
export function jobListPageBuilder() {
  JobListPage()
}

@Component
struct JobListPage {
  @StorageProp('topRectHeight') topRectHeight: number = 0
  pathStack: NavPathStack = new NavPathStack()

  build() {
    NavDestination() {
      Stack({ alignContent: Alignment.Top }) {
        Image($r('app.media.img'))
          .width('100%')

        Column() {
          Row() {
            Image($r('app.media.back2'))
              .onClick(() => {
                this.pathStack.pop()
              })
          }
          .width('100%')

          this.titleItem()
          this.companyItem()
          JobList()
        }
        .backgroundColor('rgba(0,0,0,0.05)')
        .padding({ top: this.topRectHeight + 12, left: 16, right: 16 })
      }
      .width('100%')
      .height('100%')
    }
    .hideTitleBar(true)
    .onReady((context: NavDestinationContext) => {
      this.pathStack = context.pathStack;
    })
  }
}
```

**`onReady` is not optional.** The field initialiser `new NavPathStack()`
creates a detached stack; popping it does nothing and the back arrow would be
dead. `onReady` hands the destination the stack it was actually pushed onto,
and it fires before the first frame, so the arrow works from the first tap.
The alternative — passing the parent's stack as a parameter — would defeat the
decoupling `routerMap` bought.

The hero image is the first child of a top-aligned `Stack` and the content
column is drawn over it with a translucent scrim
(`rgba(0,0,0,0.05)`); the recruiter's name is white because it sits on the
image, which is why `titleItem()` here differs from the one on the detail page.

**The list and its rows — same file** (corrected, see `HW-12-0004`)

```typescript
@Component
struct JobList {
  @State jobArr: JobType[] = JOB_LIST

  build() {
    Column() {
      Text('发布职位')                     // "posted jobs"
      Divider().strokeWidth(0.5).color('rgba(0,0,0,0.2)')

      List() {
        ForEach(this.jobArr, (item: JobType, index: number) => {
          ListItem() {
            JobListItem({ data: item })
          }

          if (index < this.jobArr.length - 1) {
            ListItem() {
              Divider()
                .strokeWidth(0.5)
                .color('rgba(0,0,0,0.2)')
                .margin({ top: 18, bottom: 18 })
            }
          }
        })
      }
      .layoutWeight(1)
      .scrollBar(BarState.Off)
    }
    .layoutWeight(1)
    .backgroundColor(Color.White)
    .borderRadius(12)
  }
}

// companyItem(), same file:
Text('已上市 10000+')                      // "listed company, 10000+ employees"
  .fontSize(14)
  .fontWeight(400)                         // FIX: shipped as fontWeight(40)
  .fontColor('rgba(0, 0, 0, 0.6)')
```

**`layoutWeight(1)` twice is what makes the list scroll.** The inner `List`
takes the space left inside the white card, and the card itself takes the space
left in the page column. Without the outer weight the `List` would be
content-height and the page would clip instead of scrolling. `BarState.Off`
hides the scrollbar because the card already has a visible boundary.

The separators are **`ListItem`s containing a `Divider`**, conditionally
emitted for every row but the last. `List` has a `.divider()` attribute that
does exactly this in one line, without doubling the item count or making every
other item a focusable, selectable row — `JOBS-02`'s job list uses it. Prefer
that; this form is here only because the divider needs 18 vp of margin on each
side, which `.divider()` expresses as `startMargin`/`endMargin`.

`fontWeight(40)` is not a legal weight. The `Text` reference is explicit: the
numeric range is [100, 900] in steps of 100, and a value that is not a multiple
of 100 falls back to 400. So the shipped line renders as 400 — the intent was
almost certainly `400` with a dropped zero — and copying it propagates an
invalid API value that no compiler will flag (`HW-12-0004`).

**Avoid areas converted at the source — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
windowStage.getMainWindow((err: BusinessError, data) => {
  windowClass = data;
  let uiContext: UIContext | null = null;
  uiContext = windowClass.getUIContext();

  let isLayoutFullScreen = true
  windowClass.setWindowLayoutFullScreen(isLayoutFullScreen)

  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR
  let avoidArea = windowClass.getWindowAvoidArea(type)
  AppStorage.setOrCreate('bottomRectHeight', uiContext.px2vp(avoidArea.bottomRect.height))

  type = window.AvoidAreaType.TYPE_SYSTEM;
  avoidArea = windowClass.getWindowAvoidArea(type)
  AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(avoidArea.topRect.height))

  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', uiContext?.px2vp(data.area.topRect.height))
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', uiContext?.px2vp(data.area.bottomRect.height))
    }
  });
})

```

**This is the variant of the boilerplate worth copying** across the three jobs
samples. `AppStorage` holds **vp**, converted once at the ability, so three
pages consume the value with a bare `this.topRectHeight + 12` and cannot get
the units wrong. `JOBS-02` stores px and converts in the page — correct, but it
must be repeated at every read site. `JOBS-03` stores px and then tries to
convert into its own `@StorageProp`, which silently does not work.

The reason this version needs the async `getMainWindow` callback rather than
`getMainWindowSync` is `getUIContext()`: it needs a window that has finished
being created. Note that `avoidAreaChange` is still never released in
`onWindowStageDestroy`.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

The two entries that matter are `"pages": "$profile:main_pages"` (which lists
only `pages/MainPage` — the destinations are deliberately not pages) and
`"routerMap": "$profile:route_map"`.

`deviceTypes` is `phone`, `tablet`, `2in1`, `wearable`. The wearable entry is
not credible: the layout assumes 16 vp side padding, 50 vp avatars and a
full-width hero image, and nothing in the pages branches on device type.

`compatibleSdkVersion` is `5.0.5(17)`, `targetSdkVersion` `5.1.0(18)` — the
oldest baseline of the three jobs samples.

## Constraints

- API Version 17 Release or later; HarmonyOS 5.0.5 Release SDK or later;
  DevEco Studio 5.0.5 Release or later.
- Every string on all three pages is a hardcoded Chinese literal in the source
  — job titles, the recruiter's name, the company, the body copy. There is no
  localisation path at all: `resources/base/element/string.json` carries only
  the module boilerplate.
- `MainPage`'s content column is not scrollable. The responsibilities and
  requirements lists are three items each and fit; a real posting would
  overflow. Wrap the column in a `Scroll` before using this layout with real
  data.
- `PersonalIntroductionPage` is four widgets — avatar, name, role, employer.
  It exists to demonstrate the second push target, not a profile.
- The job list is seven near-identical entries in `JOB_LIST`; the chevron on
  the company card is decorative and navigates nowhere.
- `windowClass` is declared `window.Window | undefined` and then dereferenced
  inside the callback without a guard; `uiContext` is nullable and used with
  both `.` and `?.` in the same block. Tighten both if you lift this ability.

## Pitfalls

- **`HW-12-0004`** (B/low, confirmed): `JobListPage.ets` renders
  `Text('已上市 10000+').fontSize(14).fontWeight(40)`. The `Text` reference
  restricts the numeric form to [100, 900] at intervals of 100 and falls back
  to 400 for anything else, so the intended lighter weight is silently ignored
  and an invalid API value is propagated to anyone copying the sample. Fix:
  `fontWeight(400)`, or `fontWeight(FontWeight.Normal)`.

## References

- `huawei_industry_tree/12_jobs/docs/04_recruitment_and_job_list_display.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recruitment_and_job_list_display-0000002456685322
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` — `Navigation`, `NavPathStack`, `pushPathByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` — `List`, `divider`, `scrollBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` — the `fontWeight` range
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navdestination.md` — `NavDestination` and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-set-navigation-routing.md` — the `routerMap` profile and `buildFunction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-set-navigation-routing
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` — `getWindowAvoidArea`, `avoidAreaChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `JOBS-02` — the same job-row layout built with `List.divider()` instead of divider items
- `JOBS-03` — the deck this list is the flat counterpart of
