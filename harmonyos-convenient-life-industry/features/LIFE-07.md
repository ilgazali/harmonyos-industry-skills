---
id: LIFE-07
title: Expandable bill list - day summary rows that unfold into transaction detail, with a segmented income/expense filter
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
sample: huawei_industry_tree/02_convenient_life/downloads/CollapseList.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItem, ListItemGroup, divider, Scroll, Progress, ProgressType, Search, Stack, Blank, "@Observed", "@ObjectLink", "@State", "@StorageProp", "@Builder", safeAreaPadding, stateEffect, "resourceManager.getRawFileContentSync", "util.TextDecoder", "util.TextDecoderOptions", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "AppStorage.setOrCreate"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0043, HW-02-0044, HW-02-0045, HW-02-0046, HW-02-0047, HW-02-0048, HW-02-0049, HW-02-0050, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when a list has **two levels that share one scroll surface** - a
summary row per group, and the individual records inside it revealed on tap.
A bank statement is the canonical case: one row per day showing the total and a
budget bar, unfolding into the individual transactions.

The sample's answer is a `List` of summary rows where each row conditionally
renders a second `List` of children. That is the part to *not* copy
(`HW-02-0043`); the right primitive is `ListItemGroup`, which the `List`
reference names for exactly this shape. What the sample does get right is
everything around it: an `@Observed` model that carries the expansion flag so it
survives a tab switch, a segmented filter that swaps the data source, and
totals computed once per group.

Take this card for statements, order histories, timesheets, attendance logs -
anything grouped by date with a per-group aggregate.

## Feature checklist

- A search field, a three-way segmented filter (全部 / 支出 / 收入 - all /
  expense / income), and a month summary card showing the two totals.
- The filter swaps both the list contents and which totals the summary shows.
- One row per day: the date in a rounded badge, the day's total, a `Progress`
  bar against a budget, the transaction count, and a chevron.
- The total and the bar turn red when the day exceeds its budget.
- Tapping a row toggles the chevron and unfolds the day's transactions.
- Expansion state survives switching between the three filter tabs.
- Expense and income rows use different budgets (500 and 10000), different
  icons, and different sign prefixes.

## Architecture

One `entry` module. The data is two rawfile JSONs read synchronously at page
start.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage
├── model
│   ├── ListModel.ets               ListModel, ChildrenObject, @Observed ListModelData
│   ├── SpendDataItem.ets           @Component - one expense day row  (a view, despite the folder)
│   └── IncomeDataItem.ets          @Component - one income day row
└── pages/BillPage.ets              @Entry - JSON loading, filter, totals, the three lists
resources/rawfile/spend_data.json   6 days, type 0
resources/rawfile/income_data.json  6 days, type 1
```

The documented tree matches the zip exactly - including the two row components
living under `model/`, which is where the document puts them too.

**The model is three shapes, and only one of them is observed.**

```typescript
class ListModel        { data: Array<ListModelData> }      // the JSON envelope
interface ChildrenObject { id, time, title, company, price }  // a transaction
@Observed class ListModelData { id?, type?, isExpand, month?, day, children[] }
```

`type` is the discriminator - 0 expense, 1 income - and it is what
`listBuilder` switches on to pick a row component. `isExpand` is the only
mutable field, and `@Observed` exists for it.

**Three arrays, one derived.** `spendList` and `incomeList` come straight from
the two files; `billList` is `spendList.concat(incomeList)` sorted by day. The
concat is shallow, so **the same `ListModelData` object appears in two lists** -
which is exactly why expansion survives a tab switch: the 全部 tab and the 支出
tab render the same object, and `aboutToAppear` re-seeds the row's local flag
from it.

```
tap a row  ->  isExpand (local @State) flips     -> chevron + detail list re-render
           ->  item.isExpand (model) flips       -> nothing reads it...
switch tab ->  row component destroyed, rebuilt
           ->  aboutToAppear: isExpand = item.isExpand   <- ...except here
```

That round trip is the whole persistence mechanism, and the document omits the
half that makes it work (`HW-02-0046`). It also means the `@ObjectLink` binding
never actually delivers a change (`HW-02-0045`).

## Implementation steps

1. **Model the group with a discriminator and an expansion flag.** Make the
   group class `@Observed`; leave the leaf an `interface` - nothing mutates it.
2. **Read the rawfiles** with
   `context.resourceManager.getRawFileContentSync(name)`, decode with
   `util.TextDecoder.create('utf-8', { ignoreBOM: true })`, and `JSON.parse`.
   Import `util` from `@kit.ArkTS`, not `@ohos.util` (`HW-02-0049`).
3. **Derive the merged list once** in `aboutToAppear` and sort it on a full
   date, not just the day-of-month (`HW-02-0050`).
4. **Build the group row as `ListItemGroup`, not a nested `List`**
   (`HW-02-0043`): the summary is the `header`, the transactions are the items.
   That keeps one scroll surface and lets both levels virtualise.
5. **Read and write the expansion flag on the model,** not through a local
   `@State` copy (`HW-02-0045`). `@ObjectLink` on an `@Observed` group then
   re-renders the row by itself and the `aboutToAppear` re-seeding disappears.
6. **Key every `ForEach` explicitly** (`HW-02-0044`). For the merged list the id
   alone is not unique - the two rawfiles reuse the same numbers - so combine it
   with `type`.
7. **Compute the group total once** and render it through a `Progress` with
   `total` set to the budget; drive the over-budget colour from the same
   comparison.
8. **Render the budget label from the budget constant** (`HW-02-0048`), and let
   `justifyContent` place it rather than a hand-tuned margin.
9. **Gate the summary card and the list on the same `selectedIndex`,** so the
   filter changes both together.

## Verified snippets

All snippets are from `CollapseList.zip`. Corrected forms are marked.

**The model - `CollapseList.zip#entry/src/main/ets/model/ListModel.ets`** (as shipped)

```typescript
export class ListModel {
  data: Array<ListModelData> = [];        // matches the JSON envelope: { "data": [...] }
}

export interface ChildrenObject {
  id: string;
  time: string;
  title: string;
  company: string;
  price: number;
}

@Observed
export class ListModelData {
  id?: string;
  type?: number;                          // 0 = spend, 1 = income - the discriminator
  isExpand: boolean = false;
  month?: string;                         // display only: "10月"
  day: number = 0;
  children: ChildrenObject[] = [];
}
```

**`ListModel` exists purely so `JSON.parse(...) as ListModel` has a shape to
cast to.** The rawfile is `{ "data": [ ... ] }`, and the cast makes
`spendModel.data` typed without a runtime validation step - which is the usual
ArkTS trade: the cast is unchecked, so a malformed file surfaces as an undefined
property much later.

`month` is a string with the 月 suffix baked in, which is what makes the sort
day-only (`HW-02-0050`).

**Loading the rawfiles - `CollapseList.zip#entry/src/main/ets/pages/BillPage.ets:70`** (as shipped)

```typescript
aboutToAppear() {
  let mContext = this.getUIContext()?.getHostContext() as common.UIAbilityContext;
  this.getJson1(mContext);
  this.getJson2(mContext);
  this.billList = this.spendList.concat(this.incomeList);
  this.billList.sort((a, b) => b.day - a.day);
  for (let i = 0; i < this.spendList.length; i++) {
    for (let j = 0; j < this.spendList[i].children.length; j++) {
      this.spendMoney += this.spendList[i].children[j].price;
    }
  }
  // ... same for incomeMoney ...
}

getJson1(context: common.UIAbilityContext) {
  let dataJson = context.resourceManager.getRawFileContentSync('spend_data.json');
  let textDecoderOptions: util.TextDecoderOptions = { ignoreBOM: true };
  let textDecoder = util.TextDecoder.create('utf-8', textDecoderOptions);
  let result = textDecoder.decodeToString(dataJson, { stream: false });
  let spendModel: ListModel = JSON.parse(result) as ListModel;
  this.spendList = spendModel.data;
}
```

**`getRawFileContentSync` returns a `Uint8Array`, not a string** - the decode
step is not optional. `ignoreBOM: true` is what stops a UTF-8 BOM ending up as a
leading `﻿` in the parsed text, which would make `JSON.parse` throw; it is
the single most common cause of "my rawfile JSON will not parse".

`{ stream: false }` tells the decoder this is the whole input, so it flushes any
trailing partial sequence rather than holding it for a next chunk.

**`concat` is shallow, and that is load-bearing.** `billList` holds the same
objects as `spendList`/`incomeList`, so expanding a row in one tab is visible in
the other. Deep-copying here would break the feature.

**The row - `CollapseList.zip#entry/src/main/ets/model/IncomeDataItem.ets:23`** (corrected, see `HW-02-0045` and `HW-02-0048`)

```typescript
const BUDGET = 10000;

@Component
export struct IncomeDataItem {
  @ObjectLink item: ListModelData;
  @State money: number = 0;
  // FIX: the sample also declares @State isExpand and seeds it from the model
  //      in aboutToAppear; read this.item.isExpand directly instead.

  aboutToAppear(): void {
    for (let i = 0; i < this.item.children.length; i++) {
      this.money += this.item.children[i].price;
    }
  }

  build() {
    Column() {
      Column() {
        Row() {
          Column() {                                        // the date badge
            Text(`${this.item.day}日`);
            Text(this.item.month).opacity(0.5);
          }
          .width(54).height(49).borderRadius(20)
          .backgroundColor($r('app.color.background'));

          Column() {
            Row() {
              Text(this.money.toString())
                .width(60)
                .fontColor(this.money <= BUDGET
                  ? $r('app.color.button_background') : $r('app.color.income_more_budget'));
              Text(`${BUDGET}`);                            // FIX: sample writes Text('10000')
            }
            Progress({ value: this.money, total: BUDGET, type: ProgressType.Linear })
              .width(202)
              .color(this.money <= BUDGET
                ? $r('app.color.button_background') : $r('app.color.income_more_budget'));
            Row() {
              Text(`${this.item.children.length}`).opacity(0.5);
              Text($r('app.string.income_number')).opacity(0.5);
              Text($r('app.string.income_money')).opacity(0.5);
            }
          }
          .alignItems(HorizontalAlign.Start);

          Image(this.item.isExpand                          // FIX: was this.isExpand
            ? $r('app.media.arrow_top') : $r('app.media.arrow_bottom'))
            .width(15).height(8);
        }
        .onClick(() => {
          this.item.isExpand = !this.item.isExpand;         // FIX: one source of truth
        })
        .height(80).width('100%');

        if (this.item.children.length > 0 && this.item.isExpand) {
          this.expandChildItem();
        }
      }
      .backgroundColor(Color.White).borderRadius(20);
    }
    .alignItems(HorizontalAlign.Start);
  }
}
```

**`Progress({ value, total })` clamps on its own**, so the over-budget case
needs no guard on the bar - only on the colour, which is why the same
`money <= BUDGET` ternary appears twice. Pulling it into a getter would be an
improvement the sample does not make.

**The `if` inside `build()` is the collapse.** ArkUI's conditional rendering
removes the subtree entirely rather than hiding it, so a collapsed day costs
nothing - which is the one thing the nested-`List` approach does have going for
it.

`money` is summed once in `aboutToAppear` and never recomputed; the data is
static here, but a live statement would need a `@Watch` on `children`.

**The detail list - same file, line 120** (corrected, see `HW-02-0043`)

```typescript
// As shipped: a second, unsized List inside the outer List's ListItem.
//   @Builder expandChildItem() {
//     List() { ForEach(this.item.children, (item: ChildrenObject) => { ListItem() { ... } }); }
//   }
// The List reference: "If a scrollable component is nested in a List component,
// their scrolling directions are the same, and the main axis size is not set for
// the List component, the List component loads all child components. As a result,
// lazy loading does not take effect. In this scenario, you are advised to use the
// ListItemGroup component."

// FIX - one scroll surface, both levels virtualised:
List() {
  ForEach(this.billList, (group: ListModelData) => {
    ListItemGroup({ header: this.dayHeader(group) }) {
      if (group.isExpand) {
        ForEach(group.children, (tx: ChildrenObject) => {
          ListItem() { this.transactionRow(tx, group.type); }
        }, (tx: ChildrenObject) => tx.id);
      }
    }
  }, (group: ListModelData) => `${group.type}_${group.id}`);   // HW-02-0044
}
.divider(this.egDivider)
```

**`ListItemGroup` is the primitive for exactly this shape:** a sticky-capable
header plus a body, inside one `List`, with no second scroll container. The
expansion becomes a conditional `ForEach` inside the group rather than a nested
list, and the outer `List` keeps its ability to virtualise.

The key `${group.type}_${group.id}` is not paranoia: `spend_data.json` and
`income_data.json` both number their days 1, 3, 7, 10, 13, 17, so `group.id`
alone collides in the merged list.

**The filter and the three lists - `CollapseList.zip#entry/src/main/ets/pages/BillPage.ets:259`** (as shipped)

```typescript
@Builder
listBuilder() {
  if (this.selectedIndex === 0) {
    Column() {
      List() {
        ForEach(this.billList, (item: ListModelData) => {
          ListItem() {
            if (item.type == 0) {
              SpendDataItem({ item: item });
            } else {
              IncomeDataItem({ item: item });
            }
          };
        });
      }
      .divider(this.egDivider);
    }
    .width('91.12%').backgroundColor(Color.White);
  } else if (this.selectedIndex === 1) {
    // ... spendList, SpendDataItem ...
  } else {
    // ... incomeList, IncomeDataItem ...
  }
}
```

**Conditional rendering *inside* a `ListItem` is how the merged list stays
heterogeneous.** `item.type` picks the component; because both take the same
`@ObjectLink item`, the two branches are interchangeable at the call site.

The `divider` is a plain class instance held in `@State egDivider: DividerTmp`
rather than an object literal - ArkTS needs a named type for the `divider`
parameter, which is why `DividerTmp` exists at the top of the file.

**Safe-area handling - same file, line 93** (as shipped)

```typescript
Scroll() {
  Column() {
    Search({ placeholder: $r('app.string.search') });
    this.buttonBuilder();
    this.lumpSumBuilder();
    this.listBuilder();
  }
  .safeAreaPadding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
  .width('100%')
  .alignItems(HorizontalAlign.Start);
}
.align(Alignment.Top)
.scrollBar(BarState.Off)
.height('100%')
```

**`safeAreaPadding` rather than `padding`** puts the inset inside the scrollable
content, so the last row can scroll clear of the navigation bar instead of the
whole page being permanently shortened. That is the right choice for a scrolling
page and is worth copying.

`.align(Alignment.Top)` on the `Scroll` keeps short content pinned to the top
instead of centring it.

## Permissions & config

None. `CollapseList.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)` and enables `strictMode`.

The two data files live under `entry/src/main/resources/rawfile/` - 3.3 KB and
2.6 KB - and are reached by name through
`context.resourceManager.getRawFileContentSync('spend_data.json')`.

`EntryAbility` writes `topRectHeight` and `bottomRectHeight` into `AppStorage`
and subscribes to `avoidAreaChange` without releasing it (`HW-02-0047`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 42-44).
- The budgets are per-component compile-time constants (`const BUDGET = 500` in
  `SpendDataItem`, `10000` in `IncomeDataItem`). There is no per-user or
  per-month budget.
- The month is a display string; the model has no year and no month index, so
  the list can only be ordered within a single month (`HW-02-0050`).
- `id` is unique within each rawfile but **not** across the two, so the merged
  list needs a composite key.
- The rawfiles are read synchronously on the UI thread in `aboutToAppear`. At
  6 KB total that is fine; a real statement would need
  `getRawFileContent`'s promise form or a network fetch.
- Nothing recomputes `money` or the page totals after load - they are summed
  once.
- `SpendDataItem.ets` and `IncomeDataItem.ets` are 168 and 183 near-identical
  lines; the only real differences are the budget, the icon, the sign prefix and
  the detail row height (90 vs 70).
- The `Search` field is decorative - it has no handler.

## Pitfalls

- **`HW-02-0043` - the detail rows are a second `List` nested inside a `List`
  item, with no height, inside a `Scroll`.** The `List` reference names this
  case and its remedy: "you are advised to use the `ListItemGroup` component".
  As shipped nothing virtualises at either level.
- **`HW-02-0044` - no `ForEach` has a `keyGenerator`.** The default is
  `index + '__' + JSON.stringify(item)`, which is index-based (ruled out by the
  guidance) and embeds the mutable `isExpand`. The obvious fix is a trap: `id`
  alone collides between the two rawfiles - use `${type}_${id}`.
- **`HW-02-0045` - the expansion flag is stored twice.** `@State isExpand` is
  seeded from `item.isExpand` in `aboutToAppear` and is the only value `build()`
  reads, so the `@ObjectLink` binding delivers nothing and the copy cannot
  resynchronise if anything else writes the model.
- **`HW-02-0046` - the document's snippet omits `aboutToAppear`.** Without the
  re-seeding line the printed code loses the expansion on every tab switch, and
  the document never says the model field is what carries it.
- **`HW-02-0047` - `on('avoidAreaChange')` has no `off()`.**
- **`HW-02-0048` - each budget is typed twice,** once as `const BUDGET` and once
  as a string literal in the label beside the bar. Render the constant.
- **`HW-02-0049` - `import util from '@ohos.util'`** in a file that imports
  `common` from `@kit.AbilityKit`. The reference documents
  `import { util } from '@kit.ArkTS';`.
- **`HW-02-0050` - the merged list sorts on `day` alone.** Every bundled record
  is in 10月, so the bug is invisible; add a second month and the history
  interleaves.
- **Do not deep-copy when merging the two lists.** `concat` sharing the element
  objects is what makes expansion survive a tab switch.
- **Do not drop `ignoreBOM: true`.** A BOM in the rawfile becomes a leading
  `﻿` and `JSON.parse` throws.
- **Do not reach for `LazyForEach` while the nested `List` is still there.**
  With no main-axis size on either list, lazy loading cannot take effect at all.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`, and the nested-scrollable warning that names `ListItemGroup`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitemgroup.md` - `ListItemGroup`, `header`, `footer`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitemgroup
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - default key generation, "use a unique id property", "do not use the data item index as the key"
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed`/`@ObjectLink` for nested-object property changes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContentSync` (returns `Uint8Array`) and the async `getRawFileContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `util.TextDecoder`, `TextDecoderOptions.ignoreBOM`, and the documented `import { util } from '@kit.ArkTS'`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress`, `ProgressType.Linear`, `value`/`total` clamping
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-safe-area-padding.md` - `safeAreaPadding` versus `padding` in a scrolling page
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-safe-area-padding
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-06` - the same industry's other list card, with drag reordering
- `LIFE-01` - the industry shell this page would sit in
