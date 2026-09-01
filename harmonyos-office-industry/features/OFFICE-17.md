---
id: OFFICE-17
title: Multi-level corporate directory - a recursive @ObjectLink component with two-way parent/child selection
industry: 05_office
doc: huawei_industry_tree/05_office/docs/17_multi_level_nesting_list.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_level_nesting_list-0000002311817336
sample: huawei_industry_tree/05_office/downloads/MultiLevelNestingList.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: ["@Observed", "@ObjectLink", "@Builder", "@State", "@StorageProp", Checkbox, "Checkbox.select", "Checkbox.onChange", "$$ two-way binding", List, ListItem, ListItemGroup, ForEach, Scroller, "List.nestedScroll", "HashMap.set", "HashMap.get", "HashMap.forEach", Search, NavDestination, NavPathStack, "UIContext.px2vp", "UIContext.getPromptAction"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0093, HW-05-0094, HW-05-0095, HW-05-0096, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app needs a **collapsible organisation tree with
checkbox selection that propagates both ways** - pick a department and every
person under it is selected; pick the last remaining person and the department
ticks itself.

This is the recursion pattern in ArkUI: a component that renders one node and
then renders **itself** for each child, driven by `@Observed` / `@ObjectLink` so
that a mutation deep in the tree re-renders only the affected subtree.

Two mechanisms carry it:

- **`@Observed class Node` + `@ObjectLink item: Node`** - the node model is
  observable, and each `NodeItem` links to one node. Changing `node.selected`
  anywhere updates exactly the components bound to that node.
- **A `@Builder` that instantiates the enclosing struct** - `expandChildItem()`
  loops the children and creates a `NodeItem` for each, which is what makes the
  nesting arbitrary-depth.

No permission is involved; the data is a flat mock array joined into a tree at
page start.

## Feature checklist

The application must be able to:

- Build a tree from a flat `id` / `pid` list.
- Render one row per node with an expand/collapse chevron, a checkbox and the
  name, indented by depth.
- Expand and collapse a branch on row tap.
- Select a branch and have every descendant become selected.
- Deselect a descendant and have its ancestors lose their "all selected" state.
- Select the last unselected sibling and have the parent tick automatically.
- Keep a running count of selected leaves and show it at the bottom.
- Search the directory (the sample ships the box, not the filter).

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `utils/Node.ets` | the `@Observed class Node` - `id`, `pid`, `name`, `isExpand`, `children`, `parent`, `selected`, `selectedChildren` |
| `utils/MockData.ets` | the flat node array, module-level `const data` |
| `pages/MultiLevelNestingList.ets` | `@Entry`; joins the flat array into a tree with a `HashMap`, renders the root `NodeItem`, holds `selectedCnt` |
| `components/NodeItem.ets` | one node **and its recursive children**: the row, the checkbox, and the two propagation walks |
| `components/SearchBox.ets` | the search field |
| `utils/Logger.ets` | hilog wrapper |

The `Node` model carries both directions plus a counter, which is what makes the
upward propagation cheap:

```ts
@Observed
class Node {
  id: string;
  pid: string;
  name: string;
  isExpand: boolean = false;
  children: Array<Node> = [];
  parent: Node | null = null;
  selected: boolean = false;
  selectedChildren: number = 0;   // how many direct children are selected
}
```

Tree construction is a two-pass join over a `HashMap`: index every node by id,
then for each node with a real `pid`, look up the parent, set `parent` and push
into `parent.children`.

Selection propagates in two walks, both in `NodeItem`:

```
itemSelected(node)
  node has children ->  checkChildren(node, node.selected)   downward: set every descendant
                        checkParent(node, node.selected)     upward: adjust ancestors' counters
  node is a leaf    ->  checkParent(node, node.selected)
                        selectedItem(node)                   report to the page's counter
```

`selectedItem` is a callback threaded down through every level, so a leaf deep in
the tree can increment the page's `selectedCnt` without any global state.

## Implementation steps

1. **Declare no permission.** Nothing here leaves the process; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Make the model observable.** `@Observed class Node` with `children`,
   `parent` and the `selectedChildren` counter. `@Observed` is what allows a
   nested property change to be seen by the `@ObjectLink` that holds it.
3. **Join the flat list into a tree once.** Index by `id` in a `HashMap`, then
   link parents and children. Check each lookup (HW-05-0096) and make the join
   idempotent, or run it once at module load - re-running it appends duplicate
   children to the shared nodes (HW-05-0093).
4. **Write the recursive component.** `@ObjectLink item: Node` plus a
   `@Builder expandChildItem()` that loops `this.item.children` and instantiates
   `NodeItem` again, passing `leftSpace + 32` for the indent and forwarding the
   `selectedItem` callback.
5. **Key the children stably.** `(subItem: Node) => subItem.id + subItem.name` -
   a real per-item key, unlike the `JSON.stringify` keys elsewhere in this
   industry.
6. **Do not nest a scrolling `List` per level.** Use one `List` at the page level
   and render the recursive levels as `ListItemGroup`s or plain `Column`s
   (HW-05-0094).
7. **Handle the checkbox in `onChange`,** taking the new value from the callback
   rather than re-reading the `$$`-bound field (HW-05-0095).
8. **Propagate downward** with `checkChildren(node, value)`: for a leaf whose
   state actually changes, set it and report through `selectedItem`; for a branch,
   recurse into each child, then set `selectedChildren` to `children.length` or
   `0`.
9. **Propagate upward** with `checkParent(node, value)`: adjust the parent's
   `selectedChildren` by one, set `parent.selected` when the counter reaches
   `children.length`, and recurse.
10. **Count only leaves.** The page's `checkResult` moves `selectedCnt` by one per
    `selectedItem` call, and `checkChildren` only fires it when a leaf's state
    genuinely changed - that guard is what keeps the count correct.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The observable node model

`MultiLevelNestingList.zip#entry/src/main/ets/utils/Node.ets`

```ts
@Observed
class Node {
  id: string;
  pid: string;
  name: string;
  isExpand: boolean = false;
  children: Array<Node> = [];
  parent: Node | null = null;
  selected: boolean = false;
  selectedChildren: number = 0;

  constructor(id: string, pid: string, name: string) {
    this.id = id;
    this.pid = pid;
    this.name = name;
  }
}

export { Node };
```

`MultiLevelNestingList.zip#entry/src/main/ets/utils/MockData.ets`

```ts
// 准备数据
const data: Array<Node> = [];
data[0] = new Node('root', 'null', '研发部门');
data[1] = new Node('2', 'root', '开发部');
data[2] = new Node('3', 'root', '测试部');
data[3] = new Node('4', '2', '前端开发部');
data[4] = new Node('5', '2', '后端开发部');
data[5] = new Node('6', '3', '测试一部');
data[6] = new Node('7', '3', '测试二部');

export { data };
```

Note the sentinel: the root's `pid` is the **string** `'null'`, not `null`, and
the join tests `item.pid !== 'null'`.

### Flat list to tree

`MultiLevelNestingList.zip#entry/src/main/ets/pages/MultiLevelNestingList.ets`

```ts
import { HashMap } from '@kit.ArkTS';

hashMap = new HashMap<string, Node>();
rootItem: Node | null = null;

aboutToAppear(): void {
  // 录入数据
  for (let i = 0; i < data.length; i++) {
    this.hashMap.set(data[i].id, data[i]);
  }
  // 构建节点关系
  this.hashMap.forEach((item: Node, key: string) => {
    if (item.pid !== 'null') {
      let parent: Node = this.hashMap.get(item.pid);   // unchecked - HW-05-0096
      item.parent = parent;
      parent.children.push(item);                      // mutates shared data - HW-05-0093
    }
  });

  this.rootItem = this.hashMap.get('root');
}
```

Corrected join:

```ts
aboutToAppear(): void {
  data.forEach((n: Node) => {           // make the join idempotent
    n.children = [];
    n.parent = null;
    this.hashMap.set(n.id, n);
  });
  this.hashMap.forEach((item: Node) => {
    if (item.pid === 'null') {
      return;
    }
    const parent = this.hashMap.get(item.pid);
    if (!parent) {
      Logger.error(`missing parent ${item.pid} for node ${item.id}`);
      return;
    }
    item.parent = parent;
    parent.children.push(item);
  });
  this.rootItem = this.hashMap.get('root') ?? null;
}
```

### The recursive component

`MultiLevelNestingList.zip#entry/src/main/ets/components/NodeItem.ets`

```ts
@Component
export struct NodeItem {
  @ObjectLink item: Node;
  @State leftSpace: number = 0;
  @State isExpand: boolean = false;
  scroller: Scroller = new Scroller();
  selectedItem: (item: Node) => void = () => {};

  aboutToAppear(): void {
    this.isExpand = this.item.isExpand;
  }

  build() {
    Column() {
      Column() {
        Row() {
          if (this.item.children.length > 0) {
            Image(this.isExpand ? $r('app.media.ic_open') : $r('app.media.ic_fold'))
              .width($r('app.float.node_expand_icon_len'))
              .height($r('app.float.node_expand_icon_len'));
          } else {
            Text().width($r('app.float.node_expand_icon_len')).height($r('app.float.node_expand_icon_len'));
          }

          Row() {
            Checkbox({ name: this.item.name })
              .select($$this.item.selected)
              .onClick(() => {                       // should be onChange - HW-05-0095
                this.itemSelected(this.item);
              });
            Text(this.item.name).layoutWeight(1);
          }
          .margin({ left: $r('app.float.node_item_row_left_margin') });
        }
        .margin({ left: this.leftSpace })            // depth indent
        .onClick(() => {
          this.isExpand = !this.isExpand;
          this.item.isExpand = this.isExpand;
        });
      };

      if (this.item.children.length > 0 && this.isExpand) {
        this.expandChildItem();
      }
    };
  }

  //子级数据
  @Builder
  expandChildItem() {
    List({ scroller: this.scroller }) {               // nested List - HW-05-0094
      ForEach(this.item.children, (subItem: Node, index: number) => {
        ListItem() {
          NodeItem({
            item: subItem,
            leftSpace: this.leftSpace + 32,
            selectedItem: (item: Node) => {
              this.selectedItem(item);
            }
          });
        };
      }, (subItem: Node) => subItem.id + subItem.name);
    };
  }
}
```

The three things worth reusing verbatim: the `@Builder` that re-instantiates the
enclosing struct, the `leftSpace + 32` indent threaded down as a parameter, and
the `subItem.id + subItem.name` key generator - a genuine stable key, unlike the
`JSON.stringify` keys used in OFFICE-01, OFFICE-13 and OFFICE-15.

The empty `Text()` in the else branch is the placeholder that keeps leaf rows
aligned with branch rows.

### Two-way selection propagation

`MultiLevelNestingList.zip#entry/src/main/ets/components/NodeItem.ets`

```ts
itemSelected(item: Node) {
  //item.selected = !item.selected; // 已绑定不需要修改
  if (item.children.length > 0) {
    Logger.info(`选择了${item.id}`);
    this.checkChildren(item, item.selected);
    this.checkParent(item, item.selected);
  } else {
    Logger.info(`选择了${item.id}`);
    this.checkParent(item, item.selected);
    this.selectedItem(item);
  }
}

checkChildren(item: Node, parentSelected: boolean) {
  if (!item.children.length && item.selected !== parentSelected) {
    item.selected = parentSelected;
    this.selectedItem(item);            // only leaves report, and only on a real change
  }
  item.children.forEach(item => {
    this.checkChildren(item, parentSelected);
    item.selected = parentSelected;
  });
  item.selectedChildren = parentSelected ? item.children.length : 0;
}

checkParent(item: Node, childSelected: boolean) {
  if (item.parent) {
    if (item.parent.selectedChildren === item.parent.children.length) {
      item.parent.selected = false;
      this.checkParent(item.parent, false);
    }
    item.parent.selectedChildren += childSelected ? 1 : -1;
    if (item.parent.selectedChildren === item.parent.children.length) {
      item.parent.selected = true;
      this.checkParent(item.parent, true);
    }
  }
}
```

`checkChildren`'s inner `forEach` parameter shadows the outer `item` - the
assignment on the line after the recursive call sets the **child**, not the node
passed in. Rename it when adapting this.

The guard `item.selected !== parentSelected` before `selectedItem` is what keeps
the page's counter accurate: a leaf that is already in the target state does not
report again.

### The page shell and the counter

`MultiLevelNestingList.zip#entry/src/main/ets/pages/MultiLevelNestingList.ets`

```ts
@State selectedCnt: number = 0;

List({ scroller: this.scroller }) {
  ListItem() {
    NodeItem({
      item: this.rootItem!, selectedItem: (item: Node) => {
        this.checkResult(item);
      }
    });
  };
}
.scrollBar(BarState.Off);

//列表点击
checkResult(item: Node) {
  if (item.selected) {
    this.selectedCnt++;
  } else {
    this.selectedCnt--;
  }
}
```

## Permissions & config

`MultiLevelNestingList.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the directory is local mock
data and the whole feature is layout plus state. The document has no 权限说明
section, which matches.

The insets come from `AppStorage` and are converted at the point of use:

```ts
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;

.padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
.padding({ top: this.getUIContext().px2vp(this.topRectHeight) })
```

The document's project tree matches the shipped ZIP exactly, including
`entrybackupability`. `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`@ObjectLink` requires `@Observed` on the class.** The recursion only
  re-renders correctly because `Node` is decorated; an undecorated model would
  leave deep mutations invisible.
- **The tree is joined from a flat list at page start**, not fetched. Any real
  directory would replace `MockData` with a service call, at which point the
  idempotence problem in HW-05-0093 becomes unavoidable rather than incidental.
- **`pid` uses the string `'null'` as the root sentinel**, not a null value - the
  join tests `item.pid !== 'null'`.
- **`selectedChildren` counts direct children only.** `checkParent` moves it by
  one per event, so any code path that changes several children at once must
  update it wholesale, as `checkChildren` does at its last line.
- **`HashMap.get` returns `undefined` for an absent key**, per the reference, so
  every lookup in the join is a potential failure point.
- **The search box is a stub.** `SearchBox` renders the field; nothing filters the
  tree.
- **No leaf people in the mock data.** All seven nodes are departments, so the
  "select a department, get its people" behaviour is demonstrated with
  sub-departments standing in for people.

## Pitfalls

- **Building the tree in `aboutToAppear` by pushing into the shared nodes'
  `children` is incorrect.** `MockData` exports module-level `Node` objects, and
  the push is never preceded by a reset, so a second visit to the page duplicates
  every child - and `checkParent`, which compares `selectedChildren` against
  `children.length`, then works against a doubled length. Clear the relationships
  first, or build the tree once at module load. (HW-05-0093)
- **Nesting a `List` per recursion level is incorrect.** The reference states that
  a scrollable component nested in a `List` with the same direction and no
  main-axis size makes the outer `List` load every child, so lazy loading does not
  take effect; and `nestedScroll` defaults to `SELF_ONLY`, so the inner lists do
  not hand scrolling back. Use one `List` with `ListItemGroup`s, and drop the
  per-node `Scroller` that nothing calls. (HW-05-0094)
- **Handling the checkbox in `onClick` is incorrect.** The documented event for a
  selection change is `onChange`, which supplies the new value; `onClick` forces
  the code to re-read the `$$`-bound field and assume the binding has already been
  written - the assumption the commented-out line at `NodeItem.ets:111` records.
  (HW-05-0095)
- **`let parent: Node = this.hashMap.get(item.pid)` is incorrect.** `get` returns
  `undefined` when the key is absent, and `parent.children.push(item)` on the next
  line then throws inside `aboutToAppear`. Check the lookup, and validate
  `rootItem` instead of forcing it with `!`. (HW-05-0096)
- Not a separate finding, but fix it when adapting: `checkChildren`'s inner
  `forEach` parameter shadows the outer `item`, which makes the two assignments in
  that function read as though they target the same object.

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` -
  "If a scrollable component is nested in a List component, their scrolling
  directions are the same, and the main axis size is not set for the List
  component, the List component loads all child components. As a result, lazy
  loading does not take effect." plus the `nestedScroll10+` default of
  `SELF_ONLY`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-checkbox.md` -
  `select(value: boolean)` with `$$` two-way binding, and `onChange` as the event
  "Invoked when the selected state of the check box changes".
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-checkbox
- `documentation/harmonyos-references/02_application-framework/js-apis-hashmap.md` -
  "get(key: K): V - Obtains the value of the specified key in this HashMap. If
  nothing is obtained, undefined is returned."
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hashmap
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` -
  `@Observed` / `@ObjectLink` for nested object property changes.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-guides/03_application-framework/arkts-builder.md` -
  `@Builder` custom build functions, used here for the recursion.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-builder
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  `keyGenerator` semantics; the sample's `subItem.id + subItem.name` satisfies it.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
