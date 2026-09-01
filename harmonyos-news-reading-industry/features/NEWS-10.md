---
id: NEWS-10
title: Ads in a lazy feed - NodeContainer placeholders filled by a BuilderNode per ad slot
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/10_ad_loading.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ad_loading-0000002257601520
sample: huawei_industry_tree/11_news_reading/downloads/ADLoading.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [UIContext, hilog, node, window, NodeContainer, NodeController, BuilderNode, FrameNode, wrapBuilder, LazyForEach, IDataSource, Swiper, IndicatorComponent, bindMenu, Video]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0011, HW-11-0012, HW-11-0031]
status: verified-with-fixes
---

## When to use

**Load this card when content the app does not own has to be interleaved into
a list the app does own** - ads in a news feed, sponsored cards in a shop, a
promoted post between social posts. The distinguishing property is not that
the content is an advertisement; it is that the item's component tree is
decided at runtime, by data that arrives with the slot, and may be a different
component type per slot.

The mechanism is `NodeContainer` + `NodeController`. A `NodeContainer` is an
empty box in the list; the `NodeController` bound to it owns a detached node
tree built with `BuilderNode` and hands its root to the container through
`makeNode`. That indirection buys two things a plain `if/else` inside the
`LazyForEach` cannot: the ad's tree is built off the list's render pass, and it
can be built, cached, replaced or disposed on the ad's lifecycle rather than
the list item's.

Use it when the injected content is genuinely dynamic. If the variants are a
fixed small set known at compile time - "big image card or small image card" -
an `if/else` over two `@Builder`s is simpler and faster; this sample's own ad
component ends up being exactly that, which is worth noticing before adopting
the heavier machinery.

**Read `HW-11-0011` before adopting it.** Node-based UI moves memory
management from the framework to you, and this sample does not do it.

## Feature checklist

- A news home page: top bar, an auto-playing banner `Swiper`, a scrolling feed,
  a bottom tab bar.
- The feed holds 30 items; every third one (index % 3 === 2) is an ad slot
  rather than a news card.
- Each ad slot renders through a `NodeContainer`, not a normal component.
- The ad's type is read from its id: even-ending ids render an image ad,
  odd-ending ids render a video ad.
- Every ad carries a 广告 (ad) badge in its top-right corner.
- Long-pressing the badge opens a menu with 关闭广告 (close ad); choosing it
  collapses the ad to a 1vp blank without disturbing the rest of the list.
- The feed is a `LazyForEach` over a custom `IDataSource` with a cache of 5.

## Architecture

One `entry` module. The ad machinery is one file; the rest is an ordinary feed.

```
entry/src/main/ets
├── common/Constants.ets              layout numbers + the AD_POSITION / ONE_CYCLE slot rule
├── components/AdBuilder.ets          the whole node pipeline: controller, builders, NODE_MAP
├── components/NewsComponent.ets      the two alternating news-card layouts
├── entryability/EntryAbility.ets     full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model/AdData.ets                  AdData + BasicDataSource/CardDataSource (IDataSource)
├── model/IconModel.ets               bottom-tab icon + label pair
├── model/NewsInfoModel.ets           NewsInfo + two static articles
├── pages/HomePage.ets                @Entry: top bar, feed, bottom bar
└── views/NewsInfoList.ets            builds the datasource, the Swiper, and the LazyForEach
    views/PageTopBuilder.ets          search row
    views/PageBottomBuilder.ets       bottom tabs
```

The documented tree names the model file `IconModels.ets`; the zip has
`IconModel.ets` (`HW-11-0012`). The doc's step-2 snippet is also stale in a
second way: it calls `queryAdById(adId)` with one argument, while the shipped
function takes `(adId, uiContext)` - a reader copying the doc gets a compile
error.

**The design decision worth copying** is that the datasource carries *slot
kind*, not content. `AdData` is two fields - `isAdCard` and an id - and the
feed's `LazyForEach` branches on `isAdCard()` alone. Nothing about ad layout,
ad media or ad lifetime reaches the list. The list's contract is "this index is
an ad, here is its id"; resolving that id into a component tree is entirely
inside `AdBuilder.ets`. That separation is what lets the ad pipeline be
replaced (real ad SDK, remote config, A/B variants) without touching the feed.

**And one worth avoiding**: `NODE_MAP` is a module-level `Map` that is written
on every controller creation and read by nobody except the line that just
wrote it. It is presented as a cache and behaves as a leak - see
`HW-11-0011` and the corrected snippet below.

## Implementation steps

1. **Encode the slot rule as constants,** not literals:
   `i % Constants.ONE_CYCLE === Constants.AD_POSITION` reads as "one ad every
   three items, in third position" and is tunable in one place.
2. **Push both kinds into one `IDataSource`** so the ad slots participate in
   lazy loading and `cachedCount` like any other row.
3. **Give the `LazyForEach` a real key.** Keys must be stable and cheap;
   `JSON.stringify` on the item is neither (see Pitfalls).
4. **Bind the slot to a controller** with
   `NodeContainer(getAdNodeController(this.getUIContext(), item.getAdId()))`.
5. **Build the ad tree inside the controller**: a `FrameNode` as root, a
   `BuilderNode` built from a `wrapBuilder`'d global `@Builder`, then append
   the builder node's render node to the root's.
6. **Return the cached controller when the id is already known,** and dispose
   the `BuilderNode` and drop the map entry when the slot detaches
   (`HW-11-0011`).
7. **Pass a `UIContext`, never assume one.** `FrameNode` and `BuilderNode` both
   require the instance's `UIContext`; `NodeContainer` explicitly does not
   support cross-instance reuse.
8. **Make "close" collapse, not delete.** Flipping a local `@State isRemoved`
   inside the ad component leaves the list geometry to the framework and needs
   no datasource mutation.

## Verified snippets

All snippets are from `ADLoading.zip`. Corrected forms are marked.

**Marking the slots and placing the containers — `entry/src/main/ets/views/NewsInfoList.ets`** (as shipped)

```typescript
@Component
export struct NewsInfoList {
  private data: CardDataSource = new CardDataSource();
  private adIdList: ArrayList<ResourceStr> = new ArrayList();

  aboutToAppear() {
    for (let i = Constants.ZERO; i < Constants.THIRTY; i++) {
      if (i % Constants.ONE_CYCLE === Constants.AD_POSITION) {   // 3-item cycle, slot 2
        let adId = $r('app.string.example_ad', i);               // resource with an index param
        let cd = new AdData(true, adId);
        this.data.pushData(cd);
        this.adIdList.add(adId);
      } else {
        this.data.pushData(new AdData(false, 'ad' + i));
      }
    }
  }

  build() {
    Column({ space: 8 }) {
      this.swiperBuilder();

      List({ space: Constants.LIST_SPACE_16 }) {
        LazyForEach(this.data, (item: AdData) => {
          ListItem() {
            if (item.isAdCard()) {
              // ad slot: a placeholder whose contents come from a NodeController
              NodeContainer(getAdNodeController(this.getUIContext(), item.getAdId()))
                .width(Constants.FULL_PERCENT);
            } else {
              Column() {
                NewsComponent();
              }
              .width(Constants.FULL_PERCENT);
            }
          };
        }, (item: AdData) => JSON.stringify(item.getAdId()));
      }
      .scrollBar(BarState.Off)
      .cachedCount(Constants.CACHE_COUNT);
    }
  }
}
```

**The ad id is a `Resource`, not a string, and that is deliberate.**
`$r('app.string.example_ad', i)` produces a parameterised resource whose
formatted value differs per index, so the id is both localisable and unique
per slot. The cost is that it can only be resolved with a `resourceManager`,
which is why `queryAdById` needs a `UIContext` - and why the doc's one-argument
version does not compile.

Two things here are worth not copying. `adIdList` is populated and never read.
And the key generator is `JSON.stringify(item.getAdId())`, which the ArkUI
lint rule `@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator`
exists specifically to flag: stringifying an object on every key computation
costs frames during scroll, and two ad slots that resolve to equal resource
objects would collide. `item.getAdId()` is already unique per slot when it is a
plain string - use an explicit `'ad-' + index`.

**Building the detached tree — `entry/src/main/ets/components/AdBuilder.ets`** (as shipped)

```typescript
import { UIContext } from '@ohos.arkui.UIContext';
import { NodeController, BuilderNode, FrameNode } from '@ohos.arkui.node';

class AdNodeController extends NodeController {
  private rootNode: FrameNode | null = null;
  private adNode: BuilderNode<[AdParams]> | null = null;
  private uiContext: UIContext | null = null;

  makeNode(): FrameNode | null {
    if (this.rootNode !== null) {
      return this.rootNode;
    }
    return null;
  }

  // custom init: build the BuilderNode, then graft it under the root FrameNode
  initAd(uiContext: UIContext, adId: ResourceStr, adType: ResourceStr) {
    this.uiContext = uiContext;
    this.rootNode = new FrameNode(this.uiContext);
    this.adNode =
      new BuilderNode(this.uiContext,
        { selfIdealSize: { width: Constants.NODE_WIDTH, height: Constants.NODE_HEIGHT } });
    this.adNode.build(wrapBuilder(getAdComponent), { adId: adId, isVideo: adType === 'video' });
    this.rootNode.getRenderNode()?.appendChild(this.adNode.getFrameNode()?.getRenderNode());
  }
}

@Builder
function getAdComponent(param: AdParams) {
  if (!param.isVideo) {
    photoAndTextAdBuilder({ params: param });
  } else {
    VideoAdComponent({ params: param });
  }
}
```

**Three lines carry the pipeline.** `new BuilderNode(uiContext, { selfIdealSize })`
creates a node with a measurement constraint of its own - the ad is laid out at
300x160 regardless of what the list would have given it, which is exactly the
guarantee an ad slot needs. `build(wrapBuilder(getAdComponent), params)` is what
turns a global `@Builder` function into a live component tree: `wrapBuilder`
only accepts a *global* builder, which is why `getAdComponent` sits outside any
struct. And `appendChild` on the **render** nodes - not the frame nodes - is
what puts the built tree under the root that `makeNode` returns.

`makeNode()` here is declared with no parameters, though the abstract signature
is `makeNode(uiContext: UIContext)`. It works because `initAd` has already
stashed a context, but it discards the framework's own instance hint; the
reference warns that this parameter can be `undefined` on cross-instance reuse
and should be checked. Take the parameter and use it.

**Getting a controller for a slot — same file** (corrected, see `HW-11-0011`)

```typescript
export const NODE_MAP: Map<ResourceStr, AdNodeController | undefined> = new Map();

export const getAdNodeController = (uiContext: UIContext, adId: ResourceStr): AdNodeController | undefined => {
  const cached = NODE_MAP.get(adId);         // FIX: shipped code always constructs a new controller
  if (cached !== undefined) {
    return cached;
  }
  const baseNode = new AdNodeController();
  NODE_MAP.set(adId, baseNode);
  baseNode.initAd(uiContext, adId, queryAdById(adId, uiContext));
  return baseNode;
};

// FIX: absent in the sample - nothing disposes a BuilderNode or drops a map entry
export const disposeAdNodeController = (adId: ResourceStr): void => {
  NODE_MAP.get(adId)?.dispose();
  NODE_MAP.delete(adId);
};

class AdNodeController extends NodeController {
  private adId: ResourceStr = '';            // FIX: added - initAd must stash the id it was built for

  dispose(): void {                          // FIX: added
    this.adNode?.dispose();
    this.adNode = null;
    this.rootNode = null;
  }

  onDetach(): void {                         // FIX: added - fires when the NodeContainer unbinds
    disposeAdNodeController(this.adId);
  }
}
```

**The shipped version writes the cache and never reads it.** `NODE_MAP.set` is
followed immediately by `NODE_MAP.get(adId)` on the value that was just
inserted, so the map never short-circuits anything - it only accumulates. Since
`getAdNodeController(...)` is called from inside the `LazyForEach` item
builder, it runs again on every rebuild of that slot, each time constructing a
fresh `FrameNode` + `BuilderNode` pair and overwriting the previous entry. The
overwritten controller's node tree stays reachable through the frontend object
until the process ends.

`BuilderNode.dispose()` is not optional bookkeeping. The reference is explicit:
"After calling `dispose()`, the BuilderNode object cancels its reference to the
backend entity node. If the frontend object BuilderNode cannot be released,
memory leaks may occur." `NodeController.onDetach` (API 18+) is the hook that
fires when the `NodeContainer` unbinds, which is the natural place to call it.

**Choosing the ad's type — same file** (as shipped)

```typescript
export function queryAdById(adId: ResourceStr, context: UIContext): string {
  let str = '';
  if (typeof adId === 'string') {
    str = adId as string;
  } else {
    let index = adId.params === undefined ? 0 : adId.params[0] as number;
    let myContext: Context = context.getHostContext() as Context;
    str = myContext.resourceManager.getStringSync(adId.id, index);
  }
  if (str.endsWith('0') || str.endsWith('2') || str.endsWith('4') || str.endsWith('6') || str.endsWith('8')) {
    return 'pic';
  } else {
    return 'video';
  }
}
```

This is the sample's stand-in for an ad config service, and it is the seam
worth keeping when the stand-in is replaced. Everything upstream of it knows
only an id; everything downstream knows only `'pic' | 'video'`. Swapping the
last-digit parity for a real lookup - a network response, a local config file,
an ad SDK callback - changes this function and nothing else.

The `ResourceStr` handling above the parity check is the part to keep verbatim:
a `Resource` id must be resolved through `resourceManager.getStringSync(id,
param)` with the parameter recovered from `adId.params[0]`, because
`$r('app.string.example_ad', i)` is a *template* until it is formatted.

Note that `AdParams.text` is declared and never used, and the built components
ignore `params` entirely - `photoAndTextAdBuilder` renders a fixed
`$r('app.media.pic')` and `VideoAdComponent` a fixed `$rawfile('video.mp4')`.
Real ad creatives would flow through `AdParams`.

## Permissions & config

**None.** The sample declares no `requestPermissions`. The ad media is bundled
(`app.media.pic`, `rawfile/video.mp4`), so not even `INTERNET` is needed - a
real ad pipeline would need it.

`deviceTypes` is `phone`, `tablet`, `2in1`. `EntryAbility` publishes
`topRectHeight` and `bottomRectHeight` into `AppStorage`; `HomePage` consumes
the top one with `@StorageProp` + `px2vp`, which is the right form (compare
`TOUR-03`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `NodeContainer` does not support cross-instance node reuse. A node tree built
  against one `UIContext` cannot be shown in another window or another ability
  instance; `makeNode`'s `uiContext` argument may arrive `undefined` if you
  try.
- `NodeController.onAttach`/`onDetach` require API 18 or later. On earlier
  levels the cleanup has to be driven from the hosting component's
  `aboutToDisappear`.
- `BuilderNode.dispose` requires API 12 or later.
- The banner `IndicatorComponent` is configured `.count(6)` over a five-image
  `Swiper`, so the dot strip shows one dot that never activates.
- `NewsComponent` renders the same two static articles in every non-ad slot, so
  the 30-item feed is 20 copies of the same pair.
- The ad component's `isRemoved` state lives inside the `BuilderNode` tree, not
  in the datasource, so a closed ad reappears if its node is ever rebuilt.

## Pitfalls

- **`HW-11-0011` — ad node controllers are cached in a module-level map that is
  never cleaned up** (B/low, probable). `getAdNodeController` constructs a new
  `AdNodeController` on every call and overwrites the map entry; no
  `NODE_MAP.delete`, no `BuilderNode.dispose()`, no lifecycle hook exists in
  the sample, so each rebuild of an ad slot strands a whole
  `FrameNode`/`BuilderNode` tree for the life of the process. Fix: return the
  cached controller when present, and dispose the node plus delete the entry
  from `onDetach` (or the page's teardown).
- **`HW-11-0012` — the documented project tree lists `IconModels.ets`; the zip
  file is `IconModel.ets`** (E/low, confirmed). Fix: rename the tree entry to
  the singular form. The same doc/zip filename drift is filed as `HW-11-0015`,
  `HW-11-0019` and `HW-11-0020` elsewhere in this industry.
- **Not filed: the doc's step-2 snippet calls `queryAdById(adId)` with one
  argument** while the shipped signature is `(adId: ResourceStr, context:
  UIContext)`. Copying the doc gives a compile error.
- **Not filed: the `LazyForEach` key generator is
  `JSON.stringify(item.getAdId())`.** ArkUI ships a lint rule against exactly
  this (`@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator`); it
  costs frames while scrolling and risks key collision between two slots whose
  resource objects serialise identically.
- **Not filed: `makeNode()` ignores its `uiContext` parameter,** which the
  reference says must be checked for `undefined` in reuse scenarios.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-nodecontainer.md` -
  `NodeContainer`, and the cross-instance-reuse restriction
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-nodecontainer
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-nodecontroller.md` -
  `makeNode`, `onAttach`/`onDetach`, `rebuild`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-nodecontroller
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-buildernode.md` -
  `build`, `getFrameNode`, and the `dispose()` leak note
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-buildernode
- `documentation/harmonyos-guides/03_application-framework/arkts-user-defined-arktsnode-buildernode.md` -
  cancelling the reference to the entity node
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-user-defined-arktsnode-buildernode
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` -
  `IDataSource` and key-generator rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` -
  the stringify-key lint rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
