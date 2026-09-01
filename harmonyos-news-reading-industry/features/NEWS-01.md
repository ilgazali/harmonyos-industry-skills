---
id: NEWS-01
title: News app skeleton - a product HAP over feature HARs, with a lazy-loaded feed and TTS read-aloud
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
sample: huawei_industry_tree/11_news_reading/downloads/NewsSolutionDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreSpeechKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: [base, cryptoFramework, hilog, http, mediaquery, preferences, relationalStore, textToSpeech, util, webview, window, LazyForEach, IDataSource, DataChangeListener, PullToRefresh, EdgeEffect, Navigation, NavPathStack, "textToSpeech.createEngine", SpeakListener, "relationalStore.getRdbStore", RdbPredicates, "mediaQuery.matchMediaSync", "Web.runJavaScript", onPageEnd, Marquee, CustomDialogController]
permissions: [ohos.permission.INTERNET]
min_api: 24
modules: [common (har), phone (entry), service (har), live (har), news (har), video (har), personal (har)]
findings: [HW-11-0001, HW-11-0002, HW-11-0003, HW-11-0029, HW-11-0030, HW-11-0032]
status: verified-with-fixes
---

## When to use

Load this card when you are **laying out a news or feed app from scratch** and
need the module split, the feed mechanics and the reading affordances to be
decided before any screen is written. This is the industry's reference
skeleton: one entry HAP for the phone product, five feature HARs, one common
HAR, four bottom tabs, an infinite feed and an article page that can read
itself aloud.

Three patterns in it are load-bearing and worth copying wholesale: **the
layering** (product HAP → feature HARs → common HAR, dependencies strictly
downward, every HAR exporting through a single `Index.ets`); **the feed**
(`PullToRefresh` wrapping a `List` whose rows come from `LazyForEach` over a
hand-written `IDataSource`, with `edgeEffect` disabled so the wrapper owns the
overscroll); and **the read-aloud** (Core Speech Kit's `textToSpeech` engine
driven from a floating player dialog, fed by text scraped out of the article
`Web` view with `runJavaScript`).

The rest of the sample - login, comments, video, live - is scaffolding around
those three. Take the skeleton; do not take the account layer, which stores
passwords in clear (`HW-11-0001`), or the player as written, which crashes on a
fast tap (`HW-11-0002`).

It generalises to any long-feed reading app - a magazine, a community timeline,
a job board. The choice worth internalising is that the feed's data source is an
*object* with `pushData` / `clear` / `notifyDataAdd`, not an array, which is
what makes refresh and load-more cheap enough to run on every gesture.

## Feature checklist

- A splash page hands over to a four-tab main page: 新闻, 视频, 直播, 我的.
- The news tab shows a red header (title, search box, scan icon), a category
  strip (头条 / 体育 / 时政 / …), a carousel and a vertical headline marquee.
- Pull down to refresh: the list is cleared and repopulated from the *other*
  mock file, so the content visibly changes. Pull up to append eight items.
- A back-to-top button appears once the first visible row is index 3 or beyond,
  and scrolls home over 500 ms.
- Tapping a row pushes the article page, which renders the body in a `Web` view
  over a local `news.html`, with like / favourite / comment / share each
  confirmed by a small bottom dialog.
- A play button opens a slim player docked above the bottom edge and reads the
  article aloud, showing a marquee of the text and a pause control.
- 我的 offers registration and login (11-digit phone plus any password),
  persisting accounts in a local RDB, plus a comment history and sign-out.
- The layout responds to sm / md / lg breakpoints: at lg the bottom tab bar
  becomes a vertical side bar.

## Architecture

Seven modules in one project: one entry HAP, six HARs.

```
common/                                   HAR - shared by every feature
├── Index.ets                             the export surface (7 named exports)
└── src/main/ets
    ├── constants/{BreakpointConstants,CommonConstants,StyleConstants}.ets
    ├── preferences/Preferences.ets       PreferenceModel wrapper
    ├── utils/BreakpointSystem.ets        three mediaQuery listeners -> AppStorage
    ├── utils/{BreakpointType,CommonDataSource,HttpUtil,Logger}.ets
    └── viewmodel/{NewsData,NewsDataSource,ViewData,VideoModel,LocalDataModel}.ets

features/                                 HARs - one per bottom tab, plus service
├── news/      AllClass, Comment, News, NewsContent (639), PullToRefreshNews (498)
├── video/     VideoPage, VideoPlayer          live/  Live
├── personal/  Personal (419), MyComment, QuitLoginDialog
└── service/   Service (239), MainPage          <- absent from the documented tree

product/phone/src/main/ets                the entry HAP
├── constants/{CommonConstants,PageConstants}.ets   incl. the RDB table DDL
├── database/{Rdb.ets,tables/AccountTable.ets}
├── entryability/EntryAbility.ets                   loads pages/SplashPage
├── pages/{SplashPage,MainPage,LoginPage,VerifyPage}.ets
└── viewmodel/{AccountInfo,ConstantsInterface,MainPageData}.ets
```

**The documented 代码结构解读 tree omits `features/service` entirely**
(`HW-11-0003`). The module is in the zip, exports `Service` through its own
`Index.ets`, and contains a 239-line component plus a leftover "Hello World"
`MainPage` - it is not referenced from `product/phone`, so the tree was
evidently not regenerated after the module was added.

**The design decision worth copying** is the direction of every dependency.
Feature HARs depend on `common` (`"@ohos/common": 'file:../../common'`) and on
nothing else - `news` does not know `personal` exists. The product HAP depends
on all of them and is the only module that knows the app has four tabs, the
only module holding the `NavPathStack`, and the only module with a
`module.json5` that declares an ability. Route names are resolved in exactly
one place:

```typescript
// product/phone/src/main/ets/pages/MainPage.ets
@Provide('pageStack') pageStack: NavPathStack = new NavPathStack();

@Builder
pageMap(name: string) {
  if (name === 'NewsContent') {
    NewsContent();
  } else if (name === 'AllClass') {
    AllClass();
  } // ... Comment, LoginPage, VerifyPage, VideoPlayer, MyComment, PullToRefreshNews
}
```

A feature component pushes by name (`this.pageStack.pushPathByName('NewsContent', null)`)
having only `@Consume`d the stack; it never imports the destination. That is
what keeps the HARs independently compilable, and it is the thing to preserve
when adding a feature module. `common` earns its place by holding only what
every feature needs: the breakpoint system (which publishes `currentBreakpoint`
into `AppStorage`, read back with `@StorageLink` in a dozen components) and the
`IDataSource` implementations.

## Implementation steps

1. **Create the product HAP first and give it the ability, the routes and the
   navigation stack.** Everything else is a HAR with no ability of its own.
2. **Put breakpoints, data sources, constants and logging in `common`**, export
   them from one `Index.ets`, and register the breakpoint listeners in
   `aboutToAppear` / unregister in `aboutToDisappear` - `matchMediaSync` handles
   leak if left subscribed.
3. **Implement `IDataSource` once**, with `pushData`, `clear` and the five
   `notifyXxx` fan-outs, and reuse it for every lazy list.
4. **Wrap the `List` in `PullToRefresh`**, passing the data source by reference
   (`data: $newsData`), the same `Scroller` the `List` uses, and a `customList`
   builder.
5. **Set `edgeEffect(EdgeEffect.None)` on the wrapped `List`.** With the default
   spring effect the list consumes the overscroll and the refresh gesture never
   reaches the wrapper.
6. **Return a `Promise` from `onRefresh` and `onLoadMore`** and resolve it with
   the toast text; the wrapper keeps the spinner up until you do.
7. **Give the `LazyForEach` a key that changes when the row does** - the sample
   uses `JSON.stringify(item) + index` - and track the first visible index with
   `onScrollIndex` to drive the back-to-top button.
8. **For read-aloud, extract the text from the `Web` view in `onPageEnd`** with
   `runJavaScript` rather than duplicating the body in ArkTS state.
9. **Create the TTS engine before enabling the pause control**, and stop plus
   shut it down when the page goes away (`HW-11-0002`).
10. **Do not persist the password.** Keep `account` and `phone` in the RDB and
    nothing else, or store a salted hash (`HW-11-0001`).

## Verified snippets

All snippets are from `NewsSolutionDemo.zip`. Corrected forms are marked.

**The feed — `features/news/src/main/ets/components/PullToRefreshNews.ets`** (as shipped)

```typescript
PullToRefresh({
  data: $newsData,
  scroller: this.scroller,
  customList: () => {
    this.getListView();
  },
  onRefresh: () => {
    return new Promise<string>((resolve) => {
      setTimeout(() => {
        this.newsData.clear();
        let newsModelMockData: ViewData[] = [];
        if (this.mockFlag) {
          newsModelMockData = getNews(this.getUIContext().getHostContext() as Context, MOCK_DATA_FILE_TWO_DIR);
        } else {
          newsModelMockData = getNews(this.getUIContext().getHostContext() as Context, MOCK_DATA_FILE_ONE_DIR);
        }
        this.mockFlag = !this.mockFlag;
        for (let j = CommonConstants.ZERO; j < NEWS_MOCK_DATA_COUNT; j++) {
          this.newsData.pushData(newsModelMockData[j]);
        }
        resolve(NEWS_RESOLVE_SUCCESS);
      }, NEWS_REFRESH_TIME);
    });
  },
  onLoadMore: () => {
    // the same promise, minus the clear(): eight more items from MOCK_DATA_FILE_ONE_DIR
  },
  customLoad: null,
  customRefresh: null
})
  .height(1)
  .flexGrow(1)
  .flexShrink(1)
  .flexBasis(1);
```

**Four options carry the design.** `data: $newsData` passes the data source *by
reference* - the `$` is required, and without it the wrapper cannot tell the
list to reload. `scroller` must be the very same `Scroller` the inner `List` is
constructed with, because the wrapper drives the scroll position during the
rubber-band. `customList` is a builder rather than a component, so the list
stays a child of the page's build tree and keeps its state bindings. And both
handlers must return promises: the wrapper holds the spinner open until the
promise settles, so resolving synchronously makes the refresh look like it
never happened.

The sizing line is the non-obvious part. `.height(1)` with
`flexGrow/Shrink/Basis` is how the wrapper fills the space remaining inside the
`Flex` column below the fixed header - a plain `height('100%')` measures against
the parent including the header, and overflows. `mockFlag` alternating between
two raw files is just the sample's way of proving a refresh happened; replace it
with a request and keep the shape.

**The list inside it — same file** (as shipped)

```typescript
@Builder
private getListView() {
  Stack({ alignContent: Alignment.BottomEnd }) {
    List({ space: CommonConstants.LIST_SPACE, scroller: this.scroller }) {
      this.CustomSwiper();                         // carousel and headline marquee
      this.FastNews();                             // scroll away with the content
      LazyForEach(this.newsData, (item: ViewData) => {
        ListItem() {
          newsItem({ newsTitle: item.newsTitle, newsContent: item.newsContent,
            newsTime: item.newsTime, newsImage: item.newsImage })
            .onClick(() => {
              this.pageStack.pushPathByName('NewsContent', null);
            });
        };
      }, (item: ViewData, index?: number) => JSON.stringify(item) + index);
    }
    .onScrollIndex((first: number) => {
      this.firstIndex = first;
    })
    .edgeEffect(EdgeEffect.None);

    Row() { Image($r('app.media.ic_public_backtotop')); }
      .onClick(() => {
        if (this.firstIndex >= this.SWITCH_BUTTON) {
          this.scroller.scrollTo({
            xOffset: CommonConstants.ZERO, yOffset: CommonConstants.ZERO,
            animation: { duration: this.ANIMATION_DURATION, curve: Curve.LinearOutSlowIn }
          });
        }
      })
      .visibility(this.firstIndex >= this.SWITCH_BUTTON ? Visibility.Visible : Visibility.None);
  }
}
```

**`edgeEffect(EdgeEffect.None)` is mandatory, not cosmetic** - the document says
so, and it is the line most often dropped when people adapt this: with the
default spring the list eats the overscroll and the wrapper never sees the
gesture. The `Stack` with `alignContent: Alignment.BottomEnd` is the whole
back-to-top mechanism - the button is a sibling of the list, not an overlay
attribute, and its visibility is a pure function of `firstIndex`, which
`onScrollIndex` maintains. Compare `NEWS-08`, which builds the same affordance
on its own. Each row's margin (elided above) is a
`BreakpointType(sm, md, lg).getValue(this.currentBreakpoint)` lookup.

**The data source — `common/src/main/ets/viewmodel/NewsDataSource.ets`** (as shipped)

```typescript
export class NewsDataSource implements IDataSource {
  private listeners: DataChangeListener[] = [];
  public dataArray: Array<ViewData> = [];

  public totalCount(): number {
    return this.dataArray.length;
  }

  public getData(index: number): ViewData {
    return this.dataArray[index];
  }

  public pushData(data: ViewData): void {
    this.dataArray.push(data);
    this.notifyDataAdd(this.dataArray.length - 1);
  }

  public clear(): void {
    this.dataArray = [];
  }

  registerDataChangeListener(listener: DataChangeListener): void {
    if (this.listeners.indexOf(listener) < 0) {
      this.listeners.push(listener);
    }
  }
  notifyDataAdd(index: number): void {
    this.listeners.forEach(listener => {
      listener.onDataAdd(index);
    });
  }
  // unregisterDataChangeListener plus notifyDataReload / Change / Delete / Move follow the same shape
}
```

**`pushData` notifying with the index is what makes load-more O(1).**
`onDataAdd(index)` tells `LazyForEach` a single row appeared; it builds that one
row and leaves the other several hundred alone. Replacing the array and calling
`notifyDataReload()` would rebuild everything, and is the usual reason a "lazy"
list still stutters at 500 items.

One asymmetry to fix when adopting this: `clear()` empties `dataArray` without
notifying anything. It works in `onRefresh` only because eight `pushData` calls
follow immediately - a bare `clear()` leaves rendered rows whose backing data is
gone. Pair it with `notifyDataReload()`.

**Read-aloud — `features/news/src/main/ets/components/NewsContent.ets`** (corrected, see `HW-11-0002`)

```typescript
let ttsEngine: textToSpeech.TextToSpeechEngine | undefined = undefined;   // FIX: was unassigned, non-optional

@CustomDialog
struct Speech {
  @State flag: boolean = false;              // FIX: was `true` before the engine exists
  controller: CustomDialogController;
  @Prop webResult: string;

  private speak() {
    let initParamsInfo: textToSpeech.CreateEngineParams = {
      language: 'zh-CN', person: 0, online: 1,
      extraParams: { 'style': 'interaction-broadcast', 'locate': 'CN', 'name': 'EngineName' }
    };
    // onStart / onComplete / onStop / onData only log; onError carries the codes that matter -
    // 1003400007 = speak() before the engine exists, 1003400006 = a second speak() while busy
    let speakListener: textToSpeech.SpeakListener = { /* five logging callbacks */ };

    textToSpeech.createEngine(initParamsInfo,
      (err: BusinessError, textToSpeechEngine: textToSpeech.TextToSpeechEngine) => {
        if (!err) {
          ttsEngine = textToSpeechEngine;
          ttsEngine.setListener(speakListener);
          let speakParams: textToSpeech.SpeakParams = {
            requestId: '123456789-c',        // one requestId per engine instance, never reused
            extraParams: { 'queueMode': 0, 'speed': 1, 'volume': 2, 'pitch': 1,
              'languageContext': 'zh-CN', 'audioType': 'pcm', 'soundChannel': 3, 'playType': 1 }
          };
          ttsEngine.speak(this.webResult, speakParams);
          this.flag = true;                  // FIX: only now is a pause control meaningful
        } else {
          // 1003400005: engine missing, resources missing, or creation timed out
          Logger.error(`Failed to create engine. Code: ${err.code}, message: ${err.message}.`);
        }
      });
  }

  aboutToDisappear() {                        // FIX: absent in the sample
    if (ttsEngine) {
      ttsEngine.stop();
      ttsEngine.shutdown();
      ttsEngine = undefined;
    }
  }

  build() {
    Row() {
      if (this.flag) {
        Image($r('app.media.ic_pause')).width(20)
          .onClick(() => {
            if (ttsEngine) {                 // FIX: sample calls stop() on a possibly undefined engine
              ttsEngine.stop();
              ttsEngine.shutdown();
              ttsEngine = undefined;
            }
            this.flag = false;
          });
      } else {
        Image($r('app.media.ic_play_fill')).width(20)
          .onClick(() => {
            this.speak();                    // FIX: sample sets flag = true before speak()
          });
      }
      Marquee({ start: true, step: this.step, loop: this.loop,
        fromStart: this.fromStart, src: this.webResult + this.src }).width('85%');
    }
  }
}
```

**`createEngine` is asynchronous and the sample's UI assumes it is not.**
`ttsEngine` is a module-level variable declared without an initialiser; the
dialog opens with `flag: true`, which renders the pause button immediately; the
pause handler calls `ttsEngine.stop()` directly. Tap pause in the second before
the callback lands - or at any time after a creation failure - and you
dereference `undefined`. Moving the `flag = true` inside the success branch and
guarding the handler costs two lines and removes the whole class.

The lifecycle half is worse in practice: the only `shutdown()` in the sample is
behind the pause button, so leaving the article while it is speaking leaves an
engine running with no handle to it. `aboutToDisappear` on the dialog is the
right hook, because the dialog owns the playback session. And `requestId` is
documented as usable once per engine instance - the constant string here is safe
only because a fresh engine is created on every play.

The text it reads is not duplicated in ArkTS state: the article `Web` view is
declared with `.javaScriptAccess(true)` and, in `.onPageEnd`, calls
`this.controller.runJavaScript('getSpan()', ...)`, parses the returned array and
accumulates it into `this.webResult`. `onPageEnd` is the earliest point at which
the DOM is complete, and `javaScriptAccess(true)` is required for
`runJavaScript` to run at all. Note the `+=`: revisiting the page without
resetting `webResult` reads the article twice.

**The account table — `product/phone/src/main/ets/...`** (corrected, see `HW-11-0001`)

```typescript
// constants/CommonConstants.ets
static readonly ACCOUNT_TABLE: AccountTable = {
  tableName: 'accountTable',
  // FIX: the shipped DDL ends with `password TEXT NOT NULL`
  sqlCreate: 'CREATE TABLE IF NOT EXISTS accountTable (id INTEGER PRIMARY KEY AUTOINCREMENT, ' +
    'account TEXT NOT NULL, phone TEXT NOT NULL)',
  columns: ['id', 'account', 'phone']
};

// database/tables/AccountTable.ets
function generateBucket(account: AccountInfo): relationalStore.ValuesBucket {
  let obj: relationalStore.ValuesBucket = {};
  obj.account = account.account;
  obj.phone = account.phone;
  // FIX: `obj.password = account.password;` removed - see HW-11-0001
  return obj;
}
```

**The demo is honest that login is UI-only** - the document says an 11-digit
phone and any password will do - but it still writes the string the user typed
into an unencrypted SQLite file, and reads it back with
`resultSet.getString(resultSet.getColumnIndex('password'))`. That is the line
developers copy into production. Nothing in the sample authenticates against the
stored value, so dropping the column costs no functionality; real credentials
belong in Asset Store Kit, not in the app's RDB.

The `Rdb` wrapper around it is worth keeping as-is: one class per table, a
lazily created `rdbStore`, `executeSql(this.sqlCreateTable)` on first open, and
callbacks rather than promises, so it can be driven from `aboutToAppear`
without an async build.

## Permissions & config

```json5
// product/phone/src/main/module.json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

`INTERNET` is `normal` (system_grant), so no runtime request and no `reason` is
required. It is declared for `HttpUtil` and the `Web` component; the feed itself
never uses it, since the mock data comes from two raw files read with
`resourceManager.getRawFileContentSync`. The HAP also registers the standard
`EntryBackupAbility` (`type: "backup"`), unmodified.

The root `oh-package.json5` pulls three third-party packages: `dayjs@^1.11.11`,
`@ohos/pulltorefresh@^2.1.3-rc.0` and `@ohos/lottie@^2.0.10`.
`@ohos/pulltorefresh` is an OpenHarmony community package, not a system
component - budget for it in a security review, and note that the whole feed
gesture depends on it.

## Constraints

- API Version 24 Release or later; DevEco Studio 6.1.1 Release or later. This
  is the newest baseline in the industry - the scenario samples (`NEWS-03`
  onward) target API 20.
- The document is explicit that this is framework code, and that **login is UI
  only**: "手机号输入满11位，任意密码可登录" - eleven digits and any password
  will do. Authentication is the reader's job.
- There is no network layer in use. `HttpUtil` exists in `common` but the feed,
  the video list and the live list all read static mock data, and all seven
  items in the 分类 strip show the same feed.
- `BreakpointType.getValue()` returns `'' as T` with the real lookup commented
  out, so every breakpoint-keyed value resolves to an empty string. The feed
  margins are effectively zero; the mechanism is wired but inert.
- `features/service` is unreferenced by the product HAP and ships a leftover
  "Hello World" `MainPage` - dead weight, and the reason the documented tree is
  out of date (`HW-11-0003`).
- `PullToRefreshNews.aboutToAppear` registers `keyboardHeightChange` on the
  last window and never removes it, and its `setWindowLayoutFullScreen(true)`
  promise is swallowed by empty handlers.

## Pitfalls

- **`HW-11-0001`** (D/medium, confirmed): user passwords are persisted in clear
  in the local RDB. `generateBucket` writes `obj.password = account.password`,
  `LoginPage` and `VerifyPage` insert what the user typed, and `AccountTable.query`
  reads it back with `getString('password')`. Fix: drop the password column
  (keep account and phone), or store a salted hash; use Asset Store Kit for
  real credentials.
- **`HW-11-0002`** (B/medium, confirmed): the TTS pause button dereferences an
  engine that may not exist yet, and leaving the page never stops or releases
  it. `ttsEngine` is a module-level unassigned variable, `flag` is `true` from
  the moment the dialog opens, and the only `shutdown()` sits behind pause.
  Fix: set the flag in `createEngine`'s success callback, guard the pause
  handler with `if (ttsEngine)`, and stop plus shut down in `aboutToDisappear`.
- **`HW-11-0003`** (E/low, confirmed): the 代码结构解读 tree lists the `live`,
  `news`, `personal` and `video` feature HARs but not `features/service`, which
  is in the zip with `Service.ets` and `MainPage.ets`. The tree is the map
  readers navigate by; a whole missing module means it was not regenerated.
  Fix: regenerate the tree from the current zip, or remove the module.

## References

- `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `LazyForEach`, `IDataSource`, `DataChangeListener`, and which `notifyXxx` to use
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `edgeEffect`, `onScrollIndex`, `Scroller.scrollTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `pushPathByName`, the `navDestination` builder
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/07_ai/hms-ai-texttospeech.md` - `createEngine`, `SpeakParams`, `SpeakListener`, the 10034000xx error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/hms-ai-texttospeech
- `documentation/harmonyos-guides/08_ai/texttospeech-guide.md` - the engine lifecycle this sample gets wrong
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/texttospeech-guide
- `documentation/harmonyos-references/02_application-framework/js-apis-data-relationalstore.md` - `getRdbStore`, `RdbPredicates`, `ValuesBucket`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-relationalstore
- `documentation/harmonyos-references/03_system/asset-store-module.md` - where credentials belong
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/asset-store-module
- `NEWS-02` - the index of the 24 scenario samples that plug into this skeleton
- `NEWS-04` / `NEWS-09` - the channel editor and the read-aloud feature, each done properly as its own sample
