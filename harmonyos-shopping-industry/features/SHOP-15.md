---
id: SHOP-15
title: Browsing history that survives a restart - AppStorage as the live list, PersistentStorage as the disk copy
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/15_offering_browsing_history.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/offering_browsing_history-0000002361003509
sample: huawei_industry_tree/16_shopping/downloads/OfferingBrowsingHistory.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, "AppStorage.get", "AppStorage.setOrCreate", "PersistentStorage.persistProp", "@StorageLink", "@StorageProp", "@Provide", "@Consume", Navigation, NavPathStack, NavDestination, pushPathByName, getParamByName, WaterFlow, FlowItem, Tabs, BottomTabBarStyle]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0017, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a feature needs **one list that several unrelated screens
read and write, and that must still be there after the app is killed** -
recently viewed products, recent searches, a "continue watching" row, the last
few addresses used at checkout. The shopping case is the 我的足迹 (my
footprints) page: tapping a product card anywhere in the app prepends it to a
history array, and the footprints page renders whatever that array currently
holds, including after a cold start.

The pattern is two APIs stacked. `AppStorage` is the in-memory, UI-observable
store: any component can `@StorageLink` a key and get two-way binding without
a provider above it in the tree. `PersistentStorage.persistProp` then marks
one of those keys as disk-backed, so the value is read from disk at startup and
written back on every change. The application code never touches a file.

**The card is as much a warning as a recipe.** `PersistentStorage` serialises
synchronously on the UI thread and its own guide caps the recommendation at
2 KB, while a browsing history is unbounded by construction - and this sample
adds neither dedup nor a length cap (`HW-16-0017`). Take the `AppStorage`
half, which is genuinely the right tool; treat the `PersistentStorage` half as
correct only while the list is short and small, and read the "worth avoiding"
paragraph below before persisting anything containing `$r(...)` resources.

## Feature checklist

- A three-tab bottom bar; the personal-centre tab holds a 我的足迹 row with a
  clock icon and a chevron.
- Tapping that row pushes the footprints page onto the `NavPathStack`.
- Tapping any product card in the waterfall records the product at the head of
  the history and pushes the product detail page.
- The footprints page lists the recorded products newest first, with image,
  title, seller, price and a buy button.
- The buy button on a history row re-opens the same product's detail page.
- The back arrow pops the footprints page.
- History survives an app restart: reopening the app and the footprints page
  shows the same list.

## Architecture

One `entry` module. Two of the three "pages" are `NavDestination` components
resolved through the route map, not `@Entry` pages.

```
entry/src/main/ets
├── components
│   ├── CardInfoCom.ets       one product card; its onClick is the history writer
│   ├── ContentFlowCom.ets    the two-column WaterFlow of cards
│   ├── MyFootprints.ets      the footprints NavDestination + the persistProp call
│   └── ProductDetails.ets    the product detail NavDestination
├── entryability/EntryAbility.ets     avoid areas -> AppStorage, full screen
├── entrybackupability/EntryBackupAbility.ets
├── model/FoodInfoModel.ets   FoodInfo, TabBarItem, CARD_INFO (6 products)
├── pages/Pages.ets           @Entry: Navigation + the three bottom Tabs
└── utils/Logger.ets
```

`resources/base/profile/route_map.json` registers `MyFootprints` and
`ProductDetails` against their `@Builder` entry functions, which is why
`Pages.ets` is the only file under `pages/`. The documented tree matches the
zip.

**The design decision worth copying** is that the history has no owner. There
is no store class, no service, no `@Provide` chain: `CardInfoCom` writes the
key from inside an `onClick`, and `MyFootprints` reads it with a
`@StorageLink`, and the two components never meet. `AppStorage` is the only
thing between them, and it is reachable from any depth without threading a
reference through the component tree - which is exactly what makes it right
for cross-cutting state like this. Contrast `SHOP-13`, where an
`@Provide`/`@Consume` pair does the same job for order statuses: that works
because provider and consumers share an ancestor, which the card and the
footprints page here do not.

**And the decision worth avoiding**: the `persistProp` call is a bare
statement at module scope in `MyFootprints.ets`:

```typescript
PersistentStorage.persistProp('historyArr', undefined);
```

That file is only loaded when the router first resolves `MyFootprints`. Until
the user opens the footprints page, the key is a plain `AppStorage` entry, so
history written before the first visit is not written to disk. Registration
belongs in `EntryAbility.onCreate` or at the top of the `@Entry` page - somewhere
guaranteed to run once, early, regardless of navigation. Passing `undefined` as
the default value is the second half of the problem: the guide's contract is a
typed default used when no persisted value exists, and it is what forces every
reader in this sample to test `!== undefined` on a field they declared as
`FoodInfo[]`.

## Implementation steps

1. **Register the persisted key once, at startup** - not at module scope in a
   lazily routed component - and give it a real typed default (`[]`), not
   `undefined`.
2. **Declare the live binding** wherever the list is rendered:
   `@StorageLink('historyArr') historyArr: FoodInfo[] = [];`. The `@StorageLink`
   is two-way, so a write from anywhere re-renders this list.
3. **Write from the card's `onClick`,** prepending with `unshift` so newest is
   first, and write the array back with `AppStorage.setOrCreate` so the change
   is observed.
4. **Deduplicate by product identity before prepending,** and cap the length
   (`HW-16-0017`); both are missing in the sample and both are what keep the
   persisted blob inside the documented 2 KB budget.
5. **Move to `@ohos.data.preferences` or `relationalStore` if the list can
   grow** - the `PersistentStorage` guide routes "substantial data" there
   explicitly, because the write is synchronous and on the UI thread
   (`HW-16-0017`).
6. **Push detail pages by name** with the item as the parameter
   (`pushPathByName('ProductDetails', this.info, true)`) and read it back with
   `getParamByName(...)[0]`.
7. **Do not put `$r(...)` resource handles in the persisted model** - see
   Constraints; store ids or paths and resolve them at render time.

## Verified snippets

All snippets are from `OfferingBrowsingHistory.zip`. Corrected forms are marked.

**The write — `entry/src/main/ets/components/CardInfoCom.ets`** (as shipped)

```typescript
@Component
export struct CardInfoCom {
  @Consume pageInfos: NavPathStack;
  @State info: FoodInfo = { picture: '', title: '', nickname: '', likeCount: 0,
    isLike: false, price: '', priceNumber: 0, technology: [], taste: [], subheading: '' };

  build() {
    Column({ space: 8 }) {
      Image(this.info.picture).size({ width: '100%', height: 180 }).objectFit(ImageFit.Cover);
      Column() {
        Text(this.info.title).maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis });
        Text(this.info.nickname).maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis });
        Image(this.info.price).height(24);          // the price is a picture, not a number
      }
      .alignItems(HorizontalAlign.Start);
    }
    .onClick(() => {
      if (AppStorage.get('historyArr') === undefined) {
        let historyArr: FoodInfo[] = [];
        historyArr.unshift(this.info);
        AppStorage.setOrCreate('historyArr', historyArr);
      } else {
        let historyArr: FoodInfo[] | undefined = AppStorage.get('historyArr');
        historyArr?.unshift(this.info);
        AppStorage.setOrCreate('historyArr', historyArr);
      }
      this.pageInfos.pushPathByName('ProductDetails', this.info, true);
    })
    .clip(true);
  }
}
```

**The `setOrCreate` after the `unshift` is not redundant.** `AppStorage.get`
returns the stored array by reference, so `unshift` has already mutated the
value in the store; what the second call does is re-publish it, which is what
notifies `@StorageLink` subscribers and triggers the persist write. Drop that
line and the footprints page updates only when something else forces a rebuild,
and nothing reaches disk.

The two branches differ only in the first line, and exist purely because
`persistProp` was seeded with `undefined`. Seed the key with `[]` and the whole
handler collapses to a read, a dedup, a prepend, a truncate and one
`setOrCreate` - which is also the shape the fix for `HW-16-0017` wants:

```typescript
// corrected — see HW-16-0017
.onClick(() => {
  const history: FoodInfo[] = AppStorage.get<FoodInfo[]>('historyArr') ?? [];
  const deduped = history.filter((item: FoodInfo) => item.title !== this.info.title);
  deduped.unshift(this.info);
  AppStorage.setOrCreate('historyArr', deduped.slice(0, 20));   // cap the persisted blob
  this.pageInfos.pushPathByName('ProductDetails', this.info, true);
})
```

`FoodInfo` carries no id, so the dedup key has to be `title` here; give the
model a real id before shipping this. The `slice(0, 20)` is the line that keeps
the serialised value near the guide's 2 KB advice.

**The read — `entry/src/main/ets/components/MyFootprints.ets`** (as shipped)

```typescript
import { FoodInfo } from '../model/FoodInfoModel';

PersistentStorage.persistProp('historyArr', undefined);   // module scope: runs on first route resolve

@Builder
export function myFootprintsBuilder() {
  MyFootprints();
}

@Component
struct MyFootprints {
  @Consume pageInfos: NavPathStack;
  @StorageLink('historyArr') historyArr: FoodInfo[] = [];

  build() {
    NavDestination() {
      Column() {
        Row() {
          Image($r('app.media.ic_leftarrowhead'))
            .onClick(() => { this.pageInfos.pop(); });
          Text($r('app.string.tab_3'));               // 我的足迹
        }
        .height(56);

        if (this.historyArr !== undefined) {
          Text($r('app.string.tab_4'));               // the section heading
        }

        List() {
          ForEach(this.historyArr, (item: FoodInfo) => {
            ListItem() {
              Row() {
                Image(item.picture).width(78).height(78).borderRadius(8);
                Column() {
                  Text(item.title).maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis });
                  Text(item.nickname).maxLines(1);
                  Row() {
                    Text() {
                      Span($r('app.string.tab_6'));    // the ￥ prefix
                      Span(item.priceNumber + '');
                    }
                    Image($r('app.media.ic_button'))
                      .onClick(() => {
                        this.pageInfos.pushPathByName('ProductDetails', item, true);
                      });
                  }
                  .justifyContent(FlexAlign.SpaceBetween);
                }
                .layoutWeight(1);
              };
            }
            .height(110);
          });
        };
      };
    }
    .hideTitleBar(true);
  }
}
```

**`@StorageLink` is the whole subscription.** No `aboutToAppear` fetch, no
refresh on `onPageShow` - the declaration binds the component to the key, and
a `setOrCreate` from a card three screens away re-renders this list. (The
sample does declare an empty `aboutToAppear` next to it, which is dead code and
a hint that an explicit load was expected and turned out unnecessary.)

Note that the history row shows `item.priceNumber` as text, while the product
card shows `item.price` - an **image** resource. The same record renders its
price two different ways because the card's price is a pre-rendered picture. A
model that carries a number and a picture of that number will drift; the
history page is where you notice.

`if (this.historyArr !== undefined)` guards the heading against a field whose
declared type is `FoodInfo[]`. It is only reachable because `persistProp` was
given an `undefined` default; with `[]` the check would be `.length > 0`,
which is what you actually want - an empty-state instead of a bare heading
over an empty list.

**The host page — `entry/src/main/ets/pages/Pages.ets`** (as shipped)

```typescript
@Entry
@Component
struct Pages {
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @Provide pageInfos: NavPathStack = new NavPathStack();
  @State currentIndex: number = 1;

  onPageShow(): void {
    this.currentIndex = 2;
  }

  build() {
    Navigation(this.pageInfos) {
      Tabs({ barPosition: BarPosition.End, index: this.currentIndex, controller: this.tabsController }) {
        // ... tab 1 and tab 2 are empty TabContents ...
        TabContent() {
          // 个人中心: the 我的足迹 row
          Row() { /* clock icon + tab_1 label + chevron */ }
            .onClick(() => {
              this.pageInfos.pushPathByName('MyFootprints', true);
            })
        }
        .tabBar(new BottomTabBarStyle($r('app.media.tab3'), '个人中心'));
      }
      .onContentWillChange(() => {
        return false;                 // vetoes every tab switch
      });
    }
    .hideTitleBar(true)
    .safeAreaPadding({ top: this.uiContext.px2vp(this.topRectHeight) });
  }
}
```

`@Provide pageInfos` is the one thing the page publishes, and both
`NavDestination`s plus every card `@Consume` it - a stack is a legitimate use
of provide/consume because it genuinely has a single owner and a single
lifetime.

Two lines here are traps rather than lessons. `onContentWillChange` returning
`false` vetoes every tab switch, so the bottom bar is decorative; the sample
compensates by forcing `currentIndex = 2` in `onPageShow` to land on the
personal-centre tab. And `safeAreaPadding` is applied to the `Navigation`, so
the `NavDestination`s inherit the top inset - which is why `MyFootprints` needs
no avoid-area handling of its own.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`resources/base/profile/route_map.json` is the interesting piece of config:

```json5
{
  "routerMap": [
    { "name": "MyFootprints",   "pageSourceFile": "src/main/ets/components/MyFootprints.ets",
      "buildFunction": "myFootprintsBuilder", "data": {} },
    { "name": "ProductDetails", "pageSourceFile": "src/main/ets/components/ProductDetails.ets",
      "buildFunction": "productDetailsBuilder", "data": {} }
  ]
}
```

Route-map entries are resolved lazily: the module is loaded the first time its
name is pushed. That is a feature for startup time and the direct cause of the
`persistProp` placement problem described above.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`PersistentStorage` writes synchronously on the UI thread** and its guide
  recommends keeping persisted values under 2 KB, directing larger data to the
  database APIs. A `FoodInfo` record carries two string arrays and several
  resource handles, so a few dozen entries already exceed that.
- **Persisted `$r(...)` handles are fragile.** `FoodInfo.picture`, `title`,
  `nickname` and `price` are `ResourceStr` values produced by `$r`. What is
  serialised is a resource descriptor, not the content; after a version bump
  that renumbers resources, or across a locale change, a restored record can
  point somewhere else. Persist stable ids and resolve to resources at render
  time.
- **There is no clear-history action.** The footprints page renders and
  navigates; nothing deletes. Add one before shipping - a persisted history a
  user cannot erase is a privacy problem, not just a missing feature.
- The first two bottom tabs contain empty `TabContent`s, and
  `onContentWillChange` returns `false`, so tab switching does not work at all.
- `FoodInfo` has no id field, which is why the dedup fix above has to key on
  `title`.

## Pitfalls

- **`HW-16-0017`** (B/medium, confirmed): `CardInfoCom`'s `onClick` unshifts
  the product into `historyArr` with no duplicate check and no length limit,
  and the key is persisted via `PersistentStorage.persistProp`, so every tap
  re-serialises a growing array to disk on the UI thread - against the
  official guidance that persisted values stay under 2 KB and that substantial
  data belongs in the database APIs. Fix: filter an existing entry out before
  the `unshift`, cap the array (20 entries is plenty for a footprints page),
  or move the list to `@ohos.data.preferences` / `relationalStore`.
- **`persistProp` at module scope in a lazily routed file.** History recorded
  before the user first opens 我的足迹 is never written to disk. Register the
  key in `EntryAbility.onCreate` instead.
- **`persistProp('historyArr', undefined)`.** The `undefined` default is what
  forces the two-branch write and the `!== undefined` guard on a field typed
  `FoodInfo[]`. Seed it with `[]`.
- **A mutation without a re-publish is invisible.** `AppStorage.get` hands back
  the live reference, so `unshift` alone changes the data but notifies nobody;
  the following `setOrCreate` is what drives both the UI update and the disk
  write.
- **`onContentWillChange(() => false)`** on the `Tabs` in `Pages.ets` blocks
  every tab change; only the forced `currentIndex = 2` makes the feature
  reachable.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-persiststorage.md` - `persistProp`, the 2 KB recommendation, the UI-thread write
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-persiststorage
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `AppStorage.get`/`setOrCreate`, `@StorageLink` vs `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - the preferences API the guide routes larger data to
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, the route map
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `NavDestination` and its builder entry function
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-references/02_application-framework/ts-container-waterflow.md` - `WaterFlow`/`FlowItem` used for the product feed
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-waterflow
- `huawei_industry_tree/16_shopping/docs/15_offering_browsing_history.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/offering_browsing_history-0000002361003509
- `SHOP-13` - the same cross-component state problem solved with `@Provide`/`@Consume` when an ancestor exists
