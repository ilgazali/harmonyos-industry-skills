---
id: SOCIAL-15
title: Adding friends from the address book - contact.selectContacts as a permission-free picker, plus a pinyin-grouped friend list
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/15_add_friends.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_friends-0000002283246714
sample: huawei_industry_tree/14_social_communication/downloads/AddFriends.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ContactsKit", "@kit.PerformanceAnalysisKit"]
apis: [contact, hilog, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0033, HW-14-0086, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when your app wants **contacts from the user's address book
without asking for the address book**. `contact.selectContacts` opens a
system-owned picker in which the user chooses who to share; your app receives
only the chosen entries and needs **no** `ohos.permission.READ_CONTACTS`. That
is the whole reason to prefer it: a "find friends" flow that costs the user one
tap instead of a permission dialog they will refuse.

The second half of the card is the receiving screen: an A-Z friend list where
Chinese names are grouped by the pinyin initial of the surname, rendered with
`ListItemGroup` so each letter gets a header and each group's first and last
rows get rounded corners. That grouping code is the reusable part - the same
shape works for any locale-aware index (a contacts app, a city picker, an
emoji-name search).

The trap to know before you copy: **grouping by pinyin while sorting by code
point** puts names in the wrong order inside a group, and it looks like a
rendering bug rather than a sorting bug (`HW-14-0033`).

## Feature checklist

- A 朋友 (friends) home screen listing contacts grouped under A, B, C … headers.
- Each group is one rounded white card: the first row rounds its top corners,
  the last row its bottom corners, rows between are square.
- A static A-Z index strip down the right edge (display only in this sample).
- A plus button in the title opens a small floating 添加朋友 (add friends)
  menu; tapping it pushes the add-friends page.
- The add-friends page shows a search field and a 手机通讯录 (phone contacts)
  row.
- Tapping that row checks `SystemCapability.Applications.Contacts` and then
  opens the system contact picker in multi-select mode.
- Chosen contacts are appended to the list, de-duplicated by contact id, each
  with a rotating placeholder avatar and an 添加 button.
- Cancelling the picker, or a device without the contacts capability, leaves
  the list untouched and logs instead of throwing.

## Architecture

One `entry` module, six ArkTS files, plus one third-party dependency.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── model/
│   ├── ContactData.ets        ContactInfo + a nine-entry static contactList
│   ├── LetterListData.ets     letterList: the 26 index letters
│   └── ListModel.ets          FriendsItem { id?, name? } + four placeholder avatars
└── pages/
    ├── ContactsFriendsPage.ets  @Builder-exported NavDestination: the picker page
    └── MainPage.ets             @Entry: the grouped friend list (234 lines)
```

The documented tree matches the zip; note it correctly omits an
`entrybackupability` folder, because this sample genuinely has none (its
`extensionAbilities` array is empty). Contrast `HW-14-0001`, where four other
social samples document files their zips do not contain.

`oh-package.json5` pulls in `@ohos/pinyin4js` `^2.0.1`. That is the only
external dependency in the project and the only way this sample knows that 白
starts with a B.

**The design decision worth copying** is that the two screens are joined by a
`routerMap` entry rather than an import. `ContactsFriendsPage.ets` exports a
`@Builder` function:

```typescript
@Builder
export function ContactsFriendsBuilder() {
  ContactsFriendsPage();
}
```

and `resources/base/profile/route_map.json` names it:

```json
{ "name": "contactsFriendsPage",
  "pageSourceFile": "src/main/ets/pages/ContactsFriendsPage.ets",
  "buildFunction": "ContactsFriendsBuilder" }
```

so `MainPage` navigates with `this.pathStack.pushPathByName('contactsFriendsPage', null)`
and never imports the page. The destination is loaded lazily, by name, which is
what keeps a growing app's start-up cost flat. Declare `"routerMap":
"$profile:route_map"` in `module.json5` and the `Navigation` resolves the name
for you; no `navDestination` builder switch is needed.

The weak point of the structure is that the two lists are unrelated types.
`MainPage` renders `ContactInfo { name, avatar }` from a static array, while
`ContactsFriendsPage` renders `FriendsItem { id?, name? }` built from the
picker. Nothing carries a picked contact back into the home list - the sample
demonstrates the picker, not the friendship.

## Implementation steps

1. **Declare the destination page in `route_map.json`** and export its
   `@Builder`; navigate by name through a `NavPathStack`.
2. **Guard the picker with `canIUse('SystemCapability.Applications.Contacts')`**
   before calling it - 2in1 and lite devices may not carry the contacts
   capability, and calling into a missing SysCap throws.
3. **Call `contact.selectContacts(options)`** with
   `isMultiSelect: true`; it returns a promise resolving to `contact.Contact[]`.
4. **Declare no contacts permission.** The picker is a separate system
   application; the data comes back through it, not through a read of your own.
5. **De-duplicate on `contact.id`** before appending - the user can run the
   picker twice and choose overlapping sets.
6. **Handle the rejection path** with `.catch`; cancelling the picker is a
   normal outcome, not an error to surface.
7. **Compute the group key with pinyin**:
   `pinyin4js.convertToPinyinString(lastName, '', pinyin4js.FIRST_LETTER)`,
   trimmed and upper-cased. Latin-initial names pass through unchanged.
8. **Sort by the same pinyin key you group by** (`HW-14-0033`). Sorting the raw
   strings is a UTF-16 comparison and disagrees with pinyin order.
9. **Build the grouped list with `ListItemGroup({ header })`,** one group per
   letter, iterating `Object.keys(...).sort()`.
10. **Round only the first and last row of each group** by comparing the index
    against `0` and `length - 1`, so the group reads as a single card.
11. **Apply the window insets with `safeAreaPadding`** on the `Navigation`,
    converting from px with `px2vp` in the binding expression.

## Verified snippets

All snippets are from `AddFriends.zip`. Corrected forms are marked.

**The picker — `entry/src/main/ets/pages/ContactsFriendsPage.ets`** (as shipped)

```typescript
import { contact } from '@kit.ContactsKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { FriendsItem } from '../model/ListModel';
import { hilog } from '@kit.PerformanceAnalysisKit';

// 选择联系人
selectContacts() {
  // 选择联系人时的筛选条件 (是否多选)
  let contactSelectionOptions: contact.ContactSelectionOptions = { isMultiSelect: true };
  // 调用通讯录选择组件，让用户选择需要传入APP的通讯录联系人
  let promise = contact.selectContacts(contactSelectionOptions);
  promise.then((data) => {
    // 用户选择确认之后，会在此处收到回调
    hilog.info(0x0000, 'testTag', `selectContacts success: data->${JSON.stringify(data)}`);
    if (data && data.length > 0) {
      data.forEach((contact: contact.Contact) => {
        let friend: FriendsItem = {
          id: contact.id,
          name: contact.name?.fullName
        };
        let isAdded: boolean = false;
        this.friendsList.forEach((item: FriendsItem) => {
          if (item.id === friend.id) {
            isAdded = true;
          }
        });
        if (!isAdded) {
          this.friendsList.push(friend);
        }
      });
    }
  }).catch((err: BusinessError) => {
    hilog.error(0x0000, 'testTag', `selectContacts fail: err->${JSON.stringify(err)}`);
  });
}

// the entry point, with the capability guard
.onClick(() => {
  let isContactsAvailable = canIUse('SystemCapability.Applications.Contacts');
  if (isContactsAvailable) {
    this.selectContacts();
  } else {
    hilog.info(0x0000, 'ContactsFriendsPage', 'The device does not support the ability to select contacts.');
  }
})
```

**This is the entire reason to use `selectContacts` instead of `contact.queryContacts`.**
The picker UI belongs to the system contacts app; the user chooses which
entries cross the boundary, and your app is handed exactly those. No
`ohos.permission.READ_CONTACTS`, no dialog, no least-privilege argument to
have with a reviewer. `module.json5` in this sample declares no
`requestPermissions` block at all, and that is correct - compare `HW-14-0003`,
where four sibling social samples ship copy-pasted `INTERNET` / `VIBRATE`
declarations and dead `LOCATION` constants that no code uses.

`canIUse` is not optional politeness. `SystemCapability.Applications.Contacts`
is absent on device types that have no telephony stack, and the sample's
`deviceTypes` includes `2in1`. The guard turns a throw into a log line.

`contact.name?.fullName` is optional-chained because `Contact.name` is itself
optional - a contact can exist with only a phone number. `FriendsItem` mirrors
that with `id?` and `name?`, so an anonymous entry renders as an empty row
rather than crashing.

**Grouping by pinyin — `entry/src/main/ets/pages/MainPage.ets`**
(corrected, see `HW-14-0033`)

```typescript
import { pinyin4js } from '@ohos/pinyin4js';
import { ContactInfo, contactList } from '../model/ContactData';

private contactList: Record<string, Array<ContactInfo>> = {};
lastNameFirstLetters: Array<string> = [];

aboutToAppear(): void {
  this.preProcessContactList();
}

// 将通讯类列表中所有联系人按姓氏首字母分组
private preProcessContactList() {
  // FIX: the sample sorts with `a.name < b.name`, a UTF-16 comparison
  const pinyinKey = (name: string): string =>
    pinyin4js.convertToPinyinString(name, '', pinyin4js.WITHOUT_TONE).trim().toUpperCase();
  let sortedContactList = contactList.sort((a: ContactInfo, b: ContactInfo) =>
    pinyinKey(a.name) < pinyinKey(b.name) ? -1 : 1);
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

**Group and order must come from the same key.** The bucket is chosen by
`FIRST_LETTER` pinyin of the surname; the order inside the bucket, in the
shipped code, comes from comparing the Chinese strings directly - which
compares UTF-16 code points. The two disagree: 包鑫 (bao) sorts before 白雨欣
(bai) by code point, so group B lists 包鑫 first even though bai precedes bao.
With nine seeded contacts it is one visibly wrong pair; with a real address
book it is pervasive.

Sorting on the full pinyin (`WITHOUT_TONE`, not just the initial) is what makes
the intra-group order right, because two names in the same bucket differ only
after the first letter. `Object.keys(...).sort()` then orders the buckets
themselves, which is safe - they are already single Latin letters.

`Record<string, Array<ContactInfo>>` rather than a `Map` is deliberate: ArkUI's
`ForEach` iterates the array `lastNameFirstLetters` and indexes into the record,
so the render order is controlled by the sorted key array and not by insertion
order.

**The grouped list — same file** (as shipped)

```typescript
// 分组列表头部
@Builder
letterColumnGroupHeader(letter: string) {
  Text(letter.toUpperCase())
    .fontSize(15)
    .fontColor('rgba(0, 0, 0, 0.6)')
    .width(328)
    .margin({ top: 28, bottom: 8 });
}

// 好友列表
@Builder
friendListBuilder() {
  List() {
    ForEach(this.lastNameFirstLetters, (letter: string) => {
      ListItemGroup({ header: this.letterColumnGroupHeader(letter) }) {
        ForEach(this.contactList[letter], (item: ContactInfo, index: number) => {
          ListItem() {
            this.contactItem(item);
          }
          .height(76)
          .backgroundColor(Color.White)
          .borderRadius({
            topLeft: index === 0 ? 16 : 0,
            topRight: index === 0 ? 16 : 0,
            bottomLeft: index === this.contactList[letter].length - 1 ? 16 : 0,
            bottomRight: index === this.contactList[letter].length - 1 ? 16 : 0,
          });
        });
      }
      .width(320)
      .divider({
        strokeWidth: 0.5,
        color: $r('sys.color.mask_fourth'),
        startMargin: 76,
        endMargin: 12
      });
    });
  }
  .scrollBar(BarState.Off)
}
```

**`ListItemGroup` is what makes the letter header behave.** Nesting a second
`ForEach` inside plain `ListItem`s would render the same pixels but lose the
group semantics - the header would scroll as an ordinary row instead of
belonging to its section, and `divider` would apply to the whole list instead
of within each group. `startMargin: 76` aligns the separator with the text
column, clearing the 40vp avatar and its margin.

The corner rounding is the trick worth stealing: four ternaries on `index`
turn a stack of independent rows into one card per letter, with no wrapper
container and no measurement. The same expression handles a one-row group -
`index === 0` and `index === length - 1` are both true, so all four corners
round.

## Permissions & config

**None declared, and none needed.**

```json5
"requestPermissions"   // absent entirely
"routerMap": "$profile:route_map",
"extensionAbilities": []
```

`contact.selectContacts` runs the picker in the system contacts application, so
the calling app never reads the contacts database. If you later need to *query*
contacts rather than let the user pick them, that is `ohos.permission.READ_CONTACTS`,
a `user_grant` permission with `reason` and `usedScene` - a different and much
heavier conversation with the user. Reach for it only when the picker genuinely
cannot serve the flow.

`deviceTypes` is `phone`, `tablet`, `2in1`; the `canIUse` guard exists for the
tail of that list.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Requires the `@ohos/pinyin4js` package (`^2.0.1`) from OHPM; without it there
  is no grouping.
- `contact.selectContacts` needs `SystemCapability.Applications.Contacts`.
  Guard with `canIUse` on any build that targets 2in1 or wearables.
- Only the *first character* of the name is used for the group key, which is
  right for Chinese surnames and wrong for a name like "Álvarez" or a
  two-character compound surname (欧阳).
- The 添加 (add) button on each picked contact does nothing; picked contacts
  never reach the home list. There is no persistence of any kind.
- The A-Z index strip on the right is display only - the source comments say so
  (`仅作展示`). Wiring it up means `scrollToItemInGroup` on a `ListScroller`.
- Placeholder avatars come from `AVATAR[index % 4]`, so the fifth picked
  contact repeats the first one's face.
- The search field on both pages is decorative; neither has an `onChange`.
- Layout widths are hardcoded (`328`, `320`, `140`, margins of `104` and `130`)
  and were measured against one phone width; on a tablet or a resized 2in1
  window the rows do not fill the screen.

## Pitfalls

- **`HW-14-0033`** (B/low, confirmed): contacts are sorted by raw code points
  (`a.name < b.name`) but grouped by pinyin initial, so ordering inside a letter
  group is wrong wherever the two disagree - 白雨欣 (bai) lists after 包鑫 (bao)
  in group B. Fix: sort on the pinyin key
  (`pinyin4js.convertToPinyinString(name, '', pinyin4js.WITHOUT_TONE)`), the
  same source the grouping uses.

## References

- `documentation/harmonyos-references/03_application-services/js-apis-contact.md` - `contact.selectContacts`, `ContactSelectionOptions`, the `Contact` structure
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-contact
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItemGroup`, `divider`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `Navigation`, `NavPathStack.pushPathByName`, `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - the `Search` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `huawei_industry_tree/14_social_communication/docs/15_add_friends.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_friends-0000002283246714
