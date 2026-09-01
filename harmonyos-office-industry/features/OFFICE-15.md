---
id: OFFICE-15
title: Handle a message later - long-press bubble menu that pushes a chat message into a to-do list
industry: 05_office
doc: huawei_industry_tree/05_office/docs/15_later_items.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
sample: huawei_industry_tree/05_office/downloads/LaterItems.zip
kits: ["@kit.ArkUI", "@kit.ArkData", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [bindPopup, LongPressGesture, "gesture()", Placement, "@Provide", "@Consume", "@ObjectLink", "@Observed", "@Prop", "@StorageLink", "preferences.getPreferencesSync", "Preferences.putSync", "Preferences.getSync", "Preferences.hasSync", "Preferences.flush", "PromptAction.showDialog", Navigation, NavPathStack, NavigationMode, Tabs, TabContent, ForEach, RichEditor, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.AvoidAreaType.TYPE_SYSTEM", "AppStorage.setOrCreate", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0085, HW-05-0086, HW-05-0087, HW-05-0088, HW-05-0089, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a messaging surface needs a **"deal with this later"** action:
the user long-presses a chat bubble, a small menu appears above it, and choosing
one entry moves that message into a to-do list that survives a restart.

Three ArkUI mechanisms carry it:

- **`LongPressGesture` + `bindPopup`** - the bubble binds a popup whose visibility
  is a `@State` boolean, and a long-press flips that boolean. The popup's
  `builder` renders a custom horizontal menu, not a system menu.
- **`@Provide` / `@Consume` across three levels** - the page provides two arrays
  (pending and finished); the chat bubble deep in the tree consumes the pending
  one and pushes into it; the to-do view consumes both and renders them.
- **`Preferences` for persistence** - the whole list is serialised to one JSON
  string on page teardown and parsed back on page appear.

No permission is involved.

## Feature checklist

The application must be able to:

- Show a chat list, open a conversation, and render each message as a bubble.
- Pop a custom menu above a bubble on long press, and dismiss it on selection or
  tap-outside.
- Add the long-pressed message to a shared pending to-do array, recording who
  sent it, their avatar, the text and the time.
- Show the pending and finished to-do groups, each with a bulk action.
- Ask for confirmation before "mark all done" or "clear all".
- Move every pending item into the finished group, or empty the finished group.
- Persist both groups across app restarts.
- Lay out under the status bar and navigation indicator.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | creates the `Preferences` instance in `onCreate`, immersive layout, publishes `avoidTop` / `avoidBottom` |
| `pages/HomePage.ets` | `@Entry`; the tab shell, **the two `@Provide` arrays**, and the load/save lifecycle hooks |
| `pages/ChatContentPage.ets` | a single conversation |
| `view/MessageTab.ets`, `view/ChatHomeView.ets` | the message tab and friend list |
| `view/LaterProcessView.ets` | renders the pending and finished groups |
| `component/CustomText.ets` | the chat bubble - **long-press gesture, popup menu, push to to-do** |
| `component/ItemComp.ets` | one to-do group with its bulk action and confirmation dialog |
| `component/CustomRichEditor.ets` | the message input |
| `model/DataModel.ets` | `Conversation`, `MsgContent`, `ToDoItem`, `MENU_ITEMS`, `GetAvatar` |
| `database/PreferenceUtils.ets` | the `Preferences` singleton wrapper |

The state contract is the interesting part. `HomePage` provides two arrays under
**explicit alias keys**, and consumers bind by the same alias:

```ts
// HomePage.ets
@Provide('toDoItems') todoItems: ToDoItem[] = [];
@Provide('toDoItems_finished') todoItemsFinished: ToDoItem[] = [];

// CustomText.ets - the bubble, several levels down
@Consume('toDoItems') toDoItems: ToDoItem[];

// ItemComp.ets - the to-do group
@Consume('toDoItems') toDoItems: ToDoItem[];
@Consume('toDoItems_finished') todoItemsFinished: ToDoItem[];
```

The alias matters: the provider's variable is `todoItems` (lower-case d) but the
key is `'toDoItems'`, so consumers must use the key, not the field name.

Persistence flow:

```
EntryAbility.onCreate
  preferenceUtils.getItemsPreferences(this.context)     -> getPreferencesSync

HomePage.aboutToAppear
  getChangeItemsPreferences()  -> getSync(KEY, '')      -> JSON string
  JSON.parse -> split by isDone -> push into the two @Provide arrays

HomePage.aboutToDisappear
  [...todoItems, ...todoItemsFinished] -> JSON.stringify
  saveChangeItemsPreferences()  -> putSync + flush
```

Note the ordering dependency: `getItemsPreferences` runs in `onCreate`, before
`loadContent`, so `this.preference` is already set when `HomePage.aboutToAppear`
reads it. Keep that ordering if you move the initialisation.

## Implementation steps

1. **Declare no permission.** `Preferences` is app-private storage and everything
   else is UI; the sample's `module.json5` has no `requestPermissions` block and
   the document has no 权限说明 section - consistent.
2. **Create the Preferences instance once, early.**
   `preferences.getPreferencesSync(context, { name: 'ItemsPreferences' })` in
   `UIAbility.onCreate`, held on a module singleton.
3. **Provide the two arrays at the page root** under explicit alias keys, and
   consume them by alias wherever they are needed - the bubble does not need to
   be a direct child.
4. **Load on appear, save on disappear.** Parse the stored JSON into `ToDoItem`
   instances (re-`new` them, do not use the parsed plain objects directly), and
   split by `isDone`.
5. **Bind the popup to a `@State` boolean.**
   `.bindPopup(this.isPopupShow, { builder: this.menuBuilder(), placement:
   Placement.Top, autoCancel: true, radius })` on the bubble `Text`.
6. **Flip it from a long press.**
   `.gesture(LongPressGesture().onAction(() => { this.isPopupShow = true; }))`.
7. **Dispatch the menu selection on a stable identifier**, not on the displayed
   title (HW-05-0088), and set `isPopupShow = false` in every branch so the popup
   closes either way.
8. **Push a fully-formed `ToDoItem`.** Resolve the sender id first
   (`this.item.isSelf ? MYSELF_ID : this.friendId`), then rebuild the avatar
   resource from that id rather than carrying a `Resource` around.
9. **Confirm bulk actions.** `getPromptAction().showDialog({ title, message,
   buttons })` and act only on `data.index === 1`; the sample checks `err` first,
   which is the right shape.
10. **Publish the insets from `TYPE_SYSTEM`**, not `TYPE_CUTOUT` (HW-05-0086),
    and check `err` in the `getMainWindow` callback before dereferencing `data`
    (HW-05-0087).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Long-press popup on the chat bubble

`LaterItems.zip#entry/src/main/ets/component/CustomText.ets`

```ts
@Component
export struct CustomText {
  @ObjectLink item: MsgContent;
  @State isPopupShow: boolean = false;
  @Prop friendId: string;
  @Consume('toDoItems') toDoItems: ToDoItem[];
  menuItems: menuItem[] = MENU_ITEMS;

  build() {
    Text() {
      ForEach(this.item.content, (content: string) => {
        Span(content)
          .fontColor(this.item.isSelf ? $r('app.color.text_color_blue') : $r('app.color.text_color_friends'));
      }, (content: string, index: number) => `${JSON.stringify(content)}_${JSON.stringify(index)}`); // HW-05-0089
    }
    .backgroundImage(this.item.isSelf ? $r('app.media.pop_of_me') : $r('app.media.pop_of_others'))
    .backgroundImageSize({ width: '100%', height: '100%' })
    .bindPopup(this.isPopupShow, {
      builder: this.menuBuilder(),
      placement: Placement.Top,
      autoCancel: true,
      radius: CommonConstants.CUSTOM_TEXT_BIND_POPUP_RADIUS,
    })
    .gesture(
      LongPressGesture()
        .onAction(() => {
          this.isPopupShow = true;
        })
    );
  }
}
```

The `bindPopup` + `LongPressGesture` pair is the reusable core: the popup's
visibility is ordinary `@State`, so the gesture just assigns `true` and
`autoCancel: true` handles dismissal on tap-outside.

### The custom menu and the push into the to-do list

`LaterItems.zip#entry/src/main/ets/component/CustomText.ets`

```ts
@Builder
menuBuilder() {
  Flex({ alignContent: FlexAlign.SpaceAround, justifyContent: FlexAlign.SpaceAround }) {
    ForEach(this.menuItems, (item: menuItem) => {
      Column() {
        Image(item.img).width('20vp').height('20vp').objectFit(ImageFit.Contain);
        Text(item.title)
          .fontColor(item.title === '稍后处理' ? $r('app.color.text_color_blue') :   // HW-05-0088
            $r('app.color.text_color_friends'));
      }
      .onClick(() => {
        if (item.title === '稍后处理') {                                             // HW-05-0088
          let personId = this.item.isSelf ? MYSELF_ID : this.friendId;
          this.toDoItems.push(new ToDoItem(personId,
            $r(GetAvatar.idToImg[personId]), this.item.content.join(''), this.item.time));
        }
        this.isPopupShow = false;
      });
    });
  }
  .width('60%');
}
```

Corrected dispatch:

```ts
// DataModel.ets
export enum MenuAction { QUOTE, LATER, STAR, TRANSLATE }
export interface menuItem { action: MenuAction; title: ResourceStr; img: Resource; }
export const MENU_ITEMS: menuItem[] = [
  { action: MenuAction.QUOTE,     title: $r('app.string.menu_quote'),     img: $r('app.media.icon_pointer') },
  { action: MenuAction.LATER,     title: $r('app.string.menu_later'),     img: $r('app.media.icon_later_process') },
  { action: MenuAction.STAR,      title: $r('app.string.menu_star'),      img: $r('app.media.icon_star') },
  { action: MenuAction.TRANSLATE, title: $r('app.string.menu_translate'), img: $r('app.media.icon_translate') },
];

// CustomText.ets
.onClick(() => {
  if (item.action === MenuAction.LATER) { /* push */ }
  this.isPopupShow = false;
});
```

### Cross-level state and the persistence hooks

`LaterItems.zip#entry/src/main/ets/pages/HomePage.ets`

```ts
@Entry
@Component
struct HomePage {
  @StorageLink('avoidTop') avoidTop: number = 0;
  @Provide('chatPathStack') chatPathStack: NavPathStack = new NavPathStack();
  @Provide('toDoItems') todoItems: ToDoItem[] = [];                 // 稍后处理的消息内容
  @Provide('toDoItems_finished') todoItemsFinished: ToDoItem[] = []; // 已完成的稍后处理的消息内容

  aboutToAppear(): void {
    let toDoItemsString: string = preferenceUtils.getChangeItemsPreferences();
    let items: ToDoItem[] = toDoItemsString.length > 0 ? JSON.parse(toDoItemsString) : [];
    for (let item of items) {
      if (!item.isDone) {
        this.todoItems.push(new ToDoItem(item.personId, item.img, item.content, item.time, item.isDone));
      } else {
        this.todoItemsFinished.push(new ToDoItem(item.personId, item.img, item.content, item.time, item.isDone));
      }
    }
  }

  aboutToDisappear(): void {
    let todoItemsAll: ToDoItem[] = [...this.todoItems, ...this.todoItemsFinished];
    preferenceUtils.saveChangeItemsPreferences(JSON.stringify(todoItemsAll));
  }
}
```

Re-`new`-ing each item rather than using the parsed plain objects is deliberate:
`JSON.parse` returns structurally-shaped objects with no prototype, and ArkUI
observation relies on the class instance.

### The Preferences wrapper

`LaterItems.zip#entry/src/main/ets/database/PreferenceUtils.ets`

```ts
const KEY_APP_LATER_ITEMS = 'ItemsToDoLater';

class PreferenceUtils {
  preference?: preferences.Preferences;

  getItemsPreferences(context: Context) {
    this.preference = preferences.getPreferencesSync(context, { name: 'ItemsPreferences' });
  }

  saveChangeItemsPreferences(items: string) {
    this.preference?.putSync(KEY_APP_LATER_ITEMS, items);
    this.preference?.flush((err: BusinessError) => {
      if (err) {
        hilog.error(0x0000, TAG, 'Failed to flush, code = ' + err.code + ', message = ' + err.message);
      }
      hilog.info(0x0000, TAG, 'succeed in flushing');
    });
  }

  getChangeItemsPreferences() {
    let items: string = '';
    items = this.preference?.getSync(KEY_APP_LATER_ITEMS, '') as string;
    return items;
  }

  isKeyExist(): boolean {                       // always returns false - HW-05-0085
    let isKeyExist: boolean = false;
    this.preference?.has(KEY_APP_LATER_ITEMS).then((isExist: boolean) => {
      isKeyExist = isExist;
    }).catch((err: Error) => {
      hilog.error(0x0000, TAG, 'Has the value failed with err: ' + err);
    });
    return isKeyExist;
  }
}

export default new PreferenceUtils();
```

`saveChangeItemsPreferences` uses the **callback** form of `flush` and checks
`err` - which is the correct shape, and better than the fire-and-forget
`flush()` in OFFICE-01 (HW-05-0006). Corrected `isKeyExist`:

```ts
isKeyExist(): boolean {
  return this.preference?.hasSync(KEY_APP_LATER_ITEMS) ?? false;
}
```

### Bulk action with confirmation

`LaterItems.zip#entry/src/main/ets/component/ItemComp.ets`

```ts
Text(this.isDone ? $r('app.string.all_remove') : $r('app.string.all_done'))
  .onClick(() => {
    this.getUIContext().getPromptAction().showDialog({
      title: this.isDone ? $r('app.string.confirm_remove_title') : $r('app.string.confirm_done_title'),
      message: this.isDone ? $r('app.string.confirm_remove_content') : $r('app.string.confirm_done_content'),
      buttons: [{ text: $r('app.string.cancel'), color: $r('app.color.confirm_or_not') },
        { text: $r('app.string.confirm'), color: $r('app.color.confirm_or_not') }]
    }, (err, data) => {
      if (err) {
        hilog.error(0x0000, TAG, 'showDialog err: ' + err);
        return;
      }
      if (data.index === 1) {
        // 未完成事项全部完成
        if (this.isDone === false) {
          for (let item of this.toDoItems) {
            item.isDone = true;
          }
          this.todoItemsFinished.push(...this.toDoItems);
          this.toDoItems = [];
        } else {
          // 已完成事项全部清除
          this.todoItemsFinished = [];
        }
      }
    });
  });
```

Note `const TAG: string = '';` at the top of this file - the `tag` argument is
mandatory and is what the log is filtered by, so give it a real value such as
`'ItemComp'` before shipping.

### One component, two groups

`LaterItems.zip#entry/src/main/ets/view/LaterProcessView.ets`

```ts
@Component
export struct LaterProcessView {
  @Consume('toDoItems') toDoItems: ToDoItem[];
  @Consume('toDoItems_finished') todoItemsFinished: ToDoItem[];
  isDone: boolean = true;
  notDone: boolean = false;

  build() {
    Column() {
      if (this.toDoItems.length > 0) {
        // 未完成的待办消息
        ItemComp({ isDone: this.notDone });
      }
      if (this.todoItemsFinished.length > 0) {
        // 已完成的待办消息
        ItemComp({ isDone: this.isDone });
      }
    };
  }
}
```

`ItemComp` consumes both arrays itself and uses the `@Prop isDone` flag only to
decide which one it is rendering - so one component serves both groups.

## Permissions & config

`LaterItems.zip#entry/src/main/module.json5` declares **no `requestPermissions`
block**, which is correct: `Preferences` writes into the application's own
private storage and nothing else here touches a protected resource. The document
correspondingly has no 权限说明 section - verified consistent.

The Preferences store is named once, in the wrapper:

```ts
preferences.getPreferencesSync(context, { name: 'ItemsPreferences' });
const KEY_APP_LATER_ITEMS = 'ItemsToDoLater';
```

`build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Persistence is tied to page teardown.** The only save point is
  `HomePage.aboutToDisappear`; the list is not written when an individual item
  changes. If the process ends without the page being destroyed, the most recent
  edits are lost - save on each mutation if that matters.
- **Everything is stored as one JSON string** under a single Preferences key, so
  the whole list is rewritten on every save and there is no partial update.
- **`ToDoItem.img` is a `ResourceStr` that gets serialised.** The avatar is
  rebuilt from `personId` through `GetAvatar.idToImg` when an item is created, so
  `personId` is the durable field; persisting the resolved resource alongside it
  is redundant and should not be relied on across builds.
- **`@Consume` requires a matching `@Provide` above it in the tree.** The two
  arrays are provided at `HomePage`, so any component rendered outside that
  subtree cannot consume them - the alias keys (`'toDoItems'`,
  `'toDoItems_finished'`) are part of the contract.
- **The popup is per-bubble.** Each `CustomText` owns its own `isPopupShow`, so
  long-pressing a second bubble opens a second popup independently; there is no
  single-open-at-a-time coordination.
- **The menu is display-only apart from one entry.** Of the four items in
  `MENU_ITEMS`, only 稍后处理 has behaviour; 引用, 收藏 and 翻译 close the popup and
  do nothing.

## Pitfalls

- **`isKeyExist()` is incorrect**: it returns its local flag synchronously, before
  the `has()` promise can assign it, so it always reports `false`. Use the
  synchronous `hasSync`, matching the `putSync`/`getSync` already used in the same
  class. (HW-05-0085)
- **Reading the top inset from `AvoidAreaType.TYPE_CUTOUT` is incorrect** when the
  comment and the usage both mean the status bar. `TYPE_SYSTEM` "usually indicates
  the status bar area"; `TYPE_CUTOUT` is zero on a device without a notch, leaving
  only the hard-coded `+ 15`. The navigation inset in the same block correctly
  uses `TYPE_NAVIGATION_INDICATOR`. (HW-05-0086)
- **Dereferencing `data` in the `getMainWindow` callback without checking `err` is
  incorrect** - on failure neither `avoidTop` nor `avoidBottom` is published and a
  TypeError is raised inside the callback; and `setWindowLayoutFullScreen`'s
  promise is discarded although the avoid areas are read straight after it.
  (HW-05-0087)
- **Dispatching the menu action on `item.title === '稍后处理'` is incorrect.** The
  only behaviour in the scenario is selected by string-matching a user-visible
  label, so localising the menu - which every other string in the project already
  is - silently disables the feature. Dispatch on an enum or id. (HW-05-0088)
- **`(content, index) => \`${JSON.stringify(content)}_${JSON.stringify(index)}\``
  is incorrect as a key generator**: both values are already primitives, the
  stringify calls are pure overhead, and folding in the index reintroduces the
  index dependency the ForEach guide warns about. Key on `content` directly.
  (HW-05-0089)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/03_application-framework/arkts-modal-transition.md` -
  `bindPopup` and its `builder` / `placement` / `autoCancel` options.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-modal-transition
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-single-gesture.md` -
  `LongPressGesture` and `onAction`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-single-gesture
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `getPreferencesSync`, `putSync`, `getSync`, `has` / `hasSync` and the `flush`
  overloads.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` as the status-bar area versus `TYPE_CUTOUT`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `getMainWindow`, `setWindowLayoutFullScreen` and `getWindowAvoidArea`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  key-generator semantics and the warning against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` -
  the rule against `JSON.stringify` in a key generator.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
