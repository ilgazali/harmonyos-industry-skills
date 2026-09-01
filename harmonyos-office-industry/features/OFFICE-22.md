---
id: OFFICE-22
title: Add special-attention contacts - alphabet-indexed List with checkbox multi-select returned through the nav stack
industry: 05_office
doc: huawei_industry_tree/05_office/docs/22_special_following.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
sample: huawei_industry_tree/05_office/downloads/SpecialFollowing.zip
kits: ["@kit.ArkUI"]
apis: [List, ListItem, "List.onScrollIndex", Scroller, "Scroller.scrollToIndex", ForEach, Checkbox, "Checkbox.select", "Checkbox.onChange", CheckBoxShape, TextInput, TextInputController, Stack, "position()", "hitTestBehavior", Navigation, NavPathStack, NavDestination, "NavPathStack.pushPath", "NavPathStack.pop", routerMap, "@Provide", "@Consume", "@State", "@StorageProp", "UIContext.px2vp", "UIContext.getPromptAction", "window.on('avoidAreaChange')", "window.off('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0124, HW-05-0125, HW-05-0126, HW-05-0127, HW-05-0128, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a contacts screen needs an **alphabet-indexed picker with
multi-select** - the standard "choose people" surface: contacts grouped by first
letter, an A-Z rail down the right edge that jumps the list, checkboxes on each
row, a running count, and a confirm button that hands the selection back to the
calling page.

Two ArkUI mechanisms carry it:

- **`Scroller.scrollToIndex` + `List.onScrollIndex`** form a two-way link between
  the alphabet rail and the list: tapping a letter scrolls the list, and
  scrolling the list highlights the letter.
- **`NavPathStack.pushPath` with `onPop`** returns the selection: the picker
  writes into a `@Consume`d array and pops, and the caller's `onPop` callback
  fires the confirmation toast.

No permission is involved - the contact list is local mock data.

## Feature checklist

The application must be able to:

- Show an empty state when nothing is followed yet, and the follow list once
  something is.
- Open the picker from the title bar's add icon and receive the result on pop.
- Group contacts by first letter and render one card per group with a letter
  header.
- Draw an A-Z rail positioned over the list, highlighting the letter of the group
  currently at the top.
- Scroll the list to a group when its letter is tapped.
- Check and uncheck a contact, keeping a running selected count.
- Reflect the existing selection when the picker is re-opened.
- Select or clear everything from a select-all control.
- Confirm the selection, write it back to the caller and pop.

## Architecture

Single `entry` HAP, two pages:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes `topRectHeight` / `bottomRectHeight`, loads `pages/MainPage` |
| `pages/MainPage.ets` | `@Entry`; owns the `NavPathStack` and the `myFollows` array, both `@Provide`d; empty state vs follow list |
| `pages/FollowPage.ets` | the picker `NavDestination`: search bar, grouped list, alphabet rail, action bar |
| `components/Title.ets` | shared title bar with optional left/right icons and a click action |
| `model/Contact.ets` | `ContactInfo` (`name`, `img`, `firstLetter`) and the `CONTACTS` seed |
| `constants/CommonConstants.ets` | sizes, spacings, the `FOLLOW_PAGE` route name |

State crosses the two pages by alias, not by parameter:

```ts
// MainPage
@Provide('pathStack') pathStack: NavPathStack = new NavPathStack();
@Provide('myFollows') myFollows: ContactInfo[] = [];

// FollowPage
@Consume('pathStack') pathStack: NavPathStack;
@Consume('myFollows') myFollows: ContactInfo[];
```

The picker therefore writes its result **directly into the caller's state**
(`this.myFollows = [...this.selectedData]`) and then pops; `onPop` is used only
for the confirmation toast, not to carry data.

Grouping is derived once at page entry:

```
aboutToAppear
  totalData.forEach -> contactGroupSet.add(item.firstLetter)   // Set<string>, dedupes
  contactGroupArray = Array.from(contactGroupSet)              // ordered, indexable

build
  List(scroller)                       one ListItem per letter -> cardBuilder(letter)
    .onScrollIndex(cardIndex =>        alphabetValue.indexOf(contactGroupArray[cardIndex])
                                       -> selectedIndex, which highlights the rail)
  List(alphabet)  .position(x, y)      26 letters; tap -> contactGroupArray.indexOf(letter)
                                       -> scroller.scrollToIndex(cardIndex)
```

The rail is a second `List` placed with `.position()` inside a `Stack`, so it
floats over the contact list rather than taking layout space.

## Implementation steps

1. **Declare no permission.** The contact data is local; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Declare the route.** `"routerMap": "$profile:route_map"` with an entry
   binding the picker's name to its global `@Builder` (`followPageBuilder`).
3. **Provide the shared state at the caller.** The `NavPathStack` and the result
   array, both under alias keys so the destination can `@Consume` them.
4. **Derive the letter groups once** in `aboutToAppear` - a `Set` for
   deduplication and an `Array.from` copy for index lookups. Precompute the
   per-letter contact lists here too, rather than filtering inside the builder
   (HW-05-0125).
5. **Render one card per group** and a divider between rows but not after the
   last one.
6. **Bind each checkbox to the selection model.** `select(...)` must be derived
   from the selection, not hard-coded, and the picker should seed itself from the
   caller's existing follows (HW-05-0124).
7. **Give every `ForEach` a key generator** over a stable field (HW-05-0126).
8. **Wire the alphabet rail both ways.** `onScrollIndex` maps the top group index
   back to a letter index for the highlight; the rail's `onClick` maps a letter
   back to a group index for `scrollToIndex`.
9. **Keep the search overlay non-interactive.** The magnifier icon sits in a
   `Row` over the `TextInput` with `hitTestBehavior(HitTestMode.None)` so taps
   fall through to the field.
10. **Confirm and pop.** Copy the selection into the `@Consume`d array and
    `pathStack.pop(...)`; the caller's `onPop` shows the toast.
11. **Release the window listener** in `onWindowStageDestroy` (HW-05-0128).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Grouping and the two linked lists

`SpecialFollowing.zip#entry/src/main/ets/pages/FollowPage.ets`

```ts
@State contactGroupSet: Set<string> = new Set<string>(); // set of contact group
@State contactGroupArray: string[] = [];
totalData: ContactInfo[] = CONTACTS;
private scroller: Scroller = new Scroller;
private alphabetValue: string[] = ['A', 'B', 'C', 'D', 'E', 'F', 'G',
  'H', 'I', 'J', 'K', 'L', 'M', 'N',
  'O', 'P', 'Q', 'R', 'S', 'T', 'U',
  'V', 'W', 'X', 'Y', 'Z'];

aboutToAppear(): void {
  this.totalData.forEach((item) => {
    this.contactGroupSet.add(item.firstLetter);
  });
  this.contactGroupArray = Array.from(this.contactGroupSet);
}

// in build()
Stack({ alignContent: Alignment.Start }) {
  List({ scroller: this.scroller, space: CommonConstants.LIST_SPACE }) {
    ForEach(Array.from(this.contactGroupSet), (letter: string) => {
      ListItem() {
        this.cardBuilder(letter);
      };
    });                                             // no key generator - HW-05-0126
  }
  .width(CommonConstants.FULL_WIDTH)
  .height(CommonConstants.FULL_HEIGHT)
  .onScrollIndex((cardIndex: number) => {
    let index = this.alphabetValue.indexOf(this.contactGroupArray[cardIndex]);
    this.selectedIndex = index;
  })
  .scrollBar(BarState.Off);

  List({ space: CommonConstants.ALPHABET_SPACE }) {
    ForEach(this.alphabetValue, (item: string, index: number) => {
      ListItem() {
        this.alphabetBuilder(item, index);
      };
    });
  }
  .position({ x: CommonConstants.ALPHABET_X, y: CommonConstants.ALPHABET_Y });
}
.layoutWeight(1);
```

The `Stack` + `.position()` rail is the reusable idea: the alphabet list is
absolutely placed over the contact list, so it never competes for layout width
and stays pinned while the contacts scroll.

### Rail to list, and list to rail

`SpecialFollowing.zip#entry/src/main/ets/pages/FollowPage.ets`

```ts
@Builder
alphabetBuilder(alphabet: string, index: number) {
  Column() {
    Text(alphabet)
      .fontSize(CommonConstants.MINI_FONTSIZE)
      .textAlign(TextAlign.Center)
      .fontColor(index === this.selectedIndex ? $r('app.color.selected_color') :
        $r('app.color.first_letter_color'));
  }
  .borderRadius(CommonConstants.ALPHABET_BORDER_RADIUS)
  .width(CommonConstants.ALPHABET_WIDTH)
  .height(CommonConstants.ALPHABET_HEIGHT)
  .border(
    index === this.selectedIndex ?
      { width: CommonConstants.BORDER_WIDTH, color: $r('app.color.selected_color') } :
      { width: CommonConstants.BORDER_WIDTH_ZERO }
  )
  .backgroundColor(index === this.selectedIndex ? $r('app.color.alphabet_selected_color') :
    $r('app.color.main_back'))
  .onClick(() => {
    let cardIndex = this.contactGroupArray.indexOf(this.alphabetValue[index]);
    this.scroller.scrollToIndex(cardIndex);
  });
}
```

Note the two different index spaces: `selectedIndex` is an index into the 26
`alphabetValue` letters, while `scrollToIndex` needs an index into
`contactGroupArray` - only the letters that actually have contacts. The two
`indexOf` calls are the translation between them.

### The contact row and its checkbox

`SpecialFollowing.zip#entry/src/main/ets/pages/FollowPage.ets`

```ts
@Builder
cardBuilder(firstLetter: string) {
  Column() {
    Row() {
      Text(firstLetter)
        .fontSize(CommonConstants.SMALL_FONTSIZE)
        .fontColor($r('app.color.first_letter_color'));
    }
    .width(CommonConstants.FULL_WIDTH)
    .height(CommonConstants.FIRST_LETTER_ROW_HEIGHT);

    Column() {
      ForEach(this.totalData.filter((item: ContactInfo) => {     // filtered here - HW-05-0125
        return item.firstLetter === firstLetter;
      }), (data: ContactInfo, index: number) => {
        Flex({ justifyContent: FlexAlign.SpaceBetween, alignItems: ItemAlign.Center }) {
          Row() {
            Checkbox()
              .select(false)                                     // not bound - HW-05-0124
              .selectedColor($r('app.color.selected_color'))
              .shape(CheckBoxShape.CIRCLE)
              .onChange((value: boolean) => {
                if (value) {
                  this.selectedData.push(data);
                } else {
                  this.selectedData = this.selectedData.filter(item => item !== data);
                }
              });
            Image(data.img)
              .width(CommonConstants.ADD_ICON_WIDTH)
              .height(CommonConstants.ADD_ICON_HEIGHT)
              .margin({ left: CommonConstants.MARGIN_SIXTEEN, right: CommonConstants.MARGIN_FOURTEEN });
            Text(data.name);
          };

          Row() {
            Image($r('app.media.call')) /* ... */;
            Image($r('app.media.message')) /* ... */;
          };
        }
        .height(CommonConstants.CARD_ROW_HEIGHT);

        if (index !== this.totalData.filter((item: ContactInfo) => {   // filtered again - HW-05-0125
          return item.firstLetter === firstLetter;
        }).length - 1) {
          Divider()
            .margin({ left: CommonConstants.MARGIN_ONE_HUNDRED })
            .strokeWidth(CommonConstants.STROKE_WIDTH)
            .color($r('app.color.divider_color'));
        }
      });
    };
  };
}
```

Corrected group handling and checkbox binding:

```ts
// aboutToAppear
private groups: Map<string, ContactInfo[]> = new Map();
// ...
this.totalData.forEach((item) => {
  this.contactGroupSet.add(item.firstLetter);
  const list = this.groups.get(item.firstLetter) ?? [];
  list.push(item);
  this.groups.set(item.firstLetter, list);
});
this.contactGroupArray = Array.from(this.contactGroupSet);
this.selectedData = [...this.myFollows];     // seed from the existing follows

// cardBuilder
const group = this.groups.get(firstLetter) ?? [];
ForEach(group, (data: ContactInfo, index: number) => {
  // ...
  Checkbox()
    .select(this.selectedData.includes(data))
    .onChange((value: boolean) => { /* ... */ });
  // ...
  if (index !== group.length - 1) {
    Divider() /* ... */;
  }
}, (data: ContactInfo) => data.name);
```

### The action bar and the hand-back

`SpecialFollowing.zip#entry/src/main/ets/pages/FollowPage.ets`

```ts
@Builder
actionBar() {
  Flex({ justifyContent: FlexAlign.SpaceBetween, alignItems: ItemAlign.Center }) {
    Row() {
      Checkbox()
        .select(false)
        .selectedColor($r('app.color.selected_color'))
        .shape(CheckBoxShape.CIRCLE);                    // no onChange at all - HW-05-0124
      Text($r('app.string.select_all'));
      Text($r('app.string.selected'));
      Text(`${this.selectedData.length}人`)
        .fontColor($r('app.color.selected_color'));
    };

    Row() {
      Button(`确定（${this.selectedData.length}/1000）`)
        .onClick(() => {
          this.myFollows = [...this.selectedData];
          this.pathStack.pop({ data: '' });
        });
    };
  }
  .position({ x: 0, y: CommonConstants.CONTAINER_HEIGHT })
  .padding({
    left: CommonConstants.PADDING_SIXTEEN,
    right: CommonConstants.PADDING_SIXTEEN,
    bottom: this.getUIContext().px2vp(this.bottomRectHeight)
  });
}
```

`this.myFollows = [...this.selectedData]` is the whole hand-back: assigning to a
`@Consume`d array writes straight through to the provider on `MainPage`, so the
`pop` needs no payload.

### The caller

`SpecialFollowing.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
@Provide('pathStack') pathStack: NavPathStack = new NavPathStack();
@Provide('myFollows') myFollows: ContactInfo[] = [];

Title({
  title: $r('app.string.title_special_attention'),
  rightIcon: $r('app.media.add'),
  clickAction: () => {
    this.pathStack.pushPath({
      name: CommonConstants.FOLLOW_PAGE,
      onPop: () => {
        this.promptAction.showToast({
          message: $r('app.string.follow_successfully'),
          textColor: $r('app.color.selected_color'),
          offset: { dx: 0, dy: CommonConstants.TOAST_OFFSET_Y }
        });
      }
    });
  }
});

Column() {
  if (this.myFollows.length >= 1) {
    this.followCardBuilder();
  } else {
    this.emptyFollowBuilder();
  }
};
```

### Search field with a non-interactive icon overlay

`SpecialFollowing.zip#entry/src/main/ets/pages/FollowPage.ets`

```ts
Stack() {
  TextInput({
    placeholder: $r('app.string.search_placeholder'),
    controller: this.controller,
    text: this.currentSearchContent
  })
    .padding({ left: CommonConstants.MARGIN_FORTY })    // room for the icon
    .backgroundColor($r('app.color.search_bar_back'));
  Row() {
    Image($r('app.media.search'))
      .width(CommonConstants.MINI_ICON_WIDTH)
      .height(CommonConstants.MINI_ICON_HEIGHT);
  }
  .width(CommonConstants.FULL_WIDTH)
  .hitTestBehavior(HitTestMode.None)                    // taps fall through to the field
  .justifyContent(FlexAlign.SpaceBetween)
  .padding({ left: CommonConstants.PADDING_SIXTEEN, right: CommonConstants.PADDING_SIXTEEN });
}
.alignContent(Alignment.Start)
```

`hitTestBehavior(HitTestMode.None)` on the overlay plus a matching left padding
on the field is the general recipe for decorating an input without stealing its
taps.

## Permissions & config

`SpecialFollowing.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the contact list is the local
`CONTACTS` array in `model/Contact.ets`. The document has no 权限说明 section,
which matches.

`"routerMap": "$profile:route_map"` is declared, and the picker is reached by
name through `pathStack.pushPath({ name: CommonConstants.FOLLOW_PAGE, onPop })`;
`FollowPage.ets` exports the global `@Builder followPageBuilder` the route entry
names.

`build-profile.json5` pins the SDK to `6.0.0(20)` and enables
`caseSensitiveCheck: true`, which is why the two project-tree path discrepancies
matter (HW-05-0127).

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`firstLetter` is supplied by the data, not derived.** `ContactInfo` carries it
  as a field, so a real directory has to compute it - including pinyin
  initials for Chinese names, which this sample side-steps by hard-coding it.
- **The two index spaces are different.** `selectedIndex` indexes the 26-letter
  rail; `scrollToIndex` needs an index into `contactGroupArray`, which only holds
  letters that have contacts. Confusing them silently scrolls to the wrong group.
- **`indexOf` returns -1 for a missing letter**, so both translations can produce
  -1 - the rail simply highlights nothing, but `scrollToIndex(-1)` is worth
  guarding in a real list.
- **The alphabet rail is absolutely positioned.** Its `x`/`y` come from constants,
  so it does not adapt to a different screen size or to the action bar's height.
- **The action bar is positioned, not laid out.** `.position({ x: 0, y:
  CommonConstants.CONTAINER_HEIGHT })` pins it below a fixed-height container
  rather than using `layoutWeight`, so changing the container height moves it.
- **The search field is inert.** `currentSearchContent` is declared and bound but
  nothing filters on it, and the `onClick` handler is empty.
- **The selection cap is cosmetic.** The confirm button reads
  `确定（n/1000）` but nothing enforces the 1000 limit.

## Pitfalls

- **`Checkbox().select(false)` is incorrect** as a selection control: the state is
  write-only, so re-entering the picker with existing follows shows every row
  unchecked, and the select-all checkbox next to its label has no `onChange` at
  all. Bind `select` to the selection and seed it from `myFollows`.
  (HW-05-0124)
- **Filtering `totalData` inside the builder - twice - is incorrect.** The group
  list is recomputed for the `ForEach` source and again for every row just to
  learn the group length. Compute the groups once in `aboutToAppear`.
  (HW-05-0125)
- **Four `ForEach` loops with no key generator are incorrect.** The default key
  embeds the index and a `JSON.stringify` of the item, which for the contact rows
  serialises a `ContactInfo` including its `Resource` on every rebuild. Key on
  `data.name` and on the letter. (HW-05-0126)
- **The project tree's `title.ets` and `page/` are incorrect** - the sample ships
  `Title.ets` and `pages/`, and the build enables `caseSensitiveCheck`.
  (HW-05-0127)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect** -
  release it in `onWindowStageDestroy`. (HW-05-0128)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-checkbox.md` -
  `select(value: boolean)` with `$$` two-way binding, and `onChange` as the
  selection-change event.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-checkbox
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` -
  `List`, `ListItem`, `onScrollIndex` and the `Scroller` binding.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the default key generator and the warning against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on`/`off('avoidAreaChange')` and `getWindowAvoidArea`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/ts-container-scrollable-common.md` -
  `Scroller.scrollToIndex`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scrollable-common
