---
id: SHOP-13
title: Order status tabs and the post-receipt review page - one @Provide array drives every tab, every button and every label
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/13_evaluation_after_received.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/evaluation_after_received-0000002356153881
sample: huawei_industry_tree/16_shopping/downloads/EvaluationAfterReceived.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, dataSharePredicates, hilog, image, photoAccessHelper, util, window, "@Provide", "@Consume", Navigation, NavPathStack, NavDestination, Tabs, TabContent, SubTabBarStyle, Rating, TextArea, Grid, GridItem, "PhotoViewPicker.select", "PhotoAccessHelper.getAssets", "PhotoAsset.getThumbnail", "ImagePacker.packToData"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0014, HW-16-0015, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to show **the same list of records under
several status tabs, with per-record actions that move a record from one tab
to the next**. The order list of a shopping app is the canonical case: 待付款
(to pay) → 待发货 (to ship) → 待收货 (to receive) → 待评价 (to review) →
已评价 (reviewed), each with its own action button, plus a review composer
that the last-but-one status opens.

The pattern is one array of status codes held by the page in a `@Provide`, and
read - and written - by every child through `@Consume`. Nothing else carries
status: the tab filter, the status caption, the button row and the review
page's completion write are all views of that one array. It generalises to
ticket workflows, delivery states, moderation queues, anything where a record
walks a fixed state machine and several distant components must agree on where
it is.

The second half of the card is the review composer: a `Rating`, a `TextArea`
and a picker-backed photo grid. **Read `HW-16-0014` before copying the photo
code** - the sample routes picker URIs back through
`photoAccessHelper.getAssets`, which needs a restricted permission it never
declares, and which the picker exists precisely to avoid.

## Feature checklist

- An order list under five sub-tabs: all, to pay, to ship, to receive, to
  review. A tab with no matching order shows an empty-state image and 暂无订单.
- Each order card shows the shop, the goods, the price line, a status caption
  in blue, and a three-button row whose labels depend on the status.
- The right-hand (primary) button advances the order one status; the caption,
  the button labels and the tab the card lives in all change together.
- At 待评价 the primary button is 立即评价 and pushes the review page instead
  of advancing the status.
- The review page shows a photo grid with an "add" tile, a star `Rating`, a
  free-text `TextArea`, an anonymous checkbox and a submit button.
- Submitting with zero stars raises a toast and does nothing.
- Submitting with a rating sets the order to 已评价 and pops back; the order
  is now in the 已评价 branch of the button table.
- The photo grid accepts at most 6 images; the add tile hides when full, and
  each thumbnail carries a delete badge.

## Architecture

One `entry` module, two pages, two components, three model files.

```
entry/src/main/ets
├── components
│   ├── AddPic.ets          the photo grid: picker -> thumbnail -> ImageInfo[]
│   └── SingleOrder.ets     one order card; owns the status->buttons table
├── entryability/EntryAbility.ets      avoid areas -> AppStorage, full screen
├── entrybackupability/EntryBackupAbility.ets
├── model
│   ├── Constants.ets       sizes, colours, MAX_PICTURES = 6, Status labels
│   ├── ContentInfo.ets     interface ImageInfo { imagePixelMap, imageArrayBuffer?, imageName }
│   └── GoodsList.ets       Order interface, 5 static orders, ORDER_STATUSES
└── pages
    ├── EvaluationPage.ets  the review composer, a NavDestination
    └── MainPage.ets        @Entry: Navigation + the five Tabs
```

The documented tree matches the zip exactly. `EvaluationPage` is reached by
name through `route_map` (`$profile:route_map` in `module.json5`), not by a
`pages` entry, which is why `MainPage` is the only `@Entry`.

**The design decision worth copying** is that the order's *status* is not a
field of the order. `GoodsList.ets` defines `Order.status` and then
immediately derives a separate array from it:

```typescript
export const ORDER_STATUSES: number[] = orders.map(order => order.status);
```

`orders` stays an immutable catalogue; `ORDER_STATUSES` is the mutable
parallel array, published once by `MainPage` as `@Provide orderStatuses` and
indexed by list position. Every consumer - the tab filter, the caption, the
button table, the review page's completion write - addresses it by the same
index. That split is what lets the review page, three levels away from the
list and on a different navigation destination, flip a status and have the
card jump tabs on the way back without any callback plumbing or event bus.

The cost is that the array is positional - see the last pitfall.

## Implementation steps

1. **Publish the status array once** in the `@Entry` page:
   `@Provide orderStatuses: number[] = ORDER_STATUSES;`, and publish the
   `NavPathStack` alongside it as `@Provide('pageInfos')`.
2. **Filter, do not partition.** Each `TabContent` renders the same `ForEach`
   over `orders` and shows a card only when `this.orderStatuses[index] ===
   status`; the "all" tab passes a flag instead of a status.
3. **Guard the empty tab** with `orders.some(...)` in the builder, so a tab
   with no matching order renders the empty-state column rather than an empty
   `List`.
4. **Consume the array in the card** (`@Consume orderStatuses: number[]`) and
   read it - never a copy - for the caption and for the button table.
5. **Advance the status in place** with a bounds check
   (`if (this.orderStatuses[this.statusIndex] < 3)`); the last transition, to
   4, belongs to the review page.
6. **Push the review page by name** with a typed param object
   (`{ orderNumber: this.statusIndex }`), and read it back in `onReady` via
   `getParamByName`.
7. **Reassign the array after the final write** - `this.orderStatuses =
   [...this.orderStatuses]` - because an element assignment on a `number[]`
   does not notify `@Provide`/`@Consume` observers.
8. **Select photos with `PhotoViewPicker`,** and build thumbnails from the
   returned URI with `image.createImageSource(uri)`; do not feed the URI back
   into `photoAccessHelper.getAssets`, which requires the restricted
   `READ_IMAGEVIDEO` permission (`HW-16-0014`).
9. **Keep the packed bytes binary.** Store the `ArrayBuffer`, or base64 it
   with `util.Base64Helper`; never run JPEG bytes through a UTF-8
   `TextDecoder` (`HW-16-0015`).
10. **Cap the selection twice**: pass the remaining budget as the picker's
    `maxSelectNumber`, and re-check inside the loop that consumes the result.

## Verified snippets

All snippets are from `EvaluationAfterReceived.zip`. Corrected forms are marked.

**The status table — `entry/src/main/ets/components/SingleOrder.ets`** (as shipped)

```typescript
@Component
export struct SingleOrder {
  @Prop order: Order;
  @Consume('pageInfos') pageInfos: NavPathStack;
  @Consume orderStatuses: number[];
  @Prop statusIndex: number;

  @Builder
  functionButton(content: string | ResourceStr, isEvaluationButton: boolean, isRightButton: boolean) {
    Button(content)
      .fontColor(isRightButton ? Color.White : Constants.FONT_COLOR_ONE)
      .backgroundColor(isRightButton ? Constants.BLUE : Constants.BUTTON_COLOR_TWO)
      .onClick(() => {
        if (isEvaluationButton) {
          const params: EvaluationPageParams = { orderNumber: this.statusIndex };
          this.pageInfos.pushPathByName('EvaluationPage', params);
        } else if (isRightButton) {
          this.updateOrderStatus();
        }
      });
  }

  private updateOrderStatus() {
    if (this.orderStatuses[this.statusIndex] < 3) {
      this.orderStatuses[this.statusIndex] += 1;
    } else {
      hilog.warn(0x0000, 'ORDER_STATUS', '订单状态已达到最大值，无法再更新。');
    }
  }

  @Builder
  buttonView() {
    if (this.orderStatuses[this.statusIndex] === 0) {
      this.button(Status.CLOSE_ORDER, Status.CHANGE_ADDRESS, Status.PAY_NOW, false);
    } else if (this.orderStatuses[this.statusIndex] === 1) {
      this.button(Status.REQUEST_REFUND, Status.BOOK_AGAIN, Status.RUSH_DELIVERY, false);
    } else if (this.orderStatuses[this.statusIndex] === 2) {
      this.button(Status.REQUEST_REFUND, Status.VIEW_LOGISTICS, Status.CONFIRM_RECEIPT, false);
    } else if (this.orderStatuses[this.statusIndex] === 3) {
      this.button(Status.REQUEST_REFUND, Status.BOOK_AGAIN, Status.COMMENT_NOW, true);
    } else if (this.orderStatuses[this.statusIndex] === 4) {
      this.button(Status.REQUEST_REFUND, Status.BOOK_AGAIN, Status.ADD_COMMENT, false);
    }
  }
}
```

**Three flags carry the whole button row.** `functionButton` takes only the
label plus `isEvaluationButton` and `isRightButton`, so the same builder makes
a secondary grey button, the blue primary that advances the status, and the
blue primary that navigates instead. `buttonView` is then a plain lookup table
from status to three labels - readable, and trivially extended with a sixth
status. The `@Consume` is a `number[]`, so the element write in
`updateOrderStatus` mutates the array the page owns: no callback goes up.

`updateOrderStatus` stops at 3 deliberately. Status 4 (已评价) is not reachable
by pressing a button on the card; only a submitted review sets it, which is
what makes 立即评价 a navigation and not a transition.

**The tab filter — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct MainPage {
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();
  @Provide orderStatuses: number[] = ORDER_STATUSES;

  aboutToAppear(): void {
    AppStorage.setOrCreate('pathStack', this.pageInfos);
  }

  @Builder
  getListView(status: number | null, isAllOrders: boolean) {
    List() {
      ForEach(orders, (order: Order, index: number) => {
        ListItem() {
          if (isAllOrders) {
            SingleOrder({ order: order, statusIndex: index });
          } else if (this.orderStatuses[index] === status) {
            SingleOrder({ order: order, statusIndex: index });
          }
        };
      }, (order: Order) => order.id.toString());
    }
    .scrollBar(BarState.Off);
  }

  @Builder
  getTabContent(status: number | null, isAllOrders: boolean) {
    if (isAllOrders || orders.some((order: Order, index: number) => this.orderStatuses[index] === status)) {
      this.getListView(status, isAllOrders);
    } else {
      Column() {
        Image($r('app.media.no_order'))
        Text($r('app.string.nothing'))          // 暂无订单
      }
    }
  }
}
```

**The filter lives inside the `ForEach`, not around it.** Every tab iterates
the full `orders` array and emits an empty `ListItem` for the non-matching
ones. That keeps `order.id` usable as a stable key across all five tabs and
avoids rebuilding a filtered array on each status change, at the price of five
empty list items per tab. For five orders that is free; past a few hundred
rows, build the filtered array in a `@Computed`-style getter instead and pay
the key-stability cost.

`getTabContent` is the empty-state guard, and it has to re-run the same
predicate that the `ForEach` applies - the one duplication this shape forces.
`pageInfos` is additionally mirrored into `AppStorage` because
`EvaluationPage`, being reached through the route map, re-reads it in
`aboutToAppear` before its `@Consume` is guaranteed to be bound.

**Picker to thumbnail — `entry/src/main/ets/components/AddPic.ets`** (corrected, see `HW-16-0014`, `HW-16-0015`)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { image } from '@kit.ImageKit';
import { util } from '@kit.ArkTS';

async fileSelect(): Promise<Array<string>> {
  let imgUri: Array<string> = [];
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = Constants.MAX_PICTURES - this.imageUriArray.length;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();
  try {
    let photoSelectResult = await photoPicker.select(photoSelectOptions);
    if (photoSelectResult && photoSelectResult.photoUris && photoSelectResult.photoUris.length > 0) {
      imgUri = photoSelectResult.photoUris;
      return imgUri;
    }
    return [];
  } catch (error) {
    hilog.error(0x0000, '[FileUtil]', `PhotoViewPicker failed with err: ${error.code}, ${error.message}`);
    return [];
  }
}

selectImage(): void {
  if (this.imageUriArray.length >= Constants.MAX_PICTURES) {
    return;
  }
  this.fileSelect().then((uri: Array<ResourceStr>) => {
    for (let i = 0; i < uri.length; i++) {
      const item = uri[i];
      if (this.imageUriArray.length >= Constants.MAX_PICTURES) {
        break;                       // 达到最大数量，终止循环
      }
      if (item) {                    // FIX: shipped code tests `uri`, the array itself
        this.getThumbnail(item as string);
      }
    }
  });
}

// FIX: shipped getThumbnail routes the picker URI through
// phAccessHelper.getAssets(fetchOption) + asset.getThumbnail(), which needs
// the restricted ohos.permission.READ_IMAGEVIDEO the sample never declares.
async getThumbnail(uri: string) {
  const source: image.ImageSource = image.createImageSource(uri);
  const pixelMap: image.PixelMap = await source.createPixelMap({
    desiredSize: { width: 256, height: 256 }      // FIX: the thumbnail size getThumbnail used to pick
  });
  const imageName = uri.substring(uri.lastIndexOf('/') + 1);
  const data: ArrayBuffer = await this.pixelMapToBuffer(pixelMap);
  // FIX: shipped code does TextDecoder.create('utf-8').decodeToString(new Uint8Array(data))
  const encoded: string = new util.Base64Helper().encodeToString(new Uint8Array(data));
  this.imageUriArray.push({ imagePixelMap: pixelMap, imageArrayBuffer: encoded, imageName: imageName });
  source.release();
}

async pixelMapToBuffer(pixelMap: image.PixelMap): Promise<ArrayBuffer> {
  const imagePackerApi: image.ImagePacker = image.createImagePacker();
  let packOpts: image.PackingOption = { format: 'image/jpeg', quality: Constants.QUALITY };
  return await imagePackerApi.packToData(pixelMap, packOpts);
}
```

**The picker already grants what the sample then asks permission for.**
`PhotoViewPicker.select` runs in a system UI extension and returns URIs the
calling app may read - that is the entire point of the picker, and why the
zip's `module.json5` declares no `requestPermissions` at all. Feeding those
URIs back into `photoAccessHelper.getAssets` re-enters the album database,
whose reference states plainly that it requires
`ohos.permission.READ_IMAGEVIDEO`; that permission is ACL-restricted and an
ordinary review composer will not be granted it. `image.createImageSource(uri)`
reads the same file through the granted URI with no permission at all, and
`desiredSize` on `createPixelMap` does the downscaling `getThumbnail` was
there for.

**`maxSelectNumber` is computed, not constant.** Passing
`MAX_PICTURES - imageUriArray.length` means the picker itself refuses the
seventh image, so the in-loop `break` is a second line of defence rather than
the only one. The shipped loop's guard is `if (uri)` - the array, always
truthy - so a null entry would reach `getThumbnail`; the corrected form tests
`item`.

**Encoding is the part that silently destroys data.** `packToData` returns
JPEG bytes. Running them through a UTF-8 `TextDecoder` replaces every invalid
byte sequence with U+FFFD, irreversibly - the resulting string looks like a
payload, uploads like a payload, and can never be decoded back to the image.
Keep the `ArrayBuffer`, or base64 it if the model really needs a string.

**The review submit — `entry/src/main/ets/pages/EvaluationPage.ets`** (as shipped)

```typescript
@Consume('pageInfos') pageInfos: NavPathStack;
@Consume orderStatuses: number[];
@State rating: number = 0;
@State currentOrder: number = 0;

Rating({ rating: this.rating, indicator: false })
  .stars(5)
  .stepSize(1)
  .onChange((value: number) => {
    this.rating = value;
  });

Button($r('app.string.commit'))
  .onClick(() => {
    if (this.rating === 0) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.must_rate') });
      return;
    }
    this.orderStatuses[this.currentOrder] = 4;
    this.orderStatuses = [...this.orderStatuses];      // re-notify the @Provide
    this.pageInfos.pop();
  });

// in build():
.onReady(() => {
  let indexStr: string = JSON.stringify(this.pageInfos.getParamByName('EvaluationPage'));
  let indexObj: Array<EvaluationPageParams> = JSON.parse(indexStr);
  if (indexObj.length > 0) {
    this.currentOrder = indexObj[0].orderNumber;
  }
})
```

**The two lines that look redundant are the load-bearing ones.** Writing
`this.orderStatuses[this.currentOrder] = 4` mutates the array element;
`@Provide`/`@Consume` observe the *reference*, so on its own that write would
update nothing on the list page until some other event forced a rebuild.
Reassigning a spread copy is what actually propagates the change back through
the provider to every card and every tab filter. (`SingleOrder`'s
`updateOrderStatus` gets away without it only because the card that performs
the write is itself re-evaluated by the same tap.)

`getParamByName` returns an **array** of every param pushed under that name,
which is why the read is `indexObj[0]` after a `JSON.stringify`/`parse`
round-trip. `Rating` with `indicator: false` is the interactive form;
`stepSize(1)` restricts it to whole stars, matching the integer score a review
API expects.

## Permissions & config

**None.** The zip's `module.json5` declares no `requestPermissions` array at
all - the correct outcome for a picker-based photo flow, and exactly why
`HW-16-0014` is a defect rather than a missing declaration: the shipped code
calls an API that needs a restricted permission which must not be added here.

`route_map` is declared as `"routerMap": "$profile:route_map"`, so
`EvaluationPage` is resolved by name through `pushPathByName` and only
`MainPage` sits in `main_pages`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` are phone, tablet, 2in1.
- `MAX_PICTURES` is 6 and the grid is `columnsTemplate('1fr 1fr 1fr')` with a
  hardcoded two-height switch (`this.imageUriArray.length < 3 ? ONE_ROW :
  TWO_ROW`), so the grid does not grow past two rows - fine at 6, wrong if the
  cap is raised.
- Orders, statuses and the review body are all in memory. Nothing is
  submitted, persisted or restored - the anonymous checkbox raises a
  "demo only" toast.
- `Order.status` is read once at module load to seed `ORDER_STATUSES` and is
  stale from then on; do not read it anywhere else.

## Pitfalls

- **`HW-16-0014`** (D/high, confirmed): `AddPic.getThumbnail` sends the
  picker's URI through `phAccessHelper.getAssets(fetchOption)` and
  `asset.getThumbnail()`, an API whose reference requires
  `ohos.permission.READ_IMAGEVIDEO`; the sample declares no permissions, so at
  runtime no thumbnail is ever produced, and the permission is ACL-restricted
  so app-market apps generally cannot obtain it for this use case. Fix:
  replace the `getAssets`/`getThumbnail` chain with
  `image.createImageSource(uri)` + `createPixelMap({ desiredSize })`, which
  reads the granted URI with no permission at all.
- **`HW-16-0015`** (B/medium, confirmed): the packed JPEG `ArrayBuffer` is run
  through `util.TextDecoder.create('utf-8').decodeToString(...)` and the
  resulting string is stored as `imageArrayBuffer`, corrupting the binary
  irreversibly; the same method's loop guard is `if (uri)` (the array, always
  truthy) instead of `if (item)`. Fix: store the `ArrayBuffer` directly, or
  base64 it with `util.Base64Helper().encodeToString(new Uint8Array(data))`,
  and fix the guard.
- **Element writes on a `@Provide` array do not notify.** The review page
  works only because it follows the element write with
  `this.orderStatuses = [...this.orderStatuses]`. Any new writer must do the
  same, or the list will not move the card.
- **`statusIndex` is a list position, not an id.** Sort or paginate `orders`
  and every status lands on the wrong record. Key the status map by
  `order.id` before adopting this in a real order list.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - `@Provide`/`@Consume` and what counts as an observed change
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions.maxSelectNumber`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getAssets` and its `READ_IMAGEVIDEO` requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/04_media/js-apis-photoaccesshelper.md` - `FetchOptions`, `PhotoAsset.getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `createImageSource`, `createPixelMap`, `desiredSize`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` and `PackingOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-rating.md` - `Rating`, `indicator`, `stepSize`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-rating
- `huawei_industry_tree/16_shopping/docs/13_evaluation_after_received.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/evaluation_after_received-0000002356153881
