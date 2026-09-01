---
id: TOUR-08
title: Hotel stay review - multi-dimension star ratings, chips and photo upload
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
sample: huawei_industry_tree/09_tourism/downloads/UserReview.zip
kits: ["@kit.MediaLibraryKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["@ComponentV2", "@Local", "@ObservedV2", "@Trace", ForEach, Navigation, NavPathStack, pushPathByName, pop, NavDestination, NavDestinationContext, "photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, PhotoViewMIMETypes, TextArea, showCounter, Toggle, ToggleType, Grid, GridItem, "UIContext.getPromptAction", showToast]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0047, HW-09-0048, HW-09-0049, HW-09-0050, HW-09-0051, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for a **post-transaction review form**: a list of things the
user has bought or done, each with a "review it" call to action, opening a
form with per-dimension star ratings, a free-text field, quick-pick tags and a
few photos, and returning a "reviewed" flag to the list.

It is the hotel case here, but the shape is the same for a restaurant order,
a tour, a rental return or a delivery. Two parts are worth taking whole: the
**photo picker with no permission** (`PhotoViewPicker` runs out of process, so
the app never asks for gallery access) and the **result flowing back through
`NavPathStack.pop`** so the list updates without shared state.

It is also the clearest example in this industry of **state management V2** -
`@ComponentV2`, `@Local`, `@ObservedV2`, `@Trace` - and of what goes wrong when
half a page uses it and half does not (`HW-09-0048`).

## Feature checklist

- An order list grouped by booking date, newest group first.
- Each order card: order id, reviewed/not-reviewed badge, hotel name, room and
  date and guest lines, price, and a 去评价 (review it) button that hides once
  reviewed.
- Tapping the button opens the review page carrying the order id.
- Four independent star ratings - overall, facilities, service, environment -
  each 1 to 5, each showing a word for the current score.
- A 120-character free-text area with a live counter.
- Four quick-pick tag chips, multi-select.
- Up to three photos from the gallery, each removable, with the add tile
  hidden once three are chosen.
- Publishing returns to the list and flips that order to 已点评 (reviewed).

## Architecture

One `entry` module. Two pages, two model classes, one utility.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model
│   ├── HotelOrder.ets      @ObservedV2, @Trace on orderId and isReviewed
│   └── GroupedOrder.ets    @ObservedV2, @Trace on orders; { date, orders[] }
├── pages
│   ├── Index.ets           @Entry @ComponentV2 - the order list + Navigation host
│   └── UserReviewPage.ets  @ComponentV2 - the NavDestination form
└── utils/FileUtil.ets      photoSelect(maxSelectNumber): Promise<string[]>
```

The documented tree matches the zip.

**The whole page tree is V2.** Both pages are `@ComponentV2` with `@Local`
state; the two models are `@ObservedV2` with `@Trace` on the fields that
change. That is what makes `order.isReviewed = true`, written from a pop
callback in the *list* page, repaint one card without any array copy or manual
notification.

**The reviewed flag travels by route callback, not by shared state:**

```
Index                                        UserReviewPage
  pushPathByName('UserReviewPage',
                 order.orderId,      ──────> onReady: orderId = ctx.pathInfo.param
                 () => order.isReviewed = true)
        ^                                          │
        └──────────────────────────────── pop(this.orderId)
```

`onPop` fires **only when `pop` is called with a result**, per the `Navigation`
reference - so the sample's `pop(this.orderId)` is what arms it, and a back
gesture correctly leaves the order un-reviewed. The captured `order` reference
in the closure is a `@Trace` field on an `@ObservedV2` class, which is why
writing it from outside the card's own builder still repaints.

## Implementation steps

1. **Group the flat order array by date** into `GroupedOrder[]` with a `Map`,
   then sort the groups newest first.
2. **Render group then orders** with two nested `ForEach`. Use `Column` for
   the inner group, not `List` (`HW-09-0051`).
3. **Model anything whose fields change as `@ObservedV2` with `@Trace`** -
   both `HotelOrder.isReviewed` and the review page's chip items
   (`HW-09-0048`).
4. **Push with a callback** and pop with a result.
5. **Read the parameter in `onReady`**, and take the live `pathStack` from the
   same context rather than the field initialiser.
6. **One `@ObservedV2` object per rating dimension**, so a `ForEach` over them
   builds identical rows and a tap writes one `@Trace` number.
7. **Pick photos with `PhotoViewPicker`**, asking for exactly the remaining
   slots.
8. **Submit, then pop** - not the other way round (`HW-09-0047`).

## Verified snippets

All snippets are from `UserReview.zip`. Corrected forms are marked.

**Grouping orders by date — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
private groupOrders(orders: HotelOrder[]): GroupedOrder[] {
  const groupedOrders: GroupedOrder[] = [];
  const orderMap = new Map<string, HotelOrder[]>();

  orders.forEach(order => {
    const key = order.bookingDate;
    if (!orderMap.has(key)) {
      orderMap.set(key, []);
    }
    orderMap.get(key)!.push(order);
  });

  orderMap.forEach((orders, date) => {
    groupedOrders.push(new GroupedOrder(date, orders));
  });

  // newest booking date first
  groupedOrders.sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime());
  return groupedOrders;
}
```

A `Map` keyed on the grouping field, then one pass to materialise the groups,
then a sort. Note the sort is on `new Date(b.date)` - the dates are
`YYYY-MM-DD` strings, which parse reliably; a locale-formatted date string
would not.

**The models — `entry/src/main/ets/model/HotelOrder.ets`** (as shipped)

```typescript
@ObservedV2
export class HotelOrder {
  @Trace orderId: string;
  status: string;
  hotelName: string;
  // ... plain fields: they never change
  @Trace isReviewed: boolean;
}

@ObservedV2
export class GroupedOrder {
  date: string;
  @Trace orders: HotelOrder[];
}
```

**`@Trace` only what changes.** Nine fields, two traced. That is the V2
discipline and it is the reason a review can repaint exactly one badge and one
button: `isReviewed` is traced, `hotelName` is not, so nothing else in the
card is even considered for update.

**Push with a callback, pop with a result — `Index.ets` and `UserReviewPage.ets`** (as shipped)

```typescript
// Index: inside the order card builder, with `order` captured
Button('去评价')
  .visibility(order.isReviewed ? Visibility.Hidden : Visibility.Visible)
  .onClick(() => {
    this.pageInfos.pushPathByName('UserReviewPage', order.orderId, () => {
      order.isReviewed = true;        // @Trace field: repaints this card only
    });
  })

// UserReviewPage: the NavDestination end
.onReady((context: NavDestinationContext) => {
  this.pageInfos = context.pathStack;               // the live stack, not a fresh one
  this.orderId = context.pathInfo.param as string;
})
```

Taking `pathStack` from the `NavDestinationContext` rather than trusting the
field's `new NavPathStack()` initialiser is the detail that makes `pop` work
from inside the destination.

**Rating rows from an observed model — `entry/src/main/ets/pages/UserReviewPage.ets`** (as shipped)

```typescript
@ObservedV2
class ReviewScore {
  dim: string;
  label: string;
  @Trace score: number;
}

@Local reviewScores: ReviewScore[] = [
  new ReviewScore('overall',     '综合评分', 5),
  new ReviewScore('facility',    '酒店设施', 5),
  new ReviewScore('service',     '服务态度', 5),
  new ReviewScore('environment', '酒店环境', 5),
];

@Builder
buildRatingSection(reviewScore: ReviewScore) {
  Row() {
    Row() {
      Text(reviewScore.label + '：')
      Row({ space: 4 }) {                                // FIX: the sample uses a horizontal List
        ForEach([1, 2, 3, 4, 5], (selectedScore: number) => {
          Text(this.starIcon)                            // '★' - a glyph, no image asset
            .fontSize(24)
            .fontColor(selectedScore <= reviewScore.score ? '#FFD700' : '#CCCCCC')
            .onClick(() => {
              reviewScore.score = selectedScore;         // @Trace: repaints this row only
            })
        })
      }
    }
    Text(this.getScoreDescription(reviewScore.score))
  }
  .justifyContent(FlexAlign.SpaceBetween)
}
```

**This is the pattern to copy for any star rating.** One `@ObservedV2` object
per dimension, a `ForEach` over `[1..5]`, and the fill decided by
`selectedScore <= score` - so N taps produce one assignment and the row
repaints itself. Using the `★` glyph rather than two image assets means the
colour is a `fontColor` and the size a `fontSize`, and there is nothing to
scale for density.

**The chips — same file** (corrected, see `HW-09-0048`)

```typescript
@ObservedV2                                   // FIX: the sample declares a bare interface
class QuickDescItem {
  text: string;
  @Trace isSelected: boolean;
}

@Builder
buildQuickButton(quickDescItem: QuickDescItem) {
  Toggle({ type: ToggleType.Button, isOn: quickDescItem.isSelected }) {
    Text(quickDescItem.text)
      .fontColor(quickDescItem.isSelected ? '#0A59F7' : '#B3000000')
  }
  .onChange((isOn: boolean) => {
    quickDescItem.isSelected = isOn;
    // FIX: the sample follows this with `this.quickItems = [...this.quickItems]`
  })
}
```

`Toggle({ type: ToggleType.Button })` with a `Text` child is the built-in
chip: it carries its own selected state and background, so only the label
colour has to be driven. With `@Trace` on `isSelected` the array copy the
sample needs disappears.

**Picking photos with no permission — `entry/src/main/ets/utils/FileUtil.ets`** (as shipped)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';

export async function photoSelect(maxSelectNumber: number): Promise<Array<string>> {
  const photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = maxSelectNumber;

  const photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  try {
    const photoSelectResult = await photoViewPicker.select(photoSelectOptions);
    if (photoSelectResult && photoSelectResult.photoUris && photoSelectResult.photoUris.length > 0) {
      return photoSelectResult.photoUris;
    }
    return [];
  } catch (err) {
    if (err instanceof Error) {
      hilog.error(0, 'SelectedImage failed: %{public}s\nStack: %{public}s',
        err.message, err.stack || 'No stack available');
    }
    return [];
  }
}
```

**`PhotoViewPicker` needs no permission at all** - it runs in the system
gallery process and hands back URIs the app is granted read access to. That is
why this sample's `module.json5` has an empty `requestPermissions`, and it is
the right default for "let the user attach a few photos". Reach for
`ohos.permission.READ_IMAGEVIDEO` only when the app must enumerate the gallery
itself.

Returning `[]` on both the empty and the error path means the caller can
always spread the result. The call site asks for exactly the free slots:

```typescript
async selectImage() {
  const uris = await photoSelect(3 - this.selectedImageUris.length);
  this.selectedImageUris.push(...uris);      // @Local observes Array push/splice
}
```

`@Local` observes the array mutation APIs - `push`, `pop`, `shift`, `unshift`,
`splice`, `copyWithin`, `fill`, `reverse`, `sort` - so `push` and the
delete-button's `splice(index, 1)` both repaint without a copy. It is only
*property changes on the objects inside* an array that V2 cannot see.

**Publishing — same file** (corrected, see `HW-09-0047`)

```typescript
Button('发表')
  .onClick(async () => {
    await this.submitReview();          // FIX: the sample fires a 1 s setTimeout
    this.pageInfos.pop(this.orderId);   //      and pops immediately, before it runs
  })
```

The shipped order - `setTimeout(submit, 1000)` then `pop()` - runs the submit
on a component that no longer exists, from a timer with no stored handle.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, and that is correct:
`PhotoViewPicker` is a system picker.

Routing is by `routerMap`:

```json5
// entry/src/main/module.json5
"routerMap": "$profile:router_map"
```

```json
// entry/src/main/resources/base/profile/router_map.json
{
  "routerMap": [
    {
      "name": "UserReviewPage",
      "pageSourceFile": "src/main/ets/pages/UserReviewPage.ets",
      "buildFunction": "UserReviewPageBuilder"
    }
  ]
}
```

Resource directories: `base` and `dark`. Colours and floats are resourced;
none of the text is (`HW-09-0049`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **State management V2** throughout - `@ComponentV2`, `@Local`,
  `@ObservedV2`, `@Trace`. Do not mix V1 decorators into these components.
- The photo limit is **3**, expressed twice: `3 - length` in `selectImage` and
  `length <= 2` on the add tile. Change both together.
- `TextArea` is capped at 120 characters via `maxLength`, with
  `showCounter(true, { highlightBorder: false })`.
- The URIs from `PhotoViewPicker` are temporary grants scoped to the app -
  fine for display, but a real upload must read the file while the grant
  holds, not persist the URI string.
- **Nothing is submitted.** `submit2Remote()` is an empty method; the ratings,
  text, chips and photos are collected and dropped.
- The four tabs on the list page are decorative: index 3 is hard-coded as the
  selected one and there is no filtering.

## Pitfalls

- **`HW-09-0047` — the publish button pops the page and then submits a second
  later** from an uncancellable `setTimeout`, so the callback runs on a
  destroyed component and the order is flagged reviewed before the request
  starts.
- **`HW-09-0048` — `QuickDescItem` is a bare interface inside an `@Local`
  array,** so its `isSelected` change is invisible to V2 and the file
  compensates by copying the array on every toggle. `ReviewScore` in the same
  file shows the correct `@ObservedV2` / `@Trace` form.
- **`HW-09-0049` — every user-visible string on both pages is a hardcoded
  Chinese literal** and there is no locale directory, although colours and
  floats are properly resourced.
- **`HW-09-0050` — `reviewText` and `uploadedImages` are dead `@Local`
  fields** duplicating `comment` and `selectedImageUris`. Whoever implements
  `submit2Remote` has a fifty-fifty chance of reading the empty pair.
- **`HW-09-0051` — `List` is used as a layout container** - nested inside a
  `Scroll` for the order groups, and around five static stars - so a scroll
  container with lazy layout is created where a `Column` or `Row` is meant.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-new-local.md` - `@Local`, what it observes, and the nested-object limitation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-local
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2` and `@Trace`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `pushPathByName`, `pop`, and when `onPop` fires
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `onReady` and `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and its key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/04_media/js-apis-photoAccessHelper.md` - `PhotoViewPicker`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-toggle.md` - `ToggleType.Button` as a chip
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-toggle
- `TOUR-10` - the order list this review flow returns to; `TOUR-11` - the form pattern earlier in the funnel
