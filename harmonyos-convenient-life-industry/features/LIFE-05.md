---
id: LIFE-05
title: Three-column cascading category picker - parallel Lists with per-level select-all and a chip dialog
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
sample: huawei_industry_tree/02_convenient_life/downloads/CascadingMenuSelection.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItem, ListScroller, Grid, GridItem, "@Observed", "@ObjectLink", "@Link", "@Watch", "@State", "@Prop", "@StorageProp", "@CustomDialog", CustomDialogController, onWillDismiss, DismissReason, expandSafeArea, SafeAreaType, SafeAreaEdge, enableScrollInteraction, alignListItem, layoutWeight, "window.getWindowAvoidArea", "AppStorage.setOrCreate", "util.format", "resourceManager.getStringSync"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0027, HW-02-0028, HW-02-0029, HW-02-0030, HW-02-0031, HW-02-0032, HW-02-0033, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when a category tree has to be **browsed and multi-selected at
the same time**, and the user must see the parent-child descent alongside the
sibling alternatives at every level.

The shape is three `List`s side by side in one `Row`: tapping an item in column
N does not navigate anywhere, it just refills column N+1. Selection happens only
at the deepest level, but each of the upper two columns carries a 全选 (select
all) entry that cascades down. The running selection is shown as removable chips
in a bottom sheet.

This is the produce-category picker of a grocery app, but the construction fits
any fixed-depth taxonomy: region/province/city, department/team/person,
brand/series/model.

Choose it over a collapsible tree when the branching factor is wide (this data
has 4 x up-to-6 x up-to-6 = 54 leaves) and over `Navigation` when the user needs
to compare siblings without losing their place.

## Feature checklist

- Three parallel `List`s in one `Row`, weighted 2:2:2, with a vertical divider
  before the third.
- Column 1 is a static heading plus the four top categories; it does not scroll.
- Tapping a first-level item fills column 2 and resets the second-level
  selection; tapping a second-level item fills column 3.
- Columns 2 and 3 each lead with a 全选 (select all) row that toggles the whole
  subtree below it.
- Only third-level items carry a check mark; tapping one toggles it.
- Selecting the last unselected sibling promotes the parent's select-all to
  checked; deselecting any child demotes it, and demotes its grandparent too.
- A bottom bar shows a live count and opens a sheet of chips, one per selected
  leaf, each with a delete badge.
- Deleting a chip clears the corresponding leaf back in the menu and refreshes
  both select-all indicators.

## Architecture

One `entry` module. The data is loaded from a rawfile JSON at module scope and
rebuilt into decorated classes.

```
entry/src/main/ets
├── common
│   ├── Constant.ets                 SELECTED 1 / NOT_SELECTED 0 / NO_SELECTED -1, layout weights
│   └── DataManager.ets              JSON -> PrimaryMenuItem / @Observed SecondMenuItem / @Observed ThirdMenuItem
├── components
│   ├── WholeMenu.ets                THE CARD: three Lists, all the selection logic
│   ├── PrimaryMenuItem.ets          first-level row  (@State  - see HW-02-0029)
│   ├── SecondLevelMenuItem.ets      second-level row (@ObjectLink)
│   ├── ThirdLevelMenuItem.ets       third-level row  (@ObjectLink, draws the check mark)
│   ├── BottomBar.ets                count + sheet trigger + 完成 button
│   └── ItemsSelectedDialog.ets      the chip sheet (@Link selectedItems / @Link deleteItem)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/SelectionPage.ets          @Entry - owns selectedItems, totalCount, deleteItem
resources/rawfile/config/menu_item_list.json    4 x N x M category tree
```

The documented tree matches the zip exactly.

**Three pieces of state live on the page and are shared by `@Link`:**

| page state | who writes it | who reads it |
| --- | --- | --- |
| `selectedItems: string[]` | `WholeMenu` on every toggle; the dialog on chip delete | the dialog's `Grid` |
| `totalCount: number` | `WholeMenu` only | `BottomBar` (`@Prop`) |
| `deleteItem: string` | the dialog on chip delete | `WholeMenu` via `@Watch('onDelete')` |

`deleteItem` is the interesting one: it is a **one-shot command channel**. The
dialog cannot reach into the menu tree, so it writes a name into `deleteItem`;
`WholeMenu`'s `@Watch` handler finds that leaf, clears it, fixes the two
select-all flags, and then blanks `deleteItem` so the same name can be sent
again. That blanking re-enters the handler (`HW-02-0031`).

**Selection state is stored on the data, not in a separate set.** Each
`ThirdMenuItem` carries `isSelected`, each `SecondMenuItem` and
`PrimaryMenuItem` carries `isAll`. `@Observed` on the two lower classes plus
`@ObjectLink` in their row components is what makes a check mark repaint without
the parent rebuilding the list. The first level is left out of that pattern
(`HW-02-0029`).

`secondAll` and `thirdAll` on `WholeMenu` are *view* state for the two select-all
rows - they mirror the selected parent's `isAll` and are recomputed whenever the
user changes column.

## Implementation steps

1. **Define the level classes with `@Observed`,** all three of them, and rebuild
   the raw JSON into them in a `DataManager` singleton so every item starts at
   `NOT_SELECTED` regardless of what the file says.
2. **Lay out one `Row` with three `List`s** at equal `layoutWeight`. Give the
   first `enableScrollInteraction(false)` if the top level is short enough to
   fit, and put a `Divider().vertical(true)` between the second and third.
3. **Gate columns 2 and 3 on a selection sentinel.** `NO_SELECTED = -1` and the
   render guard `if (this.primaryMenuSelectedId >= Constants.NOT_SELECTED)`
   means the column stays empty until a parent is chosen.
4. **Refill the next column in `onClick`, and reset the level below it.**
   Selecting a first-level item must also set `secondMenuSelectedId` back to
   `NO_SELECTED`, or column 3 keeps showing the previous branch.
5. **Recompute the select-all indicator on every column change** from the
   parent's `isAll`, not from a scan of the children - that is what `isAll` is
   for.
6. **Look children up by identity, not by id-as-subscript** (`HW-02-0027`). The
   sample writes `secondLevelMenuItemList[secondMenuSelectedId]`, which works
   only because the sample data numbers everything from 0 in array order.
7. **Write the leaf toggle as a symmetric pair.** On select: mark, push the
   name, bump the count, and promote the parent if `every()` child is now
   selected. On deselect: unmark, remove the name, drop the count, and demote
   the parent and grandparent.
8. **Guard both directions of the select-all loops** (`HW-02-0028`). The sample
   guards only the select direction, so the counter can drift below the list
   length.
9. **Give the dialog `@Link`, not `@Prop`.** The `CustomDialogController`
   reference is explicit: "To listen for data changes in the builder, use the
   @Link or @Consume decorator; other decorators, such as @Prop and
   @ObjectLink, do not apply." This sample gets it right; `LIFE-04` does not.
10. **Return deletions through a watched string** and make the handler ignore
    its own reset value (`HW-02-0031`).

## Verified snippets

All snippets are from `CascadingMenuSelection.zip`. Corrected forms are marked.

**Rebuilding raw JSON into decorated classes - `CascadingMenuSelection.zip#entry/src/main/ets/common/DataManager.ets`** (as shipped)

```typescript
import primaryMenuData from '../../resources/rawfile/config/menu_item_list.json';

export interface PrimaryMenuItem {          // <- not @Observed: HW-02-0029
  id: number;
  name: string;
  isAll: number;
  secondMenu: SecondMenuItem[];
}

@Observed
export class SecondMenuItem {
  id: number; name: string; isAll: number; thirdMenu: ThirdMenuItem[];
  constructor(id: number, name: string, isAll: number, thirdMenu: ThirdMenuItem[]) { /* ... */ }
}

@Observed
export class ThirdMenuItem {
  id: number; name: string; isSelected: number;
  constructor(id: number, name: string, isSelected: number) { /* ... */ }
}

class DataManager {
  private primaryMenuData: PrimaryMenu = {
    primaryMenuItemList: primaryMenuData.primaryMenuItemList.map(item => ({
      id: item.id,
      name: item.name,
      isAll: Constants.NOT_SELECTED,                       // never trust the file
      secondMenu: item.secondMenu.map(sItem => (new SecondMenuItem(
        sItem.id, sItem.name, Constants.NOT_SELECTED,
        sItem.thirdMenu.map(tItem => (new ThirdMenuItem(
          tItem.id, tItem.name, Constants.NOT_SELECTED)))
      )))
    } as PrimaryMenuItem))
  };

  getPrimaryMenuData(): PrimaryMenuItem[] {
    return this.primaryMenuData.primaryMenuItemList;
  }
}

export default new DataManager();
```

**A rawfile JSON can be `import`ed directly** - no `resourceManager`, no async
read. But the parsed value is plain objects, and plain objects cannot carry
`@Observed`. That is why every level is reconstructed through a constructor
rather than used as-is: **`@Observed` is a class decorator, so the data has to
be classes before the UI can watch it.**

Rebuilding is also where the selection state is forced to `NOT_SELECTED`, so a
stale flag in the data file cannot leak into the UI.

`export default new DataManager()` makes the tree a module-scope singleton -
every mount of `WholeMenu` shares one tree, so the selection survives a
remount but is never reset.

**The three columns - `CascadingMenuSelection.zip#entry/src/main/ets/components/WholeMenu.ets:44`** (as shipped)

```typescript
Row() {
  List({ scroller: this.primaryListScroll }) {
    ListItem() { /* 蔬菜 heading */ }
      .height($r('app.integer.integer_50'));

    ForEach(this.primaryMenuItemList, (item: PrimaryMenuItem) => {
      ListItem() {
        PrimaryMenu({ primaryMenu: item, primaryMenuSelectedId: this.primaryMenuSelectedId })
          .backgroundColor(this.primaryMenuSelectedId === item.id
            ? $r('app.color.page_background') : Color.White);
      }
      .onClick(() => {
        this.primaryMenuSelectedId = item.id;
        this.secondMenuSelectedId = Constants.NO_SELECTED;   // reset the level below
        this.secondLevelMenuItemList = item.secondMenu;      // refill the next column
        this.secondAll = item.isAll === Constants.SELECTED
          ? Constants.SELECTED : Constants.NOT_SELECTED;
      });
    });
    ListItem() { Column() { Blank().height($r('app.integer.integer_84')); }; };
  }
  .enableScrollInteraction(false)
  .layoutWeight(Constants.LAYOUT_WEIGHT_TWO)
  .border({ style: { right: BorderStyle.Solid }, width: { right: $r('app.integer.integer_1') } });
  // ... two more Lists ...
}
.alignItems(VerticalAlign.Top)
```

**Four assignments per tap, and all four are needed.** Setting the id drives the
highlight; resetting `secondMenuSelectedId` to `NO_SELECTED` collapses column 3
so it cannot show the previous branch; assigning `secondLevelMenuItemList`
refills column 2; and reading `item.isAll` restores the select-all indicator for
the branch you just entered. Drop the second one and column 3 shows stale data
from the sibling you left.

The document's snippet prints only the third of these four assignments
(`HW-02-0032`).

`.alignItems(VerticalAlign.Top)` on the `Row` is what stops the three lists from
being vertically centred against each other. The trailing `Blank()` `ListItem`
is bottom padding for the bar that overlays the list.

**The leaf toggle - same file, line 166** (corrected, see `HW-02-0027`)

```typescript
.onClick(() => {
  // FIX: the sample writes this.secondLevelMenuItemList[this.secondMenuSelectedId]
  const parent = this.secondLevelMenuItemList
    .find((s: SecondMenuItem) => s.id === this.secondMenuSelectedId);
  const grandparent = this.primaryMenuItemList
    .find((p: PrimaryMenuItem) => p.id === this.primaryMenuSelectedId);
  if (!parent || !grandparent) {
    return;
  }

  if (item.isSelected === Constants.NOT_SELECTED) {
    item.isSelected = Constants.SELECTED;
    this.selectedItems.push(item.name);
    this.totalCount += Constants.COUNT_INCREASE;
    const allSelected = parent.thirdMenu
      .every((t: ThirdMenuItem) => t.isSelected === Constants.SELECTED);
    if (allSelected) {                        // promote the parent
      parent.isAll = Constants.SELECTED;
      this.thirdAll = Constants.SELECTED;
    }
  } else {
    item.isSelected = Constants.NOT_SELECTED;
    this.removeByName(item.name);
    this.totalCount += Constants.COUNT_DECREASE;
    if (parent.isAll === Constants.SELECTED) {        // demote the parent
      parent.isAll = Constants.NOT_SELECTED;
      this.thirdAll = Constants.NOT_SELECTED;
    }
    if (grandparent.isAll === Constants.SELECTED) {   // and the grandparent
      grandparent.isAll = Constants.NOT_SELECTED;
      this.secondAll = Constants.NOT_SELECTED;
    }
  }
});
```

**Promotion is asymmetric with demotion, and that is correct.** Promoting needs
an `every()` scan - you only become "all selected" when the last sibling flips.
Demoting needs no scan: one child leaving is enough to break the invariant at
both levels, so the guarded assignment is all it takes.

The two `if` guards on demotion are not redundant - without them, `thirdAll` and
`secondAll` would be cleared on every deselect even when they were already
clear, which is harmless, but the guards also make the intent (demote only if
currently promoted) explicit.

**Select-all over a subtree - same file, line 257** (corrected, see `HW-02-0028`)

```typescript
thirdMenuAll(): void {
  const parent = this.secondLevelMenuItemList
    .find((s: SecondMenuItem) => s.id === this.secondMenuSelectedId);
  const grandparent = this.primaryMenuItemList
    .find((p: PrimaryMenuItem) => p.id === this.primaryMenuSelectedId);
  if (!parent || !grandparent) {
    return;
  }

  if (this.thirdAll === Constants.NOT_SELECTED) {
    this.thirdAll = Constants.SELECTED;
    parent.isAll = Constants.SELECTED;
    this.thirdLevelMenuItemList.forEach((thirdItem: ThirdMenuItem) => {
      if (thirdItem.isSelected === Constants.NOT_SELECTED) {     // guarded
        thirdItem.isSelected = Constants.SELECTED;
        this.selectedItems.push(thirdItem.name);
        this.totalCount += Constants.COUNT_INCREASE;
      }
    });
  } else {
    this.thirdAll = Constants.NOT_SELECTED;
    parent.isAll = Constants.NOT_SELECTED;
    this.secondAll = Constants.NOT_SELECTED;
    grandparent.isAll = Constants.NOT_SELECTED;
    this.thirdLevelMenuItemList.forEach((thirdItem: ThirdMenuItem) => {
      if (thirdItem.isSelected === Constants.SELECTED) {          // FIX: sample has no guard
        thirdItem.isSelected = Constants.NOT_SELECTED;
        this.removeByName(thirdItem.name);
        this.totalCount += Constants.COUNT_DECREASE;
      }
    });
  }
}
```

**The select branch is idempotent because of its guard; the deselect branch in
the sample is not.** Selecting a subtree where some children are already
selected must not push duplicate names or double-count - hence the
`if (isSelected === NOT_SELECTED)`. The mirror guard belongs on the other side
for the same reason, and the sample omits it (`HW-02-0028`).

Note that deselecting at level 3 demotes **two** levels (parent and
grandparent) while selecting promotes only one. That is deliberate: one full
child branch does not make the whole first-level category complete.

**The delete channel - same file, line 33 and line 216** (corrected, see `HW-02-0032`)

```typescript
@Link @Watch('onDelete') deleteItem: string;

onDelete(): void {
  if (this.deleteItem === Constants.NULL_STRING) {
    return;                                     // FIX: ignore our own reset
  }
  this.primaryMenuItemList.forEach((primaryItem: PrimaryMenuItem) => {
    primaryItem.secondMenu.forEach((secondItem: SecondMenuItem) => {
      secondItem.thirdMenu.forEach((thirdItem: ThirdMenuItem) => {
        if (thirdItem.name === this.deleteItem) {
          thirdItem.isSelected = Constants.NOT_SELECTED;
          this.totalCount += Constants.COUNT_DECREASE;
          secondItem.isAll = Constants.NOT_SELECTED;
          primaryItem.isAll = Constants.NOT_SELECTED;
          if (primaryItem.id === this.primaryMenuSelectedId) {
            this.secondAll = Constants.NOT_SELECTED;
          }
          if (secondItem.id === this.secondMenuSelectedId) {
            this.thirdAll = Constants.NOT_SELECTED;
          }
        }
      });
    });
  });
  this.deleteItem = Constants.NULL_STRING;      // rearm for the same name
}
```

**Blanking the trigger is what makes the channel reusable.** `@Watch` fires on
change, so deleting "白萝卜" twice in a row would be a no-op the second time
without the reset. The cost is that the reset is itself a change, so the handler
re-enters - which is why the early return is needed rather than optional.

The two `id ===` tests are the "is this branch on screen right now" check: the
data flags are always fixed, but `secondAll`/`thirdAll` are view state and only
need refreshing when the deleted leaf belongs to the branch currently displayed.

**The chip sheet - `CascadingMenuSelection.zip#entry/src/main/ets/components/ItemsSelectedDialog.ets`** (as shipped)

```typescript
@Component
@CustomDialog
export struct ItemsSelectedDialog {
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @Link selectedItems: string[];
  @Link deleteItem: string;
  uiContext: UIContext = this.getUIContext();
  controller?: CustomDialogController;

  @Builder
  SmallTag(item: string) {
    Stack({ alignContent: Alignment.TopEnd }) {
      Flex({ justifyContent: FlexAlign.Center }) { Text(item); }
      Image($r('app.media.ic_delete'))
        .margin({ top: $r('app.integer.integer_negative_5'), right: $r('app.integer.integer_negative_5') })
        .onClick(() => {
          const result = this.selectedItems.filter((name: string) => name === item);
          if (result.length > 0) {
            this.selectedItems.splice(this.selectedItems.indexOf(result[0]), 1);
            this.deleteItem = item;              // hand the removal to WholeMenu
          }
        });
    };
  }
}
```

**`@Link` is mandatory here, not stylistic.** The reference says of a dialog
builder's parameters: "To listen for data changes in the builder, use the @Link
or @Consume decorator; other decorators, such as @Prop and @ObjectLink, do not
apply." The chip grid has to track pushes made by `WholeMenu` while the sheet is
open, so `@Prop` would freeze it.

Stacking `@Component` on top of `@CustomDialog` is the documented form - the
reference's own examples do the same.

The delete badge is positioned with **negative margins inside a
`Stack({ alignContent: Alignment.TopEnd })`**, which is how it hangs outside the
chip's corner without a `position()` and without being clipped.

**Page wiring - `CascadingMenuSelection.zip#entry/src/main/ets/pages/SelectionPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct SelectionPage {
  @State selectedItems: string[] = [];
  @State totalCount: number = 0;
  @State deleteItem: string = '';
  dialogController: CustomDialogController = new CustomDialogController({
    builder: ItemsSelectedDialog({ selectedItems: this.selectedItems, deleteItem: this.deleteItem }),
    autoCancel: true,
    onWillDismiss: (dismissDialogAction: DismissDialogAction) => { /* PRESS_BACK / TOUCH_OUTSIDE */ },
    alignment: DialogAlignment.Bottom,
    height: $r('app.string.thirty_five'),
    // ...
  });

  build() {
    Column() {
      Row() { /* back arrow + Search */ }
      Flex({ direction: FlexDirection.Column }) {
        WholeMenu({ selectedItems: this.selectedItems, totalCount: this.totalCount, deleteItem: this.deleteItem });
      }
      BottomBar({ totalCount: this.totalCount, dialogController: this.dialogController });
    }
    .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP]);
  }
}
```

**`expandSafeArea` is the declarative alternative to the `AppStorage` inset
dance** used elsewhere in this industry - the page here draws under the status
bar with `[SafeAreaEdge.TOP]` and `BottomBar` does the same with
`[SafeAreaEdge.BOTTOM]`, with no window API and no stored height. The dialog is
the one place that still reads a stored inset (`HW-02-0033`), which is why this
sample uses two different strategies on one screen.

The controller is passed **down** into `BottomBar` as a plain field, so the bar
can open a dialog the page owns. That is the documented arrangement: a
`CustomDialogController` must be a member of the `@Component` struct that
declares it.

## Permissions & config

None. `CascadingMenuSelection.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

`main_pages.json` is `{ "src": ["pages/SelectionPage"] }`.

The category tree ships as a rawfile, imported directly:
`entry/src/main/resources/rawfile/config/menu_item_list.json` - 4 primary
categories, 14 second-level, 54 leaves.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 52-54).
- The depth is fixed at three. Every method in `WholeMenu` names the levels
  explicitly; a fourth level means new state fields, a new `List` and a new
  select-all handler.
- The first column has `enableScrollInteraction(false)`, so it must fit on
  screen - four items here. More top-level categories need that removed.
- The sample data numbers every level from 0 in array order, and the code
  depends on it (`HW-02-0027`).
- Leaf names must be unique across the whole tree: chips are keyed by name, and
  `onDelete` clears **every** leaf whose name matches. The bundled 54 names are
  unique; a duplicate would be cleared in two places and double-decrement the
  count.
- The 完成 (finish) button in `BottomBar` has no `onClick` - the sample ends at
  selection.
- The `Search` box on the page is decorative; it has no handler.

## Pitfalls

- **`HW-02-0027` - ids are used as array subscripts** at eleven call sites
  (`secondLevelMenuItemList[secondMenuSelectedId]`,
  `primaryMenuItemList[primaryMenuSelectedId]`). It works only because the
  bundled JSON numbers everything from 0 in order. Use `find()` on `id`.
- **`HW-02-0028` - the select-all deselect branches decrement the count
  unguarded** while the select branches guard their increments. `removeByName`
  is safe, so the counter can fall below `selectedItems.length` and never
  resynchronise. Guard both directions, or derive the count from
  `selectedItems.length`.
- **`HW-02-0029` - the first level is not `@Observed`.** `PrimaryMenuItem` is a
  plain `interface` and `PrimaryMenu` binds it with `@State`, unlike the two
  levels below which use `@Observed` + `@ObjectLink`. `isAll` is written four
  times and can never be observed; add a check mark to that row and it will not
  update.
- **`HW-02-0030` - a divider width is fetched with
  `this.context?.resourceManager.getStringSync($r('app.string.integer_1').id)`
  inside `build()`,** from a context read in a field initializer. The same file
  sets a border width with `$r('app.integer.integer_1')` forty-six lines
  earlier. Use the resource reference.
- **`HW-02-0032` - the document's snippet is not followable.** Both `ListItem`
  bodies are empty, the comment points at a 全选 button on the *first* list
  where the shipped code has a plain heading (the select-all rows are on lists
  2 and 3), and the `onClick` shows one of its four statements.
- **`HW-02-0031` - `onDelete` re-enters itself.** It ends by blanking the
  watched `deleteItem`, which fires `@Watch` again and re-walks all 54 leaves.
  Early-return on the empty string.
- **`HW-02-0033` - the navigation-bar inset is read once and never
  refreshed,** so the bottom-anchored dialog's padding is stale after a
  rotation - while the rest of the same screen uses `expandSafeArea` and has no
  such problem.
- **Do not forget to reset the level below when a column changes.** Selecting a
  first-level item must set `secondMenuSelectedId = NO_SELECTED`; otherwise
  column 3 keeps rendering the branch from the category you just left.
- **Do not put `@Observed` on an interface.** It is a class decorator; the raw
  JSON must be rebuilt through constructors before the UI can watch it.
- **Do not pass dialog data with `@Prop`.** The reference permits only `@Link`
  or `@Consume` for values a dialog builder must track.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed` is a class decorator; `@ObjectLink` for nested-object property changes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItem`, `ListScroller`, `enableScrollInteraction`, `alignListItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `CustomDialogController`, the `@Link`/`@Consume`-only rule for builder data, `@CustomDialog` + `@Component` stacking, `onWillDismiss`, `DismissReason`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` - `expandSafeArea`, `SafeAreaType`, `SafeAreaEdge`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - the chip `Grid` and `columnsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `LIFE-04` - the same `CustomDialogController` pattern, done with `@Prop` instead (`HW-02-0026`)
- `LIFE-01` - the industry shell this page would sit in
