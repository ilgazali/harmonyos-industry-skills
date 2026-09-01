---
id: LIFE-15
title: Two-level ID-type picker - a component-free half-modal reopened in place, with results returned by emitter
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
sample: huawei_industry_tree/02_convenient_life/downloads/IDType4UserRegistration.zip
kits: ["@kit.ArkUI", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.openBindSheet", "UIContext.closeBindSheet", ComponentContent, "ComponentContent.update", "ComponentContent.dispose", wrapBuilder, SheetOptions, "emitter.emit", "emitter.on", "emitter.off", "@Extend", "@Styles", "@Watch", "@State", "@Provide", "@Consume", "@StorageProp", Navigation, NavPathStack, NavDestination, navDestination, Checkbox, CheckBoxShape, focusable, TextInput, InputType, nestedScroll, NestedScrollMode, "display.getDefaultDisplaySync", "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0107, HW-02-0108, HW-02-0109, HW-02-0110, HW-02-0111, HW-02-0112, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when a form field's value comes from a **picker that is too big
for a menu and has more than one level** - "choose your ID document type", where
one of the nine options fans out into four sub-types.

The technique is `UIContext.openBindSheet`, the **component-free** half-modal.
Unlike `.bindSheet()` in `LIFE-08` and `LIFE-14`, it is not attached to any
component: you build a `ComponentContent` from a global `@Builder`, hand it to
the `UIContext`, and open and close it imperatively. That matters here for one
specific reason - **the second level reuses the same sheet**. Close it, call
`contentNode.update()` with the sub-list and a new title, reopen: the user sees
one panel whose contents change, not a stack of two.

The result travels back the other way through `emitter`, because a global
`@Builder` has no component instance to reach the page with.

Take this for document types, country/region pickers, category trees, bank
selection - any two-level choice feeding a single form field. For a one-level
list, `bindSheet` on the field itself is simpler.

## Feature checklist

- A registration form: ID type, ID number, name, phone, province/city, password,
  confirm password, an agreement checkbox and a submit button.
- The ID-type field is not typeable; tapping it (or its chevron) opens the
  picker.
- The picker is a bottom sheet listing nine document types, the current one
  highlighted.
- One row - 境外人员身份证明 (overseas personnel identity proof) - shows a
  disclosure arrow and opens a second-level list of four sub-types in the **same**
  sheet, at a shorter height and with a back arrow in the title.
- The back arrow returns to the first level; picking any leaf closes the sheet
  and fills the field.
- Reopening the picker always starts at the first level.

## Architecture

One `entry` module. The sheet content is a global `@Builder`, so it has no
access to the page - hence the emitter.

```
entry/src/main/ets
├── common/CommonConstants.ets        sizes, sheet heights (595 / 320), event id, AppStorage keys
├── components/IDTypeBindSheet.ets    global @Builder idTypeBindSheetBuilder + row click -> emitter
├── entryability/EntryAbility.ets     full screen, avoid areas -> AppStorage (inside loadContent)
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── NavigationPage.ets            @Entry - Navigation + @Provide NavPathStack
│   └── UserRegistrationPage.ets      THE CARD: the form, the sheet lifecycle, the emitter listener
├── utils
│   ├── DisplayUtil.ets               display size + AppStorage inset readers
│   └── PromptActionClass.ets         static wrapper over openBindSheet / closeBindSheet / emit
└── viewmodel/IDTypeData.ets          IDTypeModel, the two lists, BindSheetParams, event param
```

The documented tree matches the zip exactly.

**One sheet, two levels, one node.** The whole design turns on the fact that
`ComponentContent` can be re-parameterised:

```
contentNode = new ComponentContent(ctx, wrapBuilder(idTypeBindSheetBuilder),
                                   new BindSheetParams(selected, idTypeMainData, false))

open level 1   close -> contentNode.update(BindSheetParams(selected, idTypeMainData, false))
                     -> options.height = 595, title = main       -> open
open level 2   close -> contentNode.update(BindSheetParams(selected, idTypeSubData,  true))
                     -> options.height = 320, title = back+sub   -> open
```

**The return channel is an emitter, not a callback.** `idTypeBindSheetBuilder`
is a module-level function, so it cannot see the page. A row tap packs
`{ isSubBindSheet, selectedIDType, index }` into an `emitter.EventData` on a
fixed event id; the page subscribes in `aboutToAppear` and unsubscribes in
`aboutToDisappear`. That unsubscribe is present and correct - unusual in this
industry.

```
row onClick -> PromptActionClass.sendBindSheetOnClickEvent(param)
            -> emitter.emit({ eventId: 2025062 }, { data: { value: param } })
            -> page's emitter.on handler -> bindSheetOnClick(value)
                 isSubBindSheet     -> commit + close
                 index === 4        -> close, update to sub-list, reopen
                 otherwise          -> commit + close
```

The `index === 4` test is `HW-02-0111`.

## Implementation steps

1. **Model the list as `{ idTypeCode, idTypeStr }`** and give the sub-types codes
   derived from their parent (`04` -> `040`..`043`). That relationship is what
   the index-based branch should have been using.
2. **Write the sheet body as a global `@Builder`** taking a single params class,
   so `wrapBuilder` can accept it. Keep it free of page state.
3. **Create one `ComponentContent`** in the page and drive it through a small
   static helper. Type the helper's statics as optional and guard them properly
   (`HW-02-0110`).
4. **Return the selection through `emitter`,** subscribing in `aboutToAppear`
   and calling `emitter.off` in `aboutToDisappear`.
5. **Sequence close -> update -> open** by awaiting the close (`HW-02-0108`);
   the two calls act on the same node.
6. **Branch on the document code, not the row index** (`HW-02-0111`).
7. **Rebuild the `SheetOptions` for each level** - the height and the title
   builder both change - and re-register them before opening.
8. **Dispose the `ComponentContent` in `aboutToDisappear`** after the close
   resolves (`HW-02-0109`).
9. **Bind the form fields two-way** (`HW-02-0112`) and take the safe-area insets
   with `@StorageProp`, not a bare `AppStorage.get` (`HW-02-0107`).

## Verified snippets

All snippets are from `IDType4UserRegistration.zip`. Corrected forms are marked.

**The sheet body as a global builder - `IDType4UserRegistration.zip#entry/src/main/ets/components/IDTypeBindSheet.ets:39`** (corrected, see `HW-02-0111`)

```typescript
@Builder
export function idTypeBindSheetBuilder(dialogParams: BindSheetParams) {
  Column() {
    List({ space: Constant.BIND_SHEET_LIST_SPACE_NUM_15 }) {
      ForEach(dialogParams.idTypeData, (item: IDTypeModel, index: number) => {
        ListItem() {
          Row() {
            Text(item.idTypeStr)
              .fontColor(item.idTypeCode === dialogParams.selectedIDType.idTypeCode ?
                $r('app.color.selected_id_type_text_color') : $r('app.color.font_color_normal'))
              .bindSheetRowTextStyle();
            Image($r('app.media.right_arrow'))
              .bindSheetRightArrowStyle()
              // FIX: the sample tests index === Constant.BIND_SHEET_JWSFRYZM_INDEX (4)
              .visibility(item.idTypeCode === Constant.ID_TYPE_CODE_JWRYSFZM ?
                Visibility.Visible : Visibility.None);
          }
          .padding($r('app.float.page_padding_10vp'))
          .onClick(() => {
            bindSheetItemOnClick(item, index, dialogParams.isSubBindSheet);
          });
        };
      }, (item: IDTypeModel) => JSON.stringify(item));
    }
    .nestedScroll({
      scrollForward: NestedScrollMode.SELF_FIRST,
      scrollBackward: NestedScrollMode.SELF_FIRST,
    });
  };
}

function bindSheetItemOnClick(item: IDTypeModel, index: number, isSubBindSheet: boolean) {
  PromptActionClass.sendBindSheetOnClickEvent({ isSubBindSheet, selectedIDType: item, index });
}
```

**A global `@Builder` is what `wrapBuilder` needs.** `ComponentContent` takes a
`WrappedBuilder`, and only a module-scope function can be wrapped - a `@Builder`
method on a component cannot. That constraint is the reason for both the
`BindSheetParams` class (all the state has to arrive as one parameter) and the
emitter (there is no `this` to call back on).

**`nestedScroll({ SELF_FIRST, SELF_FIRST })` is the detail that makes the sheet
usable.** A `List` inside a sheet competes with the sheet's own drag gesture;
`SELF_FIRST` makes the list consume the scroll first and hand the remainder to
the sheet, so dragging in the middle of the list scrolls it and dragging past
the end moves the sheet.

**The parameters class - `IDType4UserRegistration.zip#entry/src/main/ets/viewmodel/IDTypeData.ets:81`** (as shipped)

```typescript
export class BindSheetParams {
  selectedIDType: IDTypeModel = { idTypeCode: '', idTypeStr: '' };
  idTypeData: IDTypeModel[] = [];
  isSubBindSheet: boolean = false;

  constructor(selectedIDType: IDTypeModel, idTypeData: IDTypeModel[], isSubBindSheet: boolean) {
    this.selectedIDType = selectedIDType;
    this.idTypeData = idTypeData;
    this.isSubBindSheet = isSubBindSheet;
  }
}

export interface ItemOnClickEventParam {
  isSubBindSheet: boolean;
  selectedIDType: IDTypeModel;
  index: number;
}
```

**Three fields cover both levels.** `idTypeData` swaps the list, `isSubBindSheet`
tells the row handler which level it is on so the page knows whether the tap is a
leaf or a descent, and `selectedIDType` drives the highlight. The same class is
passed to `update()` for level two, which is why one node can serve both.

**The static sheet helper - `IDType4UserRegistration.zip#entry/src/main/ets/utils/PromptActionClass.ets:27`** (corrected, see `HW-02-0108` and `HW-02-0110`)

```typescript
export class PromptActionClass {
  static ctx?: UIContext;                          // FIX: sample declares these non-optional
  static contentNode?: ComponentContent<Object>;   //      and then guards them against null
  static options?: SheetOptions;

  static openBindSheet(): Promise<void> {          // FIX: sample returns void
    if (!PromptActionClass.ctx || !PromptActionClass.contentNode || !PromptActionClass.options) {
      return Promise.resolve();
    }
    return PromptActionClass.ctx.openBindSheet(PromptActionClass.contentNode, PromptActionClass.options);
  }

  static closeBindSheet(): Promise<void> {         // FIX: same
    if (!PromptActionClass.ctx || !PromptActionClass.contentNode) {
      return Promise.resolve();
    }
    return PromptActionClass.ctx.closeBindSheet(PromptActionClass.contentNode);
  }

  static sendBindSheetOnClickEvent(value: ItemOnClickEventParam) {
    emitter.emit({ eventId: Constant.SHEET_ON_CLICK_EVENT_ID }, { data: { value: value } });
  }
}
```

**The shipped guard is `if (PromptActionClass.contentNode !== null)`,** on a
static declared without an initializer - so its value before `setContentNode` is
`undefined`, the test passes, and `openBindSheet` runs against three undefined
arguments. `undefined !== null` is the whole bug.

**Swapping the level in place - `IDType4UserRegistration.zip#entry/src/main/ets/pages/UserRegistrationPage.ets:145`** (corrected, see `HW-02-0108` and `HW-02-0111`)

```typescript
async bindSheetOnClick(value: ItemOnClickEventParam) {
  if (value.isSubBindSheet) {                                   // a leaf in the sub-list
    this.selectedIDType = value.selectedIDType;
    await PromptActionClass.closeBindSheet();
  } else if (value.selectedIDType.idTypeCode === Constant.ID_TYPE_CODE_JWRYSFZM) {   // FIX: was index === 4
    await PromptActionClass.closeBindSheet();                   // FIX: sample does not await
    this.contentNode.update(new BindSheetParams(this.selectedIDType, idTypeSubData, true));
    this.options = {
      height: Constant.SUB_SHEET_HEIGHT_320,                    // shorter: four rows
      title: () => { this.titleBuilder({ isSubSheet: true }); },// with a back arrow
      showClose: true
    };
    PromptActionClass.setOptions(this.options);
    await PromptActionClass.openBindSheet();
  } else {                                                      // a leaf in the main list
    this.selectedIDType = value.selectedIDType;
    await PromptActionClass.closeBindSheet();
  }
}

async openMainSheet() {
  await PromptActionClass.closeBindSheet();                     // FIX: sample does not await
  this.contentNode.update(new BindSheetParams(this.selectedIDType, idTypeMainData, false));
  this.options = {
    height: Constant.MAIN_SHEET_HEIGHT_595,
    title: () => { this.titleBuilder({ isSubSheet: false }); },
    showClose: true
  };
  PromptActionClass.setOptions(this.options);
  await PromptActionClass.openBindSheet();
}
```

**`close -> update -> open` is the whole two-level illusion,** and it only works
if the three steps are ordered. The shipped code issues all three synchronously
against the same `ComponentContent`, so the node's parameters are replaced while
the previous sheet may still be dismissing.

**`openMainSheet` doubles as the back button.** The title builder's left arrow
calls it (`UserRegistrationPage.ets:101-103`), and so does the field's `onClick`
- which is why reopening the picker always lands on level one, as the comment at
`UserRegistrationPage.ets:171` says.

`SheetOptions.title` takes a builder closure, so each level supplies its own -
the sub-level's shows a back arrow and the sub-type's name, the main level's
shows neither.

**Wiring the page - same file, line 126** (as shipped)

```typescript
aboutToAppear(): void {
  PromptActionClass.setContext(this.ctx);
  PromptActionClass.setContentNode(this.contentNode);
  PromptActionClass.setOptions(this.options);

  let eventDialogCloseBtnOnclick: emitter.InnerEvent = { eventId: Constant.SHEET_ON_CLICK_EVENT_ID };
  emitter.on(eventDialogCloseBtnOnclick, (eventData: emitter.EventData) => {
    if (eventData.data && eventData.data.value !== null && typeof eventData.data.value !== 'undefined' &&
      eventData.data.value !== undefined) {
      this.bindSheetOnClick(eventData.data.value);
    }
  });

  this.IDTypeChange();
}

aboutToDisappear(): void {
  PromptActionClass.closeBindSheet();
  emitter.off(Constant.SHEET_ON_CLICK_EVENT_ID);
  // FIX: also this.contentNode.dispose() once the close resolves - HW-02-0109
}
```

**The `emitter.off` is the part most samples in this industry get wrong** and
this one gets right. Note it removes *every* subscriber on that event id, not
just this component's - acceptable with a single listener, but `emitter.off(id,
callback)` is the precise form if a second page ever listens.

**The read-only field - same file, line 204** (as shipped)

```typescript
TextInput({ text: this.idType, placeholder: $r('app.string.placeholder_id_type') })
  .focusable(false)                       // no keyboard, no caret - it is a button
  .textInputStyle()
  .onClick(() => {
    this.idTypeOnClick();
  });
```

**`focusable(false)` on a `TextInput` is the idiom for a picker field:** it keeps
the input's placeholder, font and layout so the row matches the typeable fields
beside it, while making a tap open the sheet instead of the keyboard. The chevron
next to it carries the same `onClick`.

`this.idType` is written by the `@Watch('IDTypeChange')` handler on
`selectedIDType`, so committing a selection updates the field with no explicit
assignment at the call site. The other five inputs have no such path
(`HW-02-0112`).

**Shared styles - same file, line 34 and line 113** (as shipped)

```typescript
@Extend(Text)
function textStyle() {
  .fontWeight(FontWeight.Regular)
  .fontSize($r('app.float.text_font_size_normal_14fp'))
  .fontColor($r('app.color.font_color_normal'));
}

@Styles
rowStyle() {
  .width(Constant.FULL_PERCENT)
  .backgroundColor($r('app.color.background_color_white'))
  .padding($r('app.float.row_padding_15vp'))
  .borderRadius($r('app.float.border_radius_10vp'));
}
```

**`@Extend` and `@Styles` are not interchangeable, and this file shows why.**
`@Extend(Text)` is bound to one component type and may therefore use
`Text`-specific attributes such as `fontWeight`; it must be declared at file
scope. `@Styles` covers only universal attributes but works on any component, and
may be declared inside the struct - which is what lets `rowStyle` be reused
across every form row regardless of what it contains.

## Permissions & config

None. `IDType4UserRegistration.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
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

Root `build-profile.json5` targets `6.0.0(20)`.

`EntryAbility` sets `setWindowLayoutFullScreen(true)` and writes
`topRectHeight` / `bottomRectHeight` into `AppStorage` - but **inside
`loadContent`'s success callback**, i.e. after the page has been built. The page
reads them with a plain `AppStorage.get()` through `DisplayUtil`, so it never
sees them (`HW-02-0107`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 89-91).
- `openBindSheet` needs a `UIContext`, so the page must be mounted before the
  helper can be used - hence the `setContext` in `aboutToAppear`.
- `ComponentContent` requires a **global** `@Builder`; a builder method on a
  component cannot be wrapped.
- The two sheet heights are fixed vp numbers (595 and 320) sized for nine and
  four rows. Adding a document type overflows the first without changing it.
- The parent-child relationship between `04` and `040`..`043` exists only in the
  code numbering; nothing in the model links the two lists.
- `PromptActionClass` is a module-scope singleton holding the `UIContext` and the
  node, so only one such sheet can be open per process and neither reference is
  cleared on teardown.
- The submit, privacy-policy, service-agreement, province/city and back handlers
  are toasts; the sample marks them `For Demo. 正式使用时请填补操作内容，或删掉。`
  ("For Demo. Fill in the operation content for production use, or delete.").

## Pitfalls

- **`HW-02-0107` - the safe-area padding is read with a bare `AppStorage.get()`
  at build time,** while `EntryAbility` writes the values inside `loadContent`'s
  callback. `AppStorage.get` creates no dependency, so the page is padded with 0
  under a full-screen layout and never re-renders. Use `@StorageProp`.
- **`HW-02-0108` - `closeBindSheet()` is not awaited before `update()` and
  `openBindSheet()`,** and all three act on the same `ComponentContent`. Have the
  helper return its promises and sequence them.
- **`HW-02-0109` - the `ComponentContent` is never `dispose()`d,** and
  `PromptActionClass`'s statics keep the node and the `UIContext` reachable after
  the page is gone.
- **`HW-02-0110` - the helper guards uninitialised statics with `!== null`.**
  They hold `undefined`, so the guard always passes. Type them optional and test
  truthiness.
- **`HW-02-0111` - the sub-sheet trigger and the disclosure arrow are keyed by
  array index (4)** rather than by `idTypeCode` `'04'`, in two files. Reordering
  the document-type list moves the behaviour to the wrong row.
- **`HW-02-0112` - five of the six editable fields have no `$$` and no
  `onChange`,** so nothing the user types reaches the `@State` model, and
  `Checkbox().select(false)` re-applies unchecked on every rebuild - so opening
  the picker clears the agreement box.
- **Do not try to wrap a component's `@Builder` method.** `wrapBuilder` needs a
  module-scope function, which is what forces the params class and the emitter.
- **Do not omit `nestedScroll` on a `List` inside a sheet.** Without
  `SELF_FIRST` the sheet steals the drag and the list will not scroll.
- **Do not forget `emitter.off`.** The subscription outlives the page otherwise;
  this sample is one of the few in the industry that releases it.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `openBindSheet`, `closeBindSheet`, `updateBindSheet`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` - `ComponentContent`, `update`, and the `dispose()` leak warning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `SheetOptions`: `height`, `title`, `showClose`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`, `emitter.emit`, `emitter.off`, `InnerEvent`, `EventData`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-guides/03_application-framework/arkts-extend.md` - `@Extend`, file scope and component-specific attributes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-extend
- `documentation/harmonyos-guides/03_application-framework/arkts-style.md` - `@Styles`, universal attributes only, declarable inside a struct
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-style
- `documentation/harmonyos-references/02_application-framework/ts-container-scrollable-common.md` - `nestedScroll`, `NestedScrollMode.SELF_FIRST`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scrollable-common
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-focus.md` - `focusable(false)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-focus
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - `$$` on `TextInput.text` and `Checkbox.select`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `LIFE-08` - the same industry's other `ComponentContent` sheet, opened through `PromptAction` instead of `UIContext`
- `LIFE-14` - the component-attached `bindSheet`, for comparison
- `LIFE-01` - the industry shell this page would sit in
