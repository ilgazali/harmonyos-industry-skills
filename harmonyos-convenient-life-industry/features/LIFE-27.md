---
id: LIFE-27
title: Grouped photo preview with a linked category strip - two Swipers kept in sync through an index-range map
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
sample: huawei_industry_tree/02_convenient_life/downloads/demo_HouseView.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Swiper, SwiperController, "SwiperController.changeIndex", "Swiper.displayCount", "Swiper.indicator", "Swiper.onChange", ForEach, Grid, GridItem, Tabs, TabsController, TabContent, Navigation, NavPathStack, "navPathStack.getParamByName", "navPathStack.pushPathByName", "navPathStack.pop", NavDestination, NavDestinationContext, State, Provide, Consume, StorageLink, "AppStorage.setOrCreate", Line, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.getWindowProperties", "window.on('avoidAreaChange')", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0224, HW-02-0225, HW-02-0226, HW-02-0227, HW-02-0228, HW-02-0229, HW-02-0230, HW-02-0231, HW-02-0232, HW-02-0233, HW-02-0234, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **a flat photo pager with a category strip above it**: images
grouped into sections - kitchen, bedroom, living room - shown in one continuous
swipe, with a tab strip that highlights whichever section the current photo
belongs to and jumps to a section's first photo when tapped.

The whole idea is that the images are **not** re-grouped per tab. There is one
flat sequence, and a map from category index to the photo-index range it
occupies keeps the two views in agreement:

```
categories: [ 厨房(2 photos), 卧室(1), 客厅(1) ]
flat photo indexes:  0  1 | 2 | 3
idMap:  0 -> 1,  2 -> 2,  3 -> 3      // start -> end, per category

tap a category  -> photoId = idMap start for that category -> changeIndex(photoId)
swipe a photo   -> photoId = index -> find the range containing it -> categoryId
```

Build `idMap` once from the category lengths, and both directions become
arithmetic instead of bookkeeping:

```ts
let nums: number[] = [];
this.house.categories.forEach((category: Category) => { nums.push(category.details.length); });
let num: number = 0;
nums.forEach((number: number) => {
  this.idMap.set(num, num + number - 1);
  num += number;
});
```

That map is the reusable part of this card. What the sample gets wrong is the
plumbing around it - see pitfall 1 in particular.

## Feature checklist

- [ ] One `SwiperController` **per** `Swiper` (HW-02-0224).
- [ ] `idMap` rebuilt whenever the data changes.
- [ ] The tab index decorated as state (HW-02-0225).
- [ ] `@Provide` and `@Consume` types matching exactly (HW-02-0226).
- [ ] Navigation parameters checked for existence **before** `.length`
      (HW-02-0229).
- [ ] `ForEach` keys short and unique, not serialised objects
      (HW-02-0231).
- [ ] Avoid areas read inside the `setWindowLayoutFullScreen` promise
      (HW-02-0228) and `off('avoidAreaChange')` on teardown (HW-02-0227).
- [ ] No absolute pixel positions in the layout (HW-02-0232).

## Architecture

| File | Role |
| --- | --- |
| `pages/HomePage.ets` | `@Entry`. Owns the `NavPathStack` and the `UIContext`, both as `@Provide`. Four bottom tabs, of which only the first has content. |
| `components/HomeComponent.ets` | The listing grid, and the push into the detail route. |
| `pages/HouseInfoPage.ets` | The listing detail page. |
| `pages/HouseImagePage.ets` | The feature: two `Swiper`s and the `idMap` that links them. |
| `components/PhoneLineComponent.ets` | The contact bar shared by both detail pages. |
| `models/HouseModel.ets` | `House` / `Category` / `Detail`, plus the static `houseList`. |
| `common/Constants.ets` | One constant: the hilog domain. |

The model nesting is what makes the flat sequence necessary:

```ts
export class House   { categories: Category[]; /* ... */ }
export class Category { name: string; details: Detail[]; }
export class Detail   { image: Resource; name: string; }
```

`House.unitPrice` is derived in the constructor rather than stored -
`this.unitPrice = Math.floor(this.totalPrice / this.size * 10000);` - which is
the right place for it.

State is shared downward with `@Provide` / `@Consume` rather than through
`AppStorage`: `navPathStack` and `uiContext` are provided by `HomePage` and
consumed by all three descendants. The avoid-area heights go the other way,
written into `AppStorage` by the ability and read with `@StorageLink`.

Routing: `main_pages.json` holds `pages/HomePage`; the two detail pages are
`routerMap` entries reached with `pushPathByName`, and the parameter is read
back with `getParamByName`.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Set up the immersive layout, in the right order** (HW-02-0228,
   HW-02-0227):

   ```ts
   this.windowClass = windowStage.getMainWindowSync();
   this.windowClass.setWindowLayoutFullScreen(true).then(() => {
     let navArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
     AppStorage.setOrCreate('bottomRectHeight', navArea.bottomRect.height);
     let sysArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
     AppStorage.setOrCreate('topRectHeight', sysArea.topRect.height);
     this.windowClass.on('avoidAreaChange', (data) => { /* later changes */ });
   }).catch((err: BusinessError) => { hilog.error(DOMAIN, 'testTag', '%{public}s', JSON.stringify(err)); });
   ```

   ```ts
   onWindowStageDestroy(): void {
     this.windowClass?.off('avoidAreaChange');
   }
   ```

   This sample stores **px** and converts at the point of use, which is why
   every page calls `px2vp`.

2. **Provide the stack and the context once, with honest types**
   (HW-02-0226):

   ```ts
   @Provide navPathStack: NavPathStack = new NavPathStack();
   @Provide uiContext: UIContext | undefined = undefined;   // consumers must declare the same union
   ```

   The shipped consumers all declare `@Consume uiContext: UIContext;` - the
   non-optional type - which the state guidance forbids. Either widen the three
   consumers or narrow the provider.

3. **Decorate the tab index** (HW-02-0225). It is bound two-way and read for
   the tab-bar styling:

   ```ts
   @State tabId: number = 0;       // shipped: undecorated
   ...
   Tabs({ barPosition: BarPosition.End, index: $$this.tabId, controller: this.tabsController })
   ```

   `$$` only triggers UI updates for variables carrying a state decorator.

4. **Push the listing into the detail route.**

   ```ts
   this.navPathStack.pushPathByName('HouseInfo', [house]);
   ```

5. **Read the parameter back, checking existence first** (HW-02-0229):

   ```ts
   async aboutToAppear(): Promise<void> {
     try {
       let params = this.navPathStack.getParamByName('HouseInfo') as House[];
       if (params !== undefined && params !== null && params.length !== 0) {   // shipped order is reversed
         this.house = params[0];
         // step 6
       }
     } catch (e) {
       hilog.error(Constants.DOMAIN, 'testTag', 'Failed to get Params. error: %{public}s', JSON.stringify(e));
     }
   }
   ```

6. **Build the index-range map from the category lengths.** This is the core of
   the feature:

   ```ts
   let nums: number[] = [];
   this.house.categories.forEach((category: Category) => {
     nums.push(category.details.length);
   });
   let num: number = 0;
   nums.forEach((number: number) => {
     this.idMap.set(num, num + number - 1);   // start index -> end index, inclusive
     num += number;
   });
   ```

   `@State idMap: Map<number, number> = new Map();` - `Map` is an observable
   state type since API 11, so `set` inside `aboutToAppear` is fine.

7. **Render the category strip in its own Swiper, with its own controller**
   (HW-02-0224):

   ```ts
   Swiper(this.indicatorCtr) {                     // shipped: the same controller as the photos
     ForEach(this.house?.categories, (category: Category, index: number) => {
       Column() {
         Text(category.name + '(' + category.details.length + ')')
           .fontColor(this.categoryId === index ? '#ff0a59f7' : '#ffffffff');
         Line()
           .stroke(this.categoryId === index ? '#ff0a59f7' : '')
           .startPoint([0, 1]).endPoint([48, 1])
           .strokeWidth(2).strokeLineCap(LineCapStyle.Round);
       }
       .onClick(() => {
         this.categoryId = index;
         let arr: Array<[number, number]> = Array.from(this.idMap.entries());
         this.photoId = arr[this.categoryId][0];
         this.photoCtr.changeIndex(this.photoId);   // the photo controller, not a shared one
       });
     }, (category: Category) => category.name)
   }
   .displayCount(5)
   .indicator(false)
   ```

   `Array.from(this.idMap.entries())[categoryId][0]` is the start index of that
   category's range - the first photo to jump to. `displayCount(5)` shows five
   tabs at once; `indicator(false)` hides the dots.

8. **Render the photos as one flat sequence** with nested `ForEach`:

   ```ts
   Swiper(this.photoCtr) {
     ForEach(this.house?.categories, (category: Category) => {
       ForEach(category.details, (detail: Detail) => {
         Image(detail.image).objectFit(ImageFit.Cover).height(300).width(400);
       }, (detail: Detail) => detail.name)
     }, (category: Category) => category.name)
   }
   .indicator(false)
   ```

   The nesting produces one continuous page list - which is exactly what makes
   `idMap` the right shape. Key on short unique fields, not on serialised
   objects (HW-02-0231).

9. **Map a swipe back to its category.**

   ```ts
   .onChange((index: number) => {
     this.photoId = index;
     let id = 0;
     this.idMap.forEach((v: number, k: number) => {
       if (this.photoId >= k && this.photoId <= v) {
         this.categoryId = id;
       }
       id += 1;
     });
   })
   ```

   `k` is the range start, `v` the inclusive end, and `id` counts iterations to
   recover the category index - `Map` preserves insertion order, which is what
   makes the counter valid.

10. **Lay the contact bar out by flow, not by coordinate** (HW-02-0232). The
    shipped component pins itself at `y: 718` and compensates with negative
    margins; put it at the end of the `Column` instead, or in a `Stack` aligned
    to `Alignment.Bottom`.

11. **Apply the safe-area padding with a guard** (HW-02-0230):

    ```ts
    .padding({
      top: this.topRectHeight ? this.uiContext.px2vp(this.topRectHeight) : 0,
      bottom: this.bottomRectHeight ? this.uiContext.px2vp(this.bottomRectHeight) : 0
    })
    ```

    Or default the two `@StorageLink` fields to `0` and drop the union - which
    matches what the ability always writes.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`demo_HouseView.zip#entry/src/main/ets/pages/HouseImagePage.ets:39-57` - reading
the route parameter and building the index-range map:

```ts
  async aboutToAppear(): Promise<void> {
    try {
      let params = this.navPathStack.getParamByName('HouseInfo') as House[];
      if (params.length !== 0 && params !== undefined && params !== null) {
        this.house = params[0];
        let nums: number[] = [];
        this.house.categories.forEach((category: Category) => {
          nums.push(category.details.length);
        });
        let num: number = 0;
        nums.forEach((number: number) => {
          this.idMap.set(num, num + number - 1);
          num += number;
        });
      }
    } catch (e) {
      hilog.error(Constants.DOMAIN, 'testTag', 'Failed to get Params. error: %{public}s', JSON.stringify(e));
    }
  }
```

The guard on the third line is HW-02-0229 - `params.length` before the null
checks. The map construction below it is correct and is the part to copy.

`demo_HouseView.zip#entry/src/main/ets/pages/HouseImagePage.ets:72-111` - the
category strip, including the jump to a category's first photo:

```ts
  @Builder
  indicator() {
    Row() {
      Swiper(this.swiperCtr) {
        ForEach(this.house?.categories, (category: Category, index: number) => {
          Column() {
            Text(category.name + '(' + category.details.length + ')')
              .textAlign(TextAlign.Center)
              .height(30)
              .width(100)
              .fontColor(this.categoryId === index ? '#ff0a59f7' : '#ffffffff');
            Line()
              .margin({ top: 3 })
              .height(2)
              .width(48)
              .startPoint([0, 1])
              .endPoint([48, 1])
              .stroke(this.categoryId === index ? '#ff0a59f7' : '')
              .strokeWidth(2)
              .strokeLineCap(LineCapStyle.Round);
          }
          .onClick(() => {
            this.categoryId = index;
            let arr: Array<[number, number]> = Array.from(this.idMap.entries());
            this.photoId = arr[this.categoryId][0];
            this.swiperCtr.changeIndex(this.photoId);
          });
        }, (index: number) => JSON.stringify(index));
      }
      .displayCount(5)
      .indicator(false)
      .onChange((index: number) => {
        this.categoryId = index;
      });
    }
```

Two findings here: the shared `swiperCtr` (HW-02-0224) and the key generator
whose parameter is named `index` but is a `Category` (HW-02-0231).

`demo_HouseView.zip#entry/src/main/ets/pages/HouseImagePage.ets:113-142` - the
flat photo sequence and the reverse mapping:

```ts
  @Builder
  photoLine() {
    Row({ space: 100 }) {
      Swiper(this.swiperCtr) {
        ForEach(this.house?.categories, (category: Category) => {
          ForEach(category.details, (detail: Detail) => {
            Image(detail.image)
              .objectFit(ImageFit.Cover)
              .height(300)
              .width(400);
          }, (index: number) => JSON.stringify(index));
        }, (index: number) => JSON.stringify(index));
      }
      .indicator(false)
      .onChange((index: number) => {
        this.photoId = index;
        let id = 0;
        this.idMap.forEach((v: number, k: number) => {
          if (this.photoId >= k && this.photoId <= v) {
            this.categoryId = id;
          }
          id += 1;
        });
      });
    }
```

The `onChange` body is the reusable half - a range lookup over the map, with
`id` recovering the category index from insertion order.

`demo_HouseView.zip#entry/src/main/ets/models/HouseModel.ets:25-35` - the
derived unit price, computed once in the constructor:

```ts
  constructor(image: Resource, name: string, description: string, totalPrice: number, size: number, model: string,
    categories: Category[]) {
    this.image = image;
    this.name = name;
    this.description = description;
    this.totalPrice = totalPrice;
    this.size = size;
    this.unitPrice = Math.floor(this.totalPrice / this.size * 10000);
    this.model = model;
    this.categories = categories;
  }
```

`demo_HouseView.zip#entry/src/main/ets/pages/HomePage.ets:18-27` - the two
provided values and the point at which the context becomes available:

```ts
  @Provide navPathStack: NavPathStack = new NavPathStack();
  @Provide uiContext: UIContext | undefined = undefined;
  @StorageLink('topRectHeight') topRectHeight: number | undefined = undefined;
  @StorageLink('bottomRectHeight') bottomRectHeight: number | undefined = undefined;
  tabId: number = 0;
  tabsController: TabsController = new TabsController();

  aboutToAppear(): void {
    this.uiContext = this.getUIContext();
  }
```

`tabId` on the fifth line is HW-02-0225, and the union on `uiContext` against
the consumers' non-optional type is HW-02-0226.

`demo_HouseView.zip#entry/src/main/ets/entryability/EntryAbility.ets:41-67` -
the immersive setup as shipped:

```ts
    // 1. 设置窗口全屏
    let isLayoutFullScreen = true;
    windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(() => {
      hilog.info(DOMAIN, 'testTag', '%{public}s', 'Succeeded in setting the window layout to full-screen mode.');
    }).catch((err: BusinessError) => {
      hilog.error(DOMAIN, 'testTag', '%{public}s',
        'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
    });
    // 2. 获取布局避让遮挡的区域
    let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR; // 以导航条避让为例
    let avoidArea = windowClass.getWindowAvoidArea(type);
    let bottomRectHeight = avoidArea.bottomRect.height; // 获取到导航条区域的高度
    AppStorage.setOrCreate('bottomRectHeight', bottomRectHeight);
    type = window.AvoidAreaType.TYPE_SYSTEM; // 以状态栏避让为例
    avoidArea = windowClass.getWindowAvoidArea(type);
    let topRectHeight = avoidArea.topRect.height; // 获取状态栏区域高度
    AppStorage.setOrCreate('topRectHeight', topRectHeight);
    // 3. 注册监听函数，动态获取避让区域数据
    windowClass.on('avoidAreaChange', (data) => {
```

Step 2 runs before step 1's promise settles (HW-02-0228), and the listener
opened in step 3 is never closed (HW-02-0227).

## Permissions & config

**No permissions.** `demo_HouseView.zip#entry/src/main/module.json5` has no
`requestPermissions` block - the sample is entirely local, with images from the
media resources and data from a static array in `HouseModel.ets`.

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:router_map",
```

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

`oh-package.json5` has no runtime dependencies.

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **A `SwiperController` binds to one `Swiper`.** The reference describes it as
  the controller for *a* `Swiper` component, and calling its APIs when it is
  bound to none raises error 100004.
- **`changeIndex` clamps silently.** From the reference: "If the value specified
  is less than 0 or greater than the maximum page index, the value 0 is used."
  There is no error to observe, which is what makes the shared-controller bug in
  this sample quiet.
- **`$$` needs a decorated variable** for changes to trigger a UI update.
- **`@Provide` and `@Consume` must be the same type** - "Otherwise, implicit
  type conversion occurs, causing application exceptions."
- **`idMap` is built once in `aboutToAppear`.** If the category data can change
  after that, rebuild it - nothing in the page watches for it.
- **The reverse lookup depends on `Map` insertion order**, because the category
  index is recovered by counting iterations rather than being stored.
- **`getWindowAvoidArea` returns px** and this sample stores px, so every
  consumer must call `px2vp` itself.

## Pitfalls

1. **HW-02-0224 - one `SwiperController` drives two `Swiper`s with different
   page counts.** `HouseImagePage.ets:37` creates a single `swiperCtr`, and it
   is passed to both the category `Swiper` (`:75`, `displayCount(5)`) and the
   photo `Swiper` (`:116`). `:97` then calls
   `this.swiperCtr.changeIndex(this.photoId)` with a **photo** index. For the
   first house there are three categories and four photos
   (`HouseModel.ets:59-63`), and with `displayCount(5)` the category Swiper has
   a single page - so any `photoId` above 0 exceeds its maximum index and the
   documented fallback quietly sends it to page 0. Give each Swiper its own
   controller.

2. **HW-02-0225 - the tab index is not state.** `HomePage.ets:22` declares
   `tabId: number = 0;` undecorated, among four decorated neighbours, binds it
   two-way at `:31` with `index: $$this.tabId`, and reads it at `:56`, `:72` and
   `:73` to colour the tab bar. The guidance ties `$$`-driven UI updates to
   variables carrying a state decorator, so the highlight has no reason to move.

3. **HW-02-0226 - `@Provide` and `@Consume` disagree on the type.**
   `HomePage.ets:19` provides `UIContext | undefined`; `HouseImagePage.ets:26`,
   `HouseInfoPage.ets:26` and `HomeComponent.ets:18` all consume plain
   `UIContext`. The guidance is explicit: "They must be of the same type.
   Otherwise, implicit type conversion occurs, causing application exceptions."
   All three consumers then dereference it unguarded in their build methods.

4. **HW-02-0227 - `on('avoidAreaChange')` is never unsubscribed.**
   `EntryAbility.ets:59` subscribes on a `windowClass` local to
   `onWindowStageCreate` (`:34`); `onWindowStageDestroy` (`:78-81`) only logs.
   Store the window on the ability so the unsubscribe has something to call.

5. **HW-02-0228 - the avoid areas are read before the immersive layout
   applies.** `EntryAbility.ets:43-48` chains `setWindowLayoutFullScreen`
   correctly but its `.then()` only logs; `:50-57` read both areas at statement
   level right after. All four pages consume those values as their padding.

6. **HW-02-0229 - the parameter guard checks `length` first.**
   `HouseImagePage.ets:42` is
   `if (params.length !== 0 && params !== undefined && params !== null)`. The
   two null checks exist because the author knew the value might be absent, and
   their position makes them unreachable in that case. The throw lands in the
   catch and is logged as a parameter-fetch failure.

7. **HW-02-0230 - `px2vp` receives possibly-undefined values in three files.**
   `HouseImagePage.ets:27-28` and `:157`, `HouseInfoPage.ets:27-28` and `:218`,
   `HomeComponent.ets:19-20` and `:296` all declare
   `number | undefined = undefined` and then pass the value straight into a
   conversion that takes a `number`. Default them to `0`.

8. **HW-02-0231 - every `ForEach` key is a serialised object.**
   `HouseImagePage.ets:99`, `:123` and `:124` are all
   `(index: number) => JSON.stringify(index)` - the first parameter of a key
   generator is the **item**, so the annotation is false and the value
   serialised is a `Category` or a `Detail`. That is precisely what the default
   generator does, and the guide warns "When item is a complex object,
   serializing it to a JSON string results in a long string that consumes more
   memory." `HomeComponent.ets:228` has no key generator at all, over the
   largest objects in the project. Key on `category.name` / `detail.name` /
   `house.name`.

9. **HW-02-0232 - the contact bar is pinned at `y: 718`.**
   `PhoneLineComponent.ets:31` is `.position({ x: 0, y: 718 })` on a component
   used by two pages whose other sections are sized in percentages (`'9%'`,
   `'12%'`, `'55%'`), and whose padding varies with the safe area. Every offset
   inside the component (`:18`, `:19`, `:20`, `:25`) is a negative margin tuned
   to that one fixed position.

10. **HW-02-0233 - `string.json` exists and is used for nothing.** Grepping the
    whole `ets` directory for `app.string` returns no match, while media and
    colour resources are referenced through `$r` throughout the same files.
    Every user-visible string is a Chinese literal - `HouseImagePage.ets:66`,
    the four tab labels at `HomePage.ets:34`, `:38`, `:42`, `:46`, and
    `PhoneLineComponent.ets:19` and `:23`.

11. **HW-02-0234 - both document snippets misspell identifiers that exist in
    the sample.** The document writes `this.swiperctr` (`:26`, `:36`) where the
    field is `swiperCtr`, and `this.house?.categorys` (`:27`, `:48`) where the
    model field is `categories`. Neither spelling appears anywhere in the ZIP.
    These two snippets are the entire technical content of the page, and neither
    compiles as printed.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Swiper (`SwiperController`, `changeIndex` clamping, `displayCount`,
  `onChange`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- Two-way sync with `$$`:
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- @Provide and @Consume (type-matching rule):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- ForEach (key generation rules and the cost of serialising items):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- Navigation (`getParamByName`, `pushPathByName`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navigation
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
