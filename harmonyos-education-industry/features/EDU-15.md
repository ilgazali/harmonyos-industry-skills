---
id: EDU-15
title: Three-column cascading picker - school, department and graduation year in side-by-side Lists
industry: 04_education
doc: huawei_industry_tree/04_education/docs/15_cascading_selection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
sample: huawei_industry_tree/04_education/downloads/CascadingSelection.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit"]
apis: [List, ListItem, ListScroller, ForEach, EdgeEffect, Divider, "@Link", "@State", Navigation, NavPathStack, "NavPathStack.pop", "resourceManager.getRawFileContentSync", "util.TextDecoder", TextDecoderOptions, PromptAction]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0107, HW-04-0108, HW-04-0109, HW-04-0110, HW-04-0111, HW-04-0112, HW-04-0113, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **multi-level picker whose levels are visible at once** -
three columns side by side, where choosing in column 1 populates column 2 and
choosing in column 2 reveals column 3. School / department / graduation year is
the education case; province / city / district, category / subcategory / item,
and year / term / week are the same shape.

This is deliberately **not** `TextPicker` with a cascade array. Use that when the
picker is a modal wheel and the levels are opaque. Use this when the user should
see all three columns together, scroll each independently, and change an upper
level without reopening anything.

The two design decisions worth taking are: **the levels are gated by booleans,
not by array emptiness**, and **every column's click handler resets everything
below it**. Both are what stop a stale lower selection surviving an upper change.

## Feature checklist

- Three labelled columns of equal width, separated by vertical dividers.
- Column 1 always populated; column 2 appears once a school is chosen; column 3
  once a department is chosen.
- Tapping the currently selected row deselects it and collapses the levels below.
- Tapping a different row in an upper column clears every lower selection.
- The selected row in each column is highlighted.
- 重置 clears all three; 完成 validates that all three are set, then returns
  `school-department-year届` to the caller through `NavPathStack.pop`.

## Architecture

One `entry` module, two pages, two views.

```
entry/src/main/ets
├── common/Constants.ets        NOT_SELECTED (-1), START_YEAR_SEQ (1000)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/ItemModel.ets         ItemModel, Item, DepartmentItem
├── pages
│   ├── MainPage.ets            @Entry - shows the chosen string, opens the picker
│   └── SelectionPage.ets       owns all eight @State fields, reset / complete
├── utils/Logger.ets
└── views
    ├── InfoView.ets            one row: just the name
    └── SelectionView.ets       the three columns and all the cascade logic
```

The documented tree matches the zip.

**`SelectionPage` owns the state; `SelectionView` mutates it through eight
`@Link`s.** That split is the reason the reset and complete buttons - which live
on the page, outside the view - can act on the same selection:

| `@Link` | Meaning |
| --- | --- |
| `isSelectedSchool`, `isSelectedDepartment` | gates for columns 2 and 3 |
| `schoolSelectedId`, `departmentSelectedId`, `graduationYearSelectedId` | highlight + validation, `-1` when unset |
| `schoolName`, `departmentName`, `graduationYearName` | the strings the result is built from |

**Two parallel representations, deliberately.** The `...SelectedId` fields drive
the highlight and the validation; the `...Name` fields build the result string.
Keeping both means the result can be assembled without re-looking-up the items,
at the cost of having to keep them in step - which the reset button does not
quite do (it clears the ids and leaves the names, harmless only because
`complete` validates on the ids).

**The data model is where this sample goes wrong.** `data.json` is one flat
array of seventeen `Item`s: four schools with a `department` array and thirteen
graduation years without one, separated by `id >= START_YEAR_SEQ` (1000). Two of
the three columns therefore iterate the *same* array and hide the rows they do
not want with an `if` **inside** the `ListItem` (`HW-04-0108`).

## Implementation steps

1. **Model the levels as separate lists.** Load the JSON once and split it -
   `schools` and `years` - rather than distinguishing them by an id threshold at
   render time.
2. **Guard the data load.** `getRawFileContentSync` and `JSON.parse` both throw,
   and this runs in `aboutToAppear` (`HW-04-0110`).
3. **Hold selection state on the page, pass it down as `@Link`,** so buttons
   outside the picker view can reset and validate it.
4. **Use `-1` as the "nothing selected" sentinel** rather than `undefined`, so
   the highlight comparison `selectedId === item.id` is total.
5. **Give each column its own click rule:**
   - same row tapped → clear this level's id and flip its gate off;
   - different row tapped → set the gate, the id and the name;
   - **either way, reset every level below.**
6. **Gate columns 2 and 3 on the booleans**, not on the array being non-empty -
   an emptied array and a deselected level are different states.
7. **Filter the data per column** and put the `if` on the data, not inside the
   `ListItem` (`HW-04-0108`).
8. **Key every `ForEach` by `item.id`** (`HW-04-0109`).
9. **Do not share a `ListScroller`** across the three lists (`HW-04-0107`).
10. **Validate in order and report which level is missing,** then return the
    result with `pathInfos.pop(result)`.

## Verified snippets

All snippets are from `CascadingSelection.zip`. Corrected forms are marked.

**A column and its cascade rule — `entry/src/main/ets/views/SelectionView.ets`** (corrected, see `HW-04-0107`, `HW-04-0108`, `HW-04-0109`)

```typescript
// FIX: the sample declares ONE ListScroller and binds it to all three Lists.
// The reference: "A single Scroller instance cannot control multiple container
// components simultaneously." None of the three is ever used - drop them all.

// School column
List() {
  // FIX: the sample iterates the whole 17-entry array and hides years with an
  //      `if (item.id < Constants.START_YEAR_SEQ)` INSIDE the ListItem, leaving
  //      live click handlers on rows that render nothing.
  ForEach(this.schoolList, (item: Item) => {
    ListItem() {
      InfoView({ item: item })
    }
    .onClick(() => {
      if (this.schoolSelectedId === item.id) {
        this.schoolSelectedId = Constants.NOT_SELECTED;   // tap the selected row -> deselect
        this.isSelectedSchool = false;
      } else {
        this.isSelectedSchool = true;
        this.departmentList = item.department;            // populate the next column
        this.schoolSelectedId = item.id;
        this.schoolName = item.name;
      }
      // both branches: everything below this level is now invalid
      this.isSelectedDepartment = false;
      this.departmentSelectedId = Constants.NOT_SELECTED;
      this.graduationYearSelectedId = Constants.NOT_SELECTED;
    })
    .backgroundColor(this.schoolSelectedId === item.id ? '#F1F3F5' : Color.White)
    .borderRadius(8)
  }, (item: Item) => item.id.toString())                  // FIX: sample passes no key generator
}
.edgeEffect(EdgeEffect.None)
.height(257)
```

**The reset-below block sits after the if/else, not inside it.** That is the
whole cascade contract: selecting a school and deselecting a school both
invalidate the department and the year, so the reset cannot live in either
branch. Put it in one and a deselect leaves a department highlighted under no
school.

**`this.departmentList = item.department` is the only data flow between
columns.** The second column renders whatever that `@State` array holds; it is
gated separately by `isSelectedSchool`, so a deselect hides it without having to
clear it.

**The default key here is expensive.** `Item` embeds its `department` array, so
without a key generator ArkUI serialises each school *and all of its departments*
into the key on every pass. The ForEach guide names this shape specifically:
*"When item is a complex object, serializing it to a JSON string results in a
long string that consumes more memory"*, measuring ~70 MB saved by supplying one.
The data already has `id`.

**The gate for the lower columns — same file** (as shipped)

```typescript
// Department column
ForEach(this.departmentList, (item: Item) => {
  ListItem() {
    if (item.id < Constants.START_YEAR_SEQ && this.isSelectedSchool === true) {
      InfoView({ item: item })
    }
  }
  // ...
})

// Graduation-year column - iterates the SAME array as the school column
ForEach(this.schoolList, (item: Item) => {
  ListItem() {
    if (item.id >= Constants.START_YEAR_SEQ && this.isSelectedDepartment === true) {
      InfoView({ item: item })
    }
  }
  // ...
})
```

**The boolean gate is right; its placement is not.** Gating on
`isSelectedSchool` rather than on `departmentList.length` is the correct call -
it distinguishes "no school chosen" from "a school with no departments" - but
because the condition is inside the `ListItem`, the rows still exist with their
handlers and backgrounds attached. Move the condition up to wrap the whole
`List`, and filter the array for the id test.

**Loading the data — same file** (corrected, see `HW-04-0110`, `HW-04-0111`)

```typescript
getJson(context: common.UIAbilityContext) {
  try {                                                   // FIX: sample has no try/catch
    const dataJson = context.resourceManager.getRawFileContentSync('data.json');
    const options: util.TextDecoderOptions = { ignoreBOM: true };
    const decoder = util.TextDecoder.create('utf-8', options);
    const result = decoder.decodeToString(dataJson, { stream: false });
    const model: ItemModel = JSON.parse(result) as ItemModel;
    // FIX: split here rather than testing item.id at every render
    this.schoolList = model.data.filter(i => i.id < Constants.START_YEAR_SEQ);
    this.yearList = model.data.filter(i => i.id >= Constants.START_YEAR_SEQ);
  } catch (err) {
    Logger.error(`Failed to load data.json: ${err}`);
    this.schoolList = [];
    this.yearList = [];
  }
}
```

**`ignoreBOM: true` matters for a hand-authored JSON file.** A UTF-8 BOM at the
front of `data.json` becomes a leading `﻿` in the decoded string, and
`JSON.parse` rejects it. Setting the option strips it in the decoder rather than
by string surgery afterwards.

**`Item.department` is declared non-optional and is absent from thirteen of the
seventeen entries** (`HW-04-0111`); the `as ItemModel` cast is what hides the
mismatch.

**Validation and the result — `entry/src/main/ets/pages/SelectionPage.ets`** (as shipped)

```typescript
Button($r('app.string.complete'))
  .onClick(() => {
    if (this.schoolSelectedId === Constants.NOT_SELECTED) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.select_school') });
    } else if (this.departmentSelectedId === Constants.NOT_SELECTED) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.select_department') });
    } else if (this.graduationYearSelectedId === Constants.NOT_SELECTED) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.select_graduation_year') });
    } else {
      this.info = `${this.schoolName}-${this.departmentName}-${this.graduationYearName}届`;
      this.pathInfos.pop(this.info);          // hand the result back to MainPage
    }
  })
```

**`NavPathStack.pop(result)` is the return channel.** The picker does not write
into shared state or call back; it pops with a value, and the page that pushed it
receives it. That keeps `SelectionPage` reusable from anywhere.

**Validating in cascade order** means the toast always names the *first* missing
level, which is the one the user has to fix - checking them in any other order
would tell them to pick a year before a school.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block; the
data is a bundled rawfile at `entry/src/main/resources/rawfile/data.json`:

```json
{ "data": [
  { "id": 0, "name": "牧歌大学", "department": [ { "id": 0, "name": "计算机系" }, ... ] },
  ...
  { "id": 1013, "name": "2025" }
] }
```

Seventeen entries in one array: schools below `START_YEAR_SEQ` (1000), graduation
years at or above it.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Exactly three levels, hard-coded.** The columns are three literal `Column`
  blocks with the cascade logic written out per column; a fourth level means
  copying the block and adding another gate.
- Each column is a fixed 110 vp wide and 257 vp tall inside a 306 vp row - the
  picker does not adapt to the screen.
- **Department ids restart at 0 for every school**, so `departmentSelectedId`
  is only meaningful together with `schoolSelectedId`. The code is safe because
  changing school resets it, but the id alone does not identify a department.
- The year list is global, not per department.
- Nothing is persisted; the result is returned once through `pop` and the picker
  reopens empty.
- `InfoView` takes its item through `@State`, which is initialised externally -
  `@Prop` is the decorator for a value passed down and never written back.

## Pitfalls

- **`HW-04-0107` — one `ListScroller` is bound to all three `List`s,** which the
  reference forbids: "A single Scroller instance cannot control multiple
  container components simultaneously." None of the three is ever used.
- **`HW-04-0108` — schools and years live in one array behind an id sentinel,**
  so two columns iterate all seventeen rows and hide the unwanted ones with an
  `if` inside the `ListItem` - leaving click handlers and backgrounds on rows
  that render nothing.
- **`HW-04-0109` — no `ForEach` key generator,** so each school's default key is
  a JSON serialisation of the school plus its whole department subtree.
- **`HW-04-0110` — the data load has no error handling** and runs in
  `aboutToAppear`; a missing or malformed `data.json` throws out of component
  construction.
- **`HW-04-0111` — `Item.department` is non-optional but absent from most
  entries,** and the `as ItemModel` cast suppresses the mismatch.
- **`HW-04-0112` — the document references a `DepartmentView` that does not
  exist** (the component is `InfoView`) and documents only two of the three
  cascade levels.
- **`HW-04-0113` — the department handler clears the year selection twice** on
  the same path.
- **Do not gate a lower level on the upper array being empty.** "No school
  chosen" and "a school with no departments" must render differently, which is
  why the booleans exist.
- **Do not forget the reset-below block on the deselect branch.** It is the one
  line that keeps the three columns consistent.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListScroller`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroller` and the one-container-per-instance rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` - `ListItem` and where a conditional belongs
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - the default key generator and its memory cost on complex objects
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack.pop(result)` as a return channel
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContentSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `TextDecoder` and `ignoreBOM`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `EDU-04` - the other multi-`List` layout in this industry, and its shared-scroller pattern done deliberately
- `EDU-14`, `EDU-06` - the same missing-key-generator defect
