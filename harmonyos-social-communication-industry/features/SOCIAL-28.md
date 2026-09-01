---
id: SOCIAL-28
title: App message centre - a route-map Navigation stack over one LazyForEach adapter shared between pages
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/28_app_message_list.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_message_list-0000002332602789
sample: huawei_industry_tree/14_social_communication/downloads/ApplicationMessageList.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [buffer, hilog, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0063, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when you need the **notification centre** of an app: a list of
conversations or channels, a system-messages inbox behind one of them, a
detail page per item, and a settings page. It is a template rather than a
feature - the sample ships no networking and no persistence - and its value
is in three structural decisions you can copy wholesale.

First, **`route_map.json` instead of `pageMap`**: destinations are declared
in a profile file with a `pageSourceFile` and an exported `@Builder`, so
`pushPathByName('SystemMessage', ...)` resolves a page that the caller never
imports. Pages become independently loadable, which is what you want when a
message centre grows a page per notification category.

Second, **an `IDataSource` adapter passed as a navigation parameter**. The
system-messages page does not fetch anything; it receives the already-loaded
adapter object from the message centre and renders it. One parse, two
screens, and the two lists stay in sync because they are literally the same
object.

Third, **heterogeneous rows from a `messageType` string** - the same JSON
array yields horizontal, vertical and image-stacked cards through three
`@Builder` functions selected by a discriminator.

The pattern generalises to any inbox-shaped screen: an order-status list, an
approvals queue, a school-notices app. What it does *not* give you is
read/unread state, deletion or badge computation - those are drawn, not
implemented.

## Feature checklist

- A launcher page with two buttons, into message settings and into the app
  message list.
- The app message list shows five channel rows: icon, title, timestamp,
  preview line, and a red unread badge that is hidden when the count is
  absent. Tapping a channel opens the message centre.
- The message centre shows a blue banner with a settings shortcut and a
  delete icon that hides the banner, then a system-messages entry with a red
  "88" badge, then a lazily rendered list of message groups; each group
  renders its own inner list of cards in one of three layouts.
- Tapping the system-messages entry pushes a page rendering the same adapter,
  grouped under time-point chips.
- Tapping any message card pushes a detail page with the subtitle, the
  release time, the image and the body.
- Message settings shows two rows, one with a chevron and one with a switch.
- Every page insets itself by the status-bar and navigation-bar heights.

## Architecture

One `entry` module. Six pages, two shared `@Builder` component files, one
adapter and one constants file.

```
entry/src/main/ets
├── common
│   ├── model
│   │   ├── ListItemAdapter.ets    generic IDataSource over a T[] - the only data layer
│   │   └── StyleConstants.ets     colours and the four nav titles
│   └── util/UtilClass.ets         SystemInformationData, MessageDetail, MessageDetailParam
├── components
│   ├── HeaderNav.ets              @Builder back-arrow + title bar, shared by four pages
│   └── Message.ets                VerticalNews / VerticalStackNews / HorizontalNews builders
├── entryability/EntryAbility.ets  full-screen window, avoid areas -> AppStorage, live updates
├── entrybackupability/
└── pages
    ├── MainPage.ets               @Entry: the Navigation root and the two entry buttons
    ├── AppListMessage.ets         the five channel rows (static, ForEach)
    ├── MessageCenter.ets          loads news_data.json, renders groups with LazyForEach
    ├── SystemMessage.ets          renders the adapter it receives as a route param
    ├── MessageDetail.ets          one message, from a route param
    └── MessageSetting.ets         two setting rows
```

`entry/src/main/resources/base/profile/route_map.json` declares five routes;
`main_pages.json` contains only `pages/MainPage`.

The documented tree matches the zip exactly.

**The design decision worth copying** is that the data is parsed once, in
`MessageCenter.aboutToAppear`, into a `ListItemAdapter`, and then *the
adapter object itself* is the navigation payload:

```typescript
this.pathStack.pushPathByName('SystemMessage', this.newsAdapter, true);
```

`SystemMessage` picks it up in `onReady` and assigns it straight to its own
`@State` field. Because `ListItemAdapter` is an `IDataSource`, both pages'
`LazyForEach`s register as listeners on the same instance, so a future
`notifyDataChange` would update both screens with no store, no event bus and
no re-parse. It is the cheapest correct way to share a list between two
destinations in a `Navigation` stack.

**The decision worth avoiding** is that `ListItemAdapter.addList` fires a
single `notifyDataAdd` for an N-item insert - the root of `HW-14-0063`. A
data source whose notifications do not describe the mutation is a data source
that only works because nothing is listening yet.

## Implementation steps

1. **Declare destinations in `route_map.json`,** each with a `name`, a
   `pageSourceFile` and a `buildFunction`, and export a matching
   `@Builder function XxxBuilder()` from each page file.
2. **Keep `main_pages.json` to the single entry page** and root everything
   else in one `Navigation(this.pathStack)`.
3. **Take the stack from `onReady`** in every `NavDestination`
   (`context.pathStack`), and read route parameters from
   `context.pathInfo.param` in the same callback - not from `aboutToAppear`,
   which runs before the destination context exists.
4. **Load the seed JSON with `getRawFileContentSync` + `buffer.from(...).toString()`**
   and `JSON.parse` into the typed array, inside a `try/catch`.
5. **Notify per inserted index on a bulk add** (`HW-14-0063`), or add a
   ranged notification; a single `notifyDataAdd(length - 1)` leaves any
   already-registered `LazyForEach` unaware of N-1 items.
6. **Derive the empty state from the adapter,** never from a boolean set true
   in `aboutToAppear` (`HW-14-0063`).
7. **Select the row layout from a `messageType` string** in the JSON, with a
   default branch, so unknown types degrade to the simplest card instead of
   rendering nothing.
8. **Publish the avoid-area heights from the ability** into `AppStorage` and
   inset each page with `@StorageProp`/`@StorageLink`, because
   `hideTitleBar(true)` plus full-screen layout means no one else will.

## Verified snippets

All snippets are from `ApplicationMessageList.zip`. Corrected forms are
marked.

**The adapter — `entry/src/main/ets/common/model/ListItemAdapter.ets`** (corrected, see `HW-14-0063`)

```typescript
export class ListItemAdapter<T> implements IDataSource {
  private listItems: T[] = [];
  private listeners: DataChangeListener[] = [];

  getList(): T[] {
    return this.listItems;
  }

  setList(list: T[]) {
    this.listItems = list;
  }

  addList(list: T[]) {
    const start = this.listItems.length;
    this.listItems = this.listItems.concat(list);
    // FIX: shipped code fires one notifyDataAdd(this.listItems.length - 1)
    for (let i = start; i < this.listItems.length; i++) {
      this.notifyDataAdd(i);
    }
  }

  totalCount(): number {
    return this.listItems.length;
  }

  getData(index: number): T {
    return this.listItems[index];
  }

  registerDataChangeListener(listener: DataChangeListener): void {
    if (this.listeners.indexOf(listener) < 0) {
      this.listeners.push(listener);
    }
  }

  unregisterDataChangeListener(listener: DataChangeListener): void {
    const pos = this.listeners.indexOf(listener);
    if (pos >= 0) {
      this.listeners.splice(pos, 1);
    }
  }

  notifyDataAdd(index: number): void {
    this.listeners.forEach(listener => {
      listener.onDataAdd(index);
    });
  }
  // notifyDataReload / notifyDataChange / notifyDataDelete / notifyDataMove follow the same shape
}
```

**The generic parameter is what makes this reusable,** and it is why the same
class serves the message centre and the system-messages page without a
subclass. `getList` / `setList` are the escape hatch for code that wants the
raw array - `MessageCenter` never uses them, but they are what a future
delete-a-message action would go through, and any such action must be paired
with `notifyDataDelete(index)` or the list will not update.

The shipped `addList` is correct about the *data* and wrong about the
*notification*: it concatenates N items and announces one. In this sample
nothing is listening yet - `aboutToAppear` runs before the `LazyForEach`
registers, so the first render calls `totalCount()` and sees everything - but
the moment a second batch arrives (pagination, a pull-to-refresh, a push
message) the list shows one new row out of N. Notifying per index is the
minimal fix; a ranged notification is better if the framework version offers
one.

**Loading the seed data — `entry/src/main/ets/pages/MessageCenter.ets`** (as shipped)

```typescript
import { buffer } from '@kit.ArkTS';

@Component
struct MessageCenter {
  pathStack: NavPathStack = new NavPathStack();
  @State deletePrompt: boolean = false;
  @State private newsAdapter: ListItemAdapter<SystemInformationData> = new ListItemAdapter();
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageLink('bottomRectHeight') bottomRectHeight: number = 0;

  aboutToAppear(): void {
    let context = this.getUIContext().getHostContext();
    let str: string = '';
    try {
      if (context) {
        let data: Uint8Array = context.resourceManager.getRawFileContentSync('news_data.json');
        str = buffer.from(data.buffer).toString();
        let jsonResult: SystemInformationData[] = JSON.parse(str) as SystemInformationData[];
        this.newsAdapter.addList(jsonResult);
      }
    } catch (e) {
      hilog.info(-1, '', JSON.stringify(e));
    }
  }
```

**`getRawFileContentSync` returns a `Uint8Array`, and `buffer.from(...)` is
the decode step** - `@kit.ArkTS`'s `buffer`, not a global. This is the
standard three-line idiom for shipping seed content as a rawfile: read,
decode, parse. Doing it synchronously in `aboutToAppear` is acceptable for a
small bundled file and wrong for anything user-scaled; the async variant
exists for that.

The `try/catch` wraps parse *and* read, which is right - a corrupt JSON and a
missing rawfile both surface as a caught throw rather than a crash. The
`catch` logging at `hilog.info(-1, ...)` rather than `hilog.error` is a
sloppiness worth fixing when you copy it.

Note the two different avoid-area decorators: `@StorageProp` for the top
inset and `@StorageLink` for the bottom. `@StorageLink` is two-way, and
nothing here writes back, so `@StorageProp` would be correct for both. Both
still track the ability's live `avoidAreaChange` updates.

**Heterogeneous rows from one array — same file** (as shipped)

```typescript
List() {
  LazyForEach(this.newsAdapter, (newsItem: SystemInformationData) => {
    ListItem() {
      Column() {
        Row() {
          Row() {
            Image(newsItem.titleImg).height(24).width(24);
            Text(newsItem.title).fontSize(18);
          };
          Text(newsItem.timePoint).fontSize(12).fontColor(StyleConstants.FONT_COLOR3);
        }
        .width('100%')
        .justifyContent(FlexAlign.SpaceBetween);

        List() {
          ForEach(newsItem.message, (item: MessageDetail) => {
            ListItem() {
              if (item.messageType === 'VerticalNews') {
                VerticalNews(item);
              } else if (item.messageType === 'VerticalStackNews') {
                VerticalStackNews(item);
              } else {
                HorizontalNews(item);
              }
            }
            .onClick(() => {
              this.pathStack.pushPathByName('MessageDetail',
                { detail: item, releaseTime: newsItem.timePoint } as MessageDetailParam);
            });
          });
        }
        .divider({ strokeWidth: '1vp' });
      };
    }
    .backgroundColor(Color.White)
    .borderRadius(16);
  });
}
.layoutWeight(1)
.scrollBar(BarState.Off);
```

**The layout choice is data, not code.** `messageType` is a plain string in
`news_data.json` and selects one of three exported `@Builder` functions -
`VerticalNews` (text over text-plus-thumbnail), `VerticalStackNews` (a
full-width image with the title stacked over its bottom edge), and
`HorizontalNews` as the `else` default. Because the fallback is the `else`
branch rather than a third `if`, an unrecognised type still renders
something, which is the correct failure mode for server-driven content.

The nested `List` inside a `LazyForEach` `ListItem` is the part to think
twice about. It works here because each group holds one or two messages, but
a nested scrollable inside a lazily rendered item defeats the outer list's
virtualisation for that subtree; with large groups, flatten to one list with
group headers instead.

The route parameter is composed inline as
`{ detail: item, releaseTime: newsItem.timePoint } as MessageDetailParam`,
which is how the detail page gets both the message and the group's timestamp
without a lookup.

**The empty state — `entry/src/main/ets/pages/SystemMessage.ets`** (corrected, see `HW-14-0063`)

```typescript
@Component
struct SystemMessage {
  pathStack: NavPathStack = new NavPathStack();
  @State systemMessageAdapter: ListItemAdapter<SystemInformationData> = new ListItemAdapter();

  // FIX: shipped code has `@State isShowMessage: boolean = false` set to true
  //      in aboutToAppear and never set false again - the empty view is dead code.
  build() {
    NavDestination() {
      Column() {
        NavHeader(StyleConstants.NAV3, () => {
          this.pathStack.pop();
        });
        if (this.systemMessageAdapter.totalCount() === 0) {      // FIX: derive from the data
          Column() {
            Image($r('app.media.not_message')).width(120);
            Text($r('app.string.SystemMessage_Text_0'))
              .fontColor(StyleConstants.FONT_COLOR3);
          }
          .layoutWeight(1)
          .justifyContent(FlexAlign.Center);
        } else {
          List() {
            LazyForEach(this.systemMessageAdapter, (newsItem: SystemInformationData) => {
              // ... the time-point chip and the inner message list
            });
          }
          .layoutWeight(1)
          .scrollBar(BarState.Off);
        }
      }
      .padding({ top: this.topRectHeight, bottom: this.bottomRectHeight });
    }
    .hideTitleBar(true)
    .onReady((context: NavDestinationContext) => {
      this.pathStack = context.pathStack;
      this.systemMessageAdapter = context.pathInfo.param as ListItemAdapter<SystemInformationData>;
    });
  }
}
```

**`onReady` is the only place the route parameter is available.** It runs
after `aboutToAppear` and carries the `NavDestinationContext`, from which both
the stack and `pathInfo.param` come. Assigning the parameter to an `@State`
field is what re-triggers the build with real data - and it is also why the
shipped `isShowMessage` flag can never work: it is decided in
`aboutToAppear`, before the adapter has arrived, so it is a decision made
without the information it needs.

The designed empty view - a `not_message` illustration and a "no messages"
string, both shipped in the resources - is therefore unreachable. Deriving
the condition from `totalCount()` costs nothing, makes the first-run boundary
correct, and keeps working when a delete empties the list.

The title bar itself is a global `@Builder` in `components/HeaderNav.ets`
taking `(title, event?, top?, bottom?)` - no state, no component instance,
and each page passes its own status-bar inset as the `top` margin.
`SystemMessage` is the exception: it pads its `Column` instead of passing
`top`, which is worth normalising when you copy the header.

## Permissions & config

**None.** The sample declares no `requestPermissions` - all content is
bundled in `resources/rawfile`.

The two profile files carry the routing:

```json5
// resources/base/profile/main_pages.json
{ "src": ["pages/MainPage"] }

// resources/base/profile/route_map.json
{ "routerMap": [
  { "name": "AppListMessage", "pageSourceFile": "src/main/ets/pages/AppListMessage.ets",
    "buildFunction": "AppListMessageBuilder" },
  { "name": "MessageCenter",  "pageSourceFile": "src/main/ets/pages/MessageCenter.ets",
    "buildFunction": "MessageCenterBuilder" }
  // MessageSetting, SystemMessage, MessageDetail follow
]}
```

`module.json5` carries `"routerMap": "$profile:route_map"`, which is what
lets `pushPathByName` resolve; the `buildFunction` name has to match the
exported `@Builder` exactly, and a mismatch fails silently as a push that
does nothing.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **This is a UI template.** There is no networking, no persistence, no
  read/unread state and no delete. The trash icon on the banner only sets
  `deletePrompt`, which hides the banner itself; the "88" system-message
  badge and the per-channel counts are literals.
- `news_data.json` gives `titleImg` as the string
  `"resources/rawfile/messageCenter/fullName_expansion.png"` and feeds it
  straight into `Image(newsItem.titleImg)`. That is neither a `$rawfile`
  reference nor a URL, so the group icons do not resolve; use
  `$rawfile('messageCenter/...')` or store a resource id.
- `AppListMessage` and `MessageSetting` build their rows from hardcoded
  Chinese literals in `aboutToAppear`, so those two pages are not
  localisable, and every `ForEach` in the sample omits the key generator.
- `MainPage` reaches `AppListMessage` and `MessageSetting`; `MessageCenter`
  is only reachable by tapping a channel row. `MessageDetail` renders the
  same image and body text twice, by design of the mock, to fill the scroll.

## Pitfalls

- **`HW-14-0063`** (B/low, confirmed): two related defects. `SystemMessage`
  sets `isShowMessage = true` unconditionally in `aboutToAppear` and never
  clears it, so the designed "no messages" view - illustration and string,
  both shipped - is dead code and an empty inbox renders as a blank page.
  And `ListItemAdapter.addList` concatenates N items but fires a single
  `notifyDataAdd(length - 1)`, so any already-registered `LazyForEach` misses
  N-1 items. Fix: derive the empty state from `totalCount()`, and notify per
  inserted index (or use a ranged notification).
- **Route parameters are not available in `aboutToAppear`.** Read them in
  `onReady` from `context.pathInfo.param`; any decision that depends on them
  must be made after that, which is exactly what the `isShowMessage` flag got
  wrong.
- **A `route_map.json` typo fails silently.** A wrong `buildFunction` or
  `pageSourceFile` makes `pushPathByName` a no-op rather than an error.
- **`@StorageLink('bottomRectHeight')` is two-way for a read-only value.**
  Use `@StorageProp` unless the page really writes the inset back.
- **The nested `List` inside a `LazyForEach` item** works for one-or-two
  message groups and will not scale; flatten to a single list with group
  headers if groups can be long.
- **Group icons never load** because `titleImg` in the seed JSON is a plain
  file path string rather than a `$rawfile` reference.

## References

- `huawei_industry_tree/14_social_communication/docs/28_app_message_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_message_list-0000002332602789
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, `DataChangeListener`, the notify contract and bulk-change guidance
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItem`, `divider`, `scrollBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `NavDestination`, `onReady`, `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `pushPathByName` and the `route_map.json` declaration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navdestination.md` - reading `pathInfo.param` and the destination lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navdestination
- `SOCIAL-27` - `BasicDataSource<T>`, the same `IDataSource` boilerplate factored into an abstract base
