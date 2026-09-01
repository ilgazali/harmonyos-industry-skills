---
id: OFFICE-26
title: Multi-level organisation menu - a horizontal breadcrumb List driving a vertical level List
industry: 05_office
doc: huawei_industry_tree/05_office/docs/26_organization_structure.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/organization_structure-0000002388806441
sample: huawei_industry_tree/05_office/downloads/OrganizationalStructure.zip
kits: ["@kit.ArkUI"]
apis: [List, ListItem, "List.listDirection", Axis, ForEach, Search, Divider, Blank, "expandSafeArea", SafeAreaType, Visibility, "@State", "@Entry", "@Component"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0140, HW-05-0141, HW-05-0142, HW-05-0143, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a directory tree has to be browsed **one level at a time**
rather than expanded in place - the department drill-down of a corporate address
book, a category picker, any deep hierarchy where showing the whole tree at once
would be unusable.

The pattern is two `List`s bound to two pieces of state:

| List | Direction | Bound to | Role |
| --- | --- | --- | --- |
| breadcrumb | `Axis.Horizontal` | `selectedStructure: Structure[]` | the path from the root to the current level |
| level | vertical (default) | `menuItemList: Structure[]` | the children of the current level |

Tapping a row **descends** (push onto the trail, re-point the level list at that
row's children); tapping a breadcrumb **ascends** (pop the trail back to that
index, re-point the level list at that node's children). No navigation stack, no
recursion - contrast OFFICE-17, which renders the same kind of tree recursively
and all at once.

No permission is involved.

## Feature checklist

The application must be able to:

- Model the organisation as a recursive node type with children.
- Show the current level's children as a vertical list.
- Show the path from the root to the current level as a horizontal breadcrumb.
- Descend into a row only when that row has children, and indicate which rows do.
- Style leaf rows differently from branch rows.
- Return to any ancestor by tapping its breadcrumb entry, truncating the trail to
  that point.
- Colour the current breadcrumb entry differently from the tappable ancestors.
- Lay the page out under the system safe area.

## Architecture

Single `entry` HAP, four source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/StructurePage`; no window listeners, no permissions |
| `pages/StructurePage.ets` | **the whole feature**: both lists and the two navigation handlers |
| `model/MenuModel.ets` | the recursive `Structure` type |
| `model/MenuData.ets` | the `RESEARCH_LIST` seed tree |

Note the directory is `pages/` - the document's tree says `page/`
(HW-05-0143).

The model is recursive and carries its own key:

```ts
export class Structure {
  id: number;
  name: string;
  avatar: Resource = $r('app.media.structure');
  sonStructure: Structure[];
}
```

`sonStructure.length > 0` is the single predicate the whole UI keys off: it
decides whether the row is tappable, whether the chevron shows, and the row's
height, padding, avatar size and label margin. A leaf is rendered taller
(`80` vs `65`) and slightly inset, so people read as people and departments read
as departments without any extra type field.

State and transitions:

```
aboutToAppear
  selectedStructure = [root]                 // breadcrumb starts at the root
  menuItemList     = root.sonStructure       // level list starts at its children

row onClick (branch only)
  menuItemList = item.sonStructure
  selectedStructure.push(item)

breadcrumb onClick at index i
  menuItemList = item.sonStructure
  if (i !== selectedStructure.length - 1)    // not already the current level
    do { selectedStructure.pop() }
    while (selectedStructure.length - 1 > i)
```

The `do…while` pops until the clicked index becomes the last element. The guard
around it is load-bearing: without it, tapping the **current** breadcrumb would
still pop once (HW-05-0140).

## Implementation steps

1. **Declare no permission.** The tree is local seed data; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Define the recursive node type** with an `id`, a display name, an avatar and
   a `sonStructure` array.
3. **Seed both state arrays in `aboutToAppear`** - the trail with the root, the
   level list with the root's children.
4. **Render the breadcrumb horizontally.** `List().listDirection(Axis.Horizontal)`
   with a fixed height and `scrollBar(BarState.Off)`, one `ListItem` per trail
   entry.
5. **Distinguish the current entry.** Colour by `index < selectedStructure.length
   - 1` and show the separator chevron on the same test, so the last entry reads
   as "you are here" rather than as a link.
6. **Handle the ascent.** Re-point `menuItemList` at the clicked node's children
   **first**, then truncate the trail - guarded so the current entry is a no-op
   (HW-05-0140).
7. **Render the level list vertically** and derive every row affordance from
   `item.sonStructure.length > 0`.
8. **Handle the descent** on the same predicate: push the row onto the trail and
   re-point the level list at its children (HW-05-0141).
9. **Key both `ForEach` loops on `item.id`** - the model already carries it, and
   the default key would serialise the whole subtree (HW-05-0142).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The recursive model

`OrganizationalStructure.zip#entry/src/main/ets/model/MenuModel.ets`

```ts
export class Structure {
  id :number;
  name: string;
  avatar: Resource = $r('app.media.structure');
  sonStructure: Structure[];

  constructor(id: number, name: string, sonStructure: Structure[], avatar?: Resource) {
    this.id = id;
    this.name = name;
    this.sonStructure = sonStructure;
    if (typeof avatar !== 'undefined') {
      this.avatar = avatar;
    }
  }
}
```

The optional `avatar` with a field-level default is a neat touch: departments get
the generic icon, people can carry their own, and the constructor only overwrites
the default when one is actually supplied.

### State and seeding

`OrganizationalStructure.zip#entry/src/main/ets/pages/StructurePage.ets`

```ts
@Entry
@Component
struct StructurePage {
  researchList: Structure = RESEARCH_LIST;
  @State selectedStructure: Structure[] = [];
  @State menuItemList: Structure[] = [];

  aboutToAppear(): void {
    this.selectedStructure.push(this.researchList);
    this.menuItemList = this.researchList.sonStructure;
  }
}
```

Only the two navigation arrays are `@State`; the seed tree is a plain field,
because it is never mutated - the sample only ever re-points `menuItemList` at an
existing `sonStructure` array rather than editing one.

### The horizontal breadcrumb

`OrganizationalStructure.zip#entry/src/main/ets/pages/StructurePage.ets`

```ts
List() {
  ForEach(this.selectedStructure, (item: Structure, index: number) => {
    ListItem() {
      Row() {
        Text(item.name)
          .fontSize(16)
          .fontColor(index < this.selectedStructure.length - 1 ? '#0A59F7' : '#99000000')
          .onClick(() => {
            this.menuItemList = item.sonStructure;
            if (index !== this.selectedStructure.length - 1) {
              do {
                this.selectedStructure.pop();
              } while (this.selectedStructure.length - 1 > index);
            }
          });

        Image($r('app.media.ic_public_chevron_up'))
          .width(16)
          .margin({ left: 8, right: 8 })
          .visibility(index < this.selectedStructure.length - 1 ? Visibility.Visible : Visibility.Hidden);
      };
    };
  });                                        // no key generator - HW-05-0142
}
.height(38)
.listDirection(Axis.Horizontal)
.scrollBar(BarState.Off);
```

`listDirection(Axis.Horizontal)` on a fixed-height `List` is the whole breadcrumb
- it scrolls sideways for free once the trail is deeper than the screen, which a
`Row` would not.

The same `index < length - 1` test drives both the link colour and the separator
visibility, so the last entry is neither blue nor followed by a chevron.

### The vertical level list

`OrganizationalStructure.zip#entry/src/main/ets/pages/StructurePage.ets`

```ts
List() {
  ForEach(this.menuItemList, (item: Structure) => {
    ListItem() {
      Column() {
        Row() {
          Image(item.avatar)
            .width(item.sonStructure.length > 0 ? 48 : 52)
            .height(item.sonStructure.length > 0 ? 48 : 52);

          Text(item.name)
            .fontSize(16)
            .margin(item.sonStructure.length > 0 ? { left: 16 } : { left: 12 })
            .fontWeight(500);

          Blank();
          Image($r('app.media.chevron_right'))
            .width(12)
            .visibility(item.sonStructure.length > 0 ? Visibility.Visible : Visibility.Hidden);
        }
        .width('100%')
        .height('100%');

        Divider()
          .strokeWidth(0.5)
          .color('#33000000')
          .margin({ left: 64 });
      }
      .width('100%')
      .height('100%');
    }
    .width('100%')
    .padding(item.sonStructure.length > 0 ? { left: 0, right: 0 } : { left: 4, right: 4 })
    .height(item.sonStructure.length > 0 ? 65 : 80)
    .onClick(() => {
      if (item.sonStructure.length > 0) {
        this.menuItemList = item.sonStructure;
        this.selectedStructure.push(item);
      }
    });
  });                                        // no key generator - HW-05-0142
}
.width('100%')
.borderRadius(16)
.backgroundColor(Color.White)
.padding({ left: 12, right: 12 });
```

One predicate, six uses: `item.sonStructure.length > 0` sets the avatar size, the
label margin, the chevron visibility, the row padding, the row height and whether
the tap does anything. That is the reusable idea - a leaf and a branch differ only
by whether they have children, so nothing else needs to be modelled.

Corrected loops:

```ts
}, (item: Structure) => item.id.toString());
```

### Page frame

`OrganizationalStructure.zip#entry/src/main/ets/pages/StructurePage.ets`

```ts
Column() {
  Text($r('app.string.organizational_structure'))
    .fontSize(26)
    .fontWeight(700)
    .margin({ top: 10 });

  Search({ placeholder: $r('app.string.input_reminder') })
    .margin({ top: 16, bottom: 16 });

  // breadcrumb List, then level List
}
.width('100%')
.height('100%')
.backgroundColor('#F1F3F5')
.alignItems(HorizontalAlign.Start)
.padding({ left: 16, right: 16 })
.expandSafeArea([SafeAreaType.SYSTEM]);
```

`expandSafeArea([SafeAreaType.SYSTEM])` on the root column is why this sample
needs no `topRectHeight` / `px2vp` bookkeeping - the background extends under the
status bar and the framework handles the inset.

## Permissions & config

`OrganizationalStructure.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the organisation tree is the
local `RESEARCH_LIST` seed in `model/MenuData.ets`. The document has no 权限说明
section, which matches.

The sample also registers **no window listeners** at all in `EntryAbility` -
unlike most of this industry, which subscribes to `avoidAreaChange` and never
releases it. Using `expandSafeArea` instead of manual inset arithmetic is what
removes the need.

`build-profile.json5` pins the SDK to `6.0.0(20)` and enables
`caseSensitiveCheck: true`, which is why the `page/` vs `pages/` discrepancy in
the document's tree matters (HW-05-0143).

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The whole tree is in memory.** `RESEARCH_LIST` is a fully materialised
  recursive structure; a real directory that loads levels on demand would have to
  fetch `sonStructure` when a row is tapped, and the `sonStructure.length > 0`
  predicate would need a separate "has children" flag.
- **`Structure` is recursive**, so anything that serialises a node - including the
  default `ForEach` key - walks the entire subtree.
- **The breadcrumb never truncates.** A deep path scrolls sideways rather than
  collapsing to an ellipsis, which is workable but not what a deep hierarchy
  usually wants.
- **The `do…while` needs its guard.** It pops before testing, so it must not run
  when the clicked index is already the last one.
- **The search field is inert.** `Search` is rendered with a placeholder but has
  no `onChange` or `onSubmit` and nothing filters `menuItemList`.
- **Leaves are not selectable.** Tapping a row with no children does nothing at
  all - there is no detail page and no selection callback, so a real picker needs
  an else branch here.

## Pitfalls

- **The document's breadcrumb snippet is incorrect twice.** It drops the
  `if (index !== this.selectedStructure.length - 1)` guard, so tapping the current
  level still pops once and desynchronises the trail from the list a
  `do…while` always executes its body first - and it omits
  `this.menuItemList = item.sonStructure`, which is the line that actually moves
  the lower list. (HW-05-0140)
- **The document's drill-down snippet gates on `this.level > 1`, which is
  incorrect** - no such field exists anywhere in the sample. The shipped condition
  is `item.sonStructure.length > 0`, the same test that shows the chevron, and the
  snippet also omits the `selectedStructure.push(item)` that grows the breadcrumb.
  (HW-05-0141)
- **Neither `ForEach` declares a key generator, which is incorrect** for a
  recursive model: the default key runs `JSON.stringify` over a `Structure`, which
  serialises the entire subtree on every rebuild, and folds in the index, which
  shifts on every breadcrumb pop. Key on `item.id`. (HW-05-0142)
- **The project tree's `page/` is incorrect** - the sample ships `pages/`, and the
  build enables `caseSensitiveCheck`. (HW-05-0143)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` -
  `List`, `ListItem`, `listDirection` with `Axis.Horizontal`, and `scrollBar`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the default key generator `(item, index) => index + '__' + JSON.stringify(item)`
  and the warning against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` -
  `expandSafeArea` with `SafeAreaType.SYSTEM`, used here instead of manual inset
  arithmetic.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` -
  the `Search` component rendered in the page header.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
