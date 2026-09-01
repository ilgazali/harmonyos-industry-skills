---
id: SOCIAL-09
title: Contacts list grouped by pinyin initial - AlphabetIndexer sidebar driving a ListItemGroup list
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/09_contracts.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/contracts-0000002289306481
sample: huawei_industry_tree/14_social_communication/downloads/Contacts.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit", "@kit.TelephonyKit"]
apis: [call, common, display, hilog, window]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-14-0020, HW-14-0033, HW-14-0086, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a screen shows **a long alphabetical list of people that a
user scans rather than reads** - a contact book, a group member roster, a
mention picker, a city or airline chooser. The pattern is a `List` of
`ListItemGroup`s, one group per initial, with an `AlphabetIndexer` pinned to
the right edge that scrolls the list, and a scroll handler that pushes the
current group back into the indexer.

The transferable part is the **two-way binding between an indexer and a
scroller, and the flag that stops it looping**. `AlphabetIndexer.onSelect`
scrolls the list; the list's `onScrollIndex` moves the indexer; without a guard
each one keeps re-triggering the other and the highlight fights the finger.
This sample solves it with one boolean and a 200 ms timer, which is the
cheapest correct answer.

The second half of the card is the **hand-off to system apps**: dialling
through `call.makeCall` and opening the SMS composer through an explicit
`Want`. Neither needs a permission, because in both cases the system app owns
the sensitive action and the user confirms it there. That is worth internalising
before reaching for `ohos.permission.PLACE_CALL`.

## Feature checklist

- The contact list opens grouped by the pinyin initial of the surname, one
  sticky-looking header per letter.
- Only letters that actually have contacts appear in the right-hand indexer.
- Tapping a letter in the indexer scrolls the list to the first contact of that
  group.
- Scrolling the list by hand moves the indexer highlight to the group now at
  the top - and the indexer does not fight the scroll while it is animating.
- A popup bubble shows the letter being selected while a finger is on the
  indexer.
- Each row carries an SMS icon and a phone icon; tapping the phone icon opens
  the system dialler with the number pre-filled, tapping the SMS icon opens the
  system messaging app on a conversation with that number.
- Tapping a row pushes a detail page with the avatar, name, phone number and
  the two e-mail addresses, and the same call/SMS pair.
- A footer line reports the total contact count.

## Architecture

One `entry` module, two pages, one component, one model file and one utility.
No data layer beyond a static array.

```
entry/src/main/ets
├── components/TitleComponent.ets   the 44vp title bar with the three-dot menu glyph
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage (already px2vp'd)
├── model/ContactData.ets           ContactInfo + sourceContactList (10 static people)
├── pages/Contacts.ets              @Entry: Navigation host, grouping, the list and the indexer
├── pages/ContactDetail.ets         the NavDestination reached by pushPathByName
└── utils/Phone.ets                 makeCall + sendMessage (Want into com.ohos.mms)
```

The documented 工程目录 matches the zip exactly. (The industry has a
tree-mismatch systematic, `HW-14-0001`, but it covers four other social
samples - `05_image_recognition`, `29_custom_album_style`,
`36_quick_reply_in_chat` and `44_web_socket_client2` - not this one.)

Routing is declared, not imperative: `module.json5` points `routerMap` at
`profile:router_map`, which maps the name `contactDetail` to
`ContactDetailBuilder` in `pages/ContactDetail.ets`. The list therefore pushes
by name and never imports the detail page.

**The design decision worth copying** is that the grouping is a
`Record<string, Array<ContactInfo>>` computed once in `aboutToAppear`, and the
indexer's `arrayValue` is `Object.keys(...)` of that same record. One
derivation feeds both the list structure and the indexer, so a letter can never
appear in the sidebar without a group behind it, and `AlphabetIndexer`'s
selected index is by construction the same index as the `ListItemGroup` at that
position. That identity is what makes `scrollToItemInGroup(index, 0)` a
one-liner. The alternative - a fixed A-Z sidebar with empty letters - needs a
lookup table and a decision about what an empty letter does.

`ContactInfo` objects are passed whole as the navigation parameter, so the
detail page needs no id, no store and no lookup.

## Implementation steps

1. **Sort the source list first, then group it.** Sorting after grouping would
   only order within a group and leave the group order to insertion.
2. **Sort on the pinyin key, not on the raw string** (`HW-14-0033`). The shipped
   comparator compares Chinese names by UTF-16 code point while the grouping
   uses pinyin initials, so the order inside a letter group is wrong whenever
   the two disagree.
3. **Derive the initial from the first character only**
   (`name.substring(0, 1)`), through `pinyin4js.convertToPinyinString(...,
   FIRST_LETTER)`, then `.trim().toUpperCase()` - the library returns a
   lowercase letter with padding.
4. **Feed `AlphabetIndexer` with `Object.keys(contactList).sort()`** so the
   sidebar and the list share one index space.
5. **Bind the indexer to a `ListScroller`** with `scrollToItemInGroup(index, 0)`
   in `onSelect`.
6. **Guard the return path.** Set a private flag before scrolling
   programmatically and clear it on a timer, so `onScrollIndex` does not write
   `letterIndex` back while the indexer-initiated scroll is still running.
7. **Push the whole `ContactInfo` as the navigation parameter** with
   `pushPathByName('contactDetail', item)`.
8. **Read the parameter from the top of the stack, not from index 0**
   (`HW-14-0020`).
9. **Call and text through the system apps**: `call.makeCall(number)` guarded by
   `canIUse`, and `startAbilityForResult` with the `com.ohos.mms` Want. Declare
   no permissions.

## Verified snippets

All snippets are from `Contacts.zip`. Corrected forms are marked.

**Grouping by pinyin initial — `entry/src/main/ets/pages/Contacts.ets`** (as shipped)

```typescript
import { pinyin4js } from '@ohos/pinyin4js';
import { ContactInfo, sourceContactList } from '../model/ContactData';

private contactList: Record<string, Array<ContactInfo>> = {};
lastNameFirstLetters: Array<string> = [];

aboutToAppear(): void {
  this.preProcessContactList();
}

private preProcessContactList() {
  let sortedContactList = sourceContactList.sort((a: ContactInfo, b: ContactInfo) => a.name < b.name ? -1 : 1);
  for (let i = 0; i < sortedContactList.length; i++) {
    const lastName = sortedContactList[i].name.substring(0, 1);
    const firstLetter: string =
      pinyin4js.convertToPinyinString(lastName, '', pinyin4js.FIRST_LETTER).trim().toUpperCase();
    if (!this.contactList[firstLetter]) {
      this.contactList[firstLetter] = [];
    }
    this.contactList[firstLetter].push(sortedContactList[i]);
  }
  this.lastNameFirstLetters = Object.keys(this.contactList).sort();
}
```

**Three details carry this method.** `convertToPinyinString` is called with an
empty separator and the `FIRST_LETTER` mode, so 张 comes back as `z` rather
than `zhang`; the `.trim().toUpperCase()` is not decoration, the library pads
its output. Only the first character of the name is converted, which is the
correct rule for Chinese surnames and the wrong rule for a Latin name - if the
book can hold both, branch on the character range. And `lastNameFirstLetters`
is derived from `Object.keys`, never written by hand, which is what keeps the
sidebar and the list in the same index space.

The comparator is the weak point (`HW-14-0033`): `a.name < b.name` on Chinese
strings is a UTF-16 code-point comparison, and code-point order is not pinyin
order. 白雨欣 sorts after 包鑫 even though *bai* precedes *bao*. Sort on
`pinyin4js.convertToPinyinString(a.name)` instead, so the ordering and the
grouping agree on one key. The same defect is filed against `AddFriends`
(`SOCIAL-15`); the code here is identical.

**The indexer and the loop breaker — same file** (as shipped)

```typescript
private isScrollingBySelect: boolean = false;
listScroller: ListScroller = new ListScroller();
displayClass = display.getDefaultDisplaySync();
@State letterIndex: number = 0;

@Builder
alphabetIndex() {
  AlphabetIndexer({ arrayValue: this.lastNameFirstLetters, selected: this.letterIndex })
    .width(24)
    .height('100%')
    .usingPopup(true)
    .itemSize(16)
    .alignStyle(IndexerAlign.Right)
    .popupPosition({ y: this.displayClass.height / this.displayClass.densityPixels / 2 - 20, x: 10 })
    .onSelect((index: number) => {
      this.isScrollingBySelect = true;
      this.listScroller.scrollToItemInGroup(index, 0);

      setTimeout(() => {
        this.isScrollingBySelect = false;
      }, 200);
    })
    .position({ x: this.displayClass.width / this.displayClass.densityPixels - 25 });
}

// ... inside build(), on the List:
.onScrollIndex((start: number) => {
  // 只有不是由 onSelect 引起的滚动，才更新 letterIndex
  if (!this.isScrollingBySelect) {
    this.letterIndex = start;
  }
})
```

**`selected` is an input, `onSelect` is an output, and they are wired to each
other** - that is a cycle, and the boolean is what cuts it. Without
`isScrollingBySelect`, tapping `W` scrolls the list, the scroll fires
`onScrollIndex` with whatever group is passing under the top edge mid-flight,
that writes `letterIndex`, and the indexer highlight jumps back to a letter the
user did not choose. The 200 ms timer is a heuristic, not a guarantee: it is
long enough for `scrollToItemInGroup` without animation and short enough that a
following manual scroll is not swallowed.

Both `position` and `popupPosition` are computed from
`display.getDefaultDisplaySync()` in vp (`height / densityPixels`), because
`position` takes vp while the display reports px. Note the display is read once
into a field, so a fold or a window resize does not move the sidebar.

**The detail page's parameter — `entry/src/main/ets/pages/ContactDetail.ets`** (corrected, see `HW-14-0020`)

```typescript
@Consume('pageInfos') pageInfos: NavPathStack;
@State contactInfo: ContactInfo = new ContactInfo();

aboutToAppear(): void {
  this.getParams();
}

getParams() {
  // FIX: the sample reads getParamByIndex(0) - always the bottom of the stack
  const infoString = JSON.stringify(this.pageInfos.getParamByIndex(this.pageInfos.size() - 1));
  this.contactInfo = JSON.parse(infoString);
}
```

`getParamByIndex(0)` returns the parameter of the **first** entry pushed onto
the stack, not of the page reading it. With one level of navigation the two
coincide, which is why the sample looks correct; push a second contact detail
(a double tap on two different rows, or a "shared contact" link inside the
detail page) and the top page renders the first contact's data. `size() - 1`
is the top; `getParamByName('contactDetail')` returns an array and is the safer
form when the same destination can appear twice.

The `JSON.stringify`/`JSON.parse` round-trip is not paranoia: the parameter
arrives typed as `ParamType`, and the copy detaches the state object from the
stack entry so `@State` sees a fresh reference.

**Dialling and texting — `entry/src/main/ets/utils/Phone.ets`** (as shipped)

```typescript
import { call } from '@kit.TelephonyKit';
import { common, Want } from '@kit.AbilityKit';

export function makeCall(phoneNumber: string) {
  if (canIUse('SystemCapability.Applications.Contacts')) {
    call.makeCall(phoneNumber);
  }
}

export function sendMessage(uiAbilityContext: common.UIAbilityContext, phoneNumber: string) {
  let params: Array<Record<string, string>> = [{ 'telephone': phoneNumber }];
  let want: Want = {
    bundleName: 'com.ohos.mms',
    abilityName: 'com.ohos.mms.MainAbility',
    parameters: {
      contactObjects: JSON.stringify(params),
      pageFlag: 'conversation'
    }
  };
  uiAbilityContext.startAbilityForResult(want);
}
```

**Neither function needs a permission, and that is the point.** `call.makeCall`
opens the dialler with the number filled in - the user still presses the call
button, so the app never places a call and never needs `PLACE_CALL`. The SMS
side is the same idea done through an explicit `Want`: `com.ohos.mms` is the
system messaging bundle, `pageFlag: 'conversation'` selects the conversation
screen, and `contactObjects` is a **JSON string**, not an object - the
`parameters` map is string-valued, so the array has to be stringified by the
caller.

`canIUse('SystemCapability.Applications.Contacts')` guards devices without a
telephony stack (a tablet or a 2in1, both of which this module declares in
`deviceTypes`). The SMS path has no such guard, so on a device without
`com.ohos.mms` the `startAbilityForResult` rejects; the sample drops that
promise.

## Permissions & config

**None.** The sample declares no `requestPermissions` at all - the dialler and
the messaging app carry the sensitive operations. Copying this file into an app
that *does* hold `ohos.permission.PLACE_CALL` changes nothing here;
`makeCall` still only pre-fills.

`module.json5` declares `routerMap: "$profile:router_map"`, and
`router_map.json` maps `contactDetail` to `ContactDetailBuilder`. The detail
page is never imported by the list page, which is what makes the route
declarative.

## Constraints

- API Version 24 Release or later, DevEco Studio 6.1.1 Release or later.
- The contact list is ten static entries in `ContactData.ets`; there is no
  contacts-database read, so nothing here requires `ohos.permission.READ_CONTACTS`.
  Reading the real address book does, and is a different card.
- `pinyin4js` is a third-party ohpm dependency (`@ohos/pinyin4js`). It handles
  Chinese only - Latin, Cyrillic or Japanese names fall through to whatever the
  library returns for an unmapped character.
- The sidebar position is computed from the display size once, in a field
  initializer, so it does not follow a fold or a window resize.
- `EntryAbility` converts the avoid areas with `px2vp` **before** writing them
  into `AppStorage`, so the pages apply `this.topRectHeight` directly as a
  margin. Most other samples in this industry store raw px and convert at the
  point of use - do not mix the two conventions in one app.
- `windowClass.on('avoidAreaChange', ...)` is registered and never released.

## Pitfalls

- **`HW-14-0020`** (B/low, probable): `ContactDetail.getParams` reads
  `getParamByIndex(0)`, the bottom of the navigation stack, rather than the
  entry it is rendering; a second push shows the first contact's data on the
  top page. Fix: use `getParamByIndex(size() - 1)` or `getParamByName`.
- **`HW-14-0033`** (B/low, confirmed; filed on `SOCIAL-15`, identical code
  here): contacts are sorted by raw UTF-16 code points but grouped by pinyin
  initial, so the order inside a letter group is wrong (白雨欣 after 包鑫).
  Fix: sort on `convertToPinyinString(name)`, the same key the grouping uses.
- **`HW-14-0001`** (E/low, confirmed; systematic): four social project trees in
  the industry list files their zips do not contain. This sample is not one of
  them - its 工程目录 matches - but verify the tree before trusting any other
  social doc's file list.
- Unfiled, worth knowing: `startAbilityForResult` in `sendMessage` returns a
  promise that is dropped, so a device without `com.ohos.mms` fails silently;
  and the `avoidAreaChange` listener in `EntryAbility` is never unregistered.

## References

- `huawei_industry_tree/14_social_communication/docs/09_contracts.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/contracts-0000002289306481
- `documentation/harmonyos-references/02_application-framework/ts-container-alphabet-indexer.md` - `AlphabetIndexer`, `onSelect`, `popupPosition`, `alignStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-alphabet-indexer
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `ListItemGroup`, `onScrollIndex`, `ListScroller.scrollToItemInGroup`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/03_system/js-apis-call.md` - `call.makeCall` and what it does not require
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-call
- `documentation/harmonyos-guides/03_application-framework/want-overview.md` - explicit Want fields
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/want-overview
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-uiabilitycontext.md` - `startAbilityForResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext
- `documentation/harmonyos-guides/04_system/telephony-sms.md` - the `com.ohos.mms` parameter contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/telephony-sms
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-jump.md` - `pushPathByName` and reading parameters off the stack
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-jump
- `SOCIAL-15` - the same pinyin grouping, with the sorting defect filed
