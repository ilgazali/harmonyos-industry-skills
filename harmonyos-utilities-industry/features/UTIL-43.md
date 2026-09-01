---
id: UTIL-43
title: Local persistence side by side - the same user list over a SingleKVStore and over an RDB table
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/43_educate-v1_1-ts_23.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/educate-v1_1-ts_23-0000002309706602
sample: huawei_industry_tree/15_utilities/downloads/LocalDatabase.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [common, distributedKVStore, hilog, relationalStore, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0009, HW-15-0088, HW-15-0089, HW-15-0090, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when you have to choose between the key-value store and the
relational store** for local persistence, and want the two written against the
same feature so the difference is visible rather than argued. The sample builds
one screen - a searchable user list with add and swipe-to-delete - twice: once
on `distributedKVStore.SingleKVStore`, once on `relationalStore.RdbStore`,
switched by a segmented control at the top.

The rule the sample demonstrates: **KV when the record is a key and a value and
nothing queries across records; RDB when anything needs a predicate, a join or
a constraint.** The KV page's search can only fetch an exact key, and listing
everything needs an empty `Query` and a result-set walk; the RDB page's search
is `predicates.equalTo('name', value)` and its uniqueness is
`UNIQUE(name)` in the DDL. That asymmetry is the whole lesson.

The doc for this page is published under the slug `educate-v1_1-ts_23`.
**It has nothing to do with education**: the doc title is 使用本地数据库 (using
a local database), the zip is `LocalDatabase.zip`, and every screenshot is the
user list. Do not chase the slug into the education industry tree.

## Feature checklist

- A top segmented control animating between 键值型数据库 (KV) and 关系型数据库
  (RDB) pages with a 300 ms `animateTo`.
- Each page: a `Search` box, an add button opening a bottom sheet, and a list
  of stored records.
- KV page stores name → id pairs; the sheet has two fields (name, id).
- RDB page stores names only; the sheet has one field.
- Submitting an empty search restores the full list; a non-empty one shows
  matches.
- Swiping a row left reveals a red 删除 button that deletes the record from
  the store and the list.
- The sheet grows by the soft-keyboard height, tracked live through an
  `avoidAreaChange` listener for `TYPE_KEYBOARD`.
- Both stores are created on page entry; the KV store is closed on exit.

## Architecture

One `entry` module. Two sibling page components, one host, one constants file.
No repository or model layer - each page talks to its store directly.

```
entry/src/main/ets
├── constant/Constants.ets            colors, percentages, spacings, sheet heights
├── entryability/EntryAbility.ets
├── entrybackupability/
└── pages
    ├── MainPage.ets                  @Entry: the KV/RDB switch, renders one child at a time
    ├── KvStorePage.ets               KVManager -> SingleKVStore: put / delete / get / getResultSet
    └── RdbStorePage.ets              getRdbStore -> executeSql DDL, insert / delete / query
```

The documented 工程目录 has the directory as `constants`; the zip ships
`constant` (singular), and it also draws `MainPage.ets` and its two siblings at
inconsistent tree depths. The file set itself matches.

**The design decision worth copying** is `MainPage` rendering the two pages
with a plain `if` rather than a `Tabs`:

```typescript
if (this.isKvStore) {
  KvStorePage();
} else {
  RdbStorePage();
}
```

Each page owns its store in `aboutToAppear` and (for KV) releases it in
`aboutToDisappear`, and the `if` **destroys** the component that goes away, so
the store lifecycle follows the switch exactly. A `Tabs` would keep both alive
and both stores open. When a child owns an expensive resource, the conditional
render is the cheaper structure.

The decision **worth avoiding** is that the two pages share no abstraction at
all - the search bar, the sheet, the list and the swipe action are copy-pasted
with small divergences (`this.userInfos` is a `UserInfo[]` on one page and a
`Set<string>` on the other; the delete method is `deleteUser` on one and
`deletUser` on the other). For a comparison demo that is defensible. In a real
app the store belongs behind one interface and the list is written once.

## Implementation steps

1. **Derive the KV bundle name from the app**, never a literal - the shipped
   `'com.example.datatest'` does not match the app's actual
   `com.example.localdatabase` and every call fails on a real install
   (`HW-15-0089`).
2. **Open the KV store with `createIfMissing: true` and an explicit
   `securityLevel`.** `S1` is right for non-sensitive demo data;
   `kvStoreType: SINGLE_VERSION` is the single-device store.
3. **Close every result set.** `getResultSet` allocates from a pool capped at
   eight; walk it, then `closeResultSet` in a `finally` (`HW-15-0088`).
4. **`await` the DDL before the first query.** `getRdbStore`'s callback fires
   before `executeSql` completes, so a fresh install can query a table that
   does not exist yet (`HW-15-0090`).
5. **Check the `err` argument of `getRdbStore`** before assigning `rdb` -
   the sample assigns it unconditionally and would dereference `undefined`.
6. **Handle the insert's promise.** `UNIQUE(name)` makes a duplicate name
   reject; the shipped code neither awaits nor catches, so the row is refused
   while the list shows it as added (`HW-15-0090`).
7. **Track the keyboard height** with `window.getLastWindow(...).on('avoidAreaChange')`
   filtered to `TYPE_KEYBOARD`, and add it to the sheet's `detents` so the
   fields stay visible.
8. **Reference `Constants.KVSTORE_SHEET_HEIGHT`** - the doc's snippet spells it
   `KVSORE_SHEET_HEIGHT` and will not compile (`HW-15-0009`).

## Verified snippets

All snippets are from `LocalDatabase.zip`. Corrected forms are marked.

**Opening the KV store - `entry/src/main/ets/pages/KvStorePage.ets`** (corrected, see `HW-15-0089`)

```typescript
initKVManager() {
  const kvManagerConfig: distributedKVStore.KVManagerConfig = {
    context: this.context,
    bundleName: this.context.abilityInfo.bundleName   // FIX: was the literal 'com.example.datatest'
  };
  try {
    this.kvManager = distributedKVStore.createKVManager(kvManagerConfig);
    const options: distributedKVStore.Options = {
      createIfMissing: true,
      encrypt: false,
      backup: false,
      autoSync: false,
      kvStoreType: distributedKVStore.KVStoreType.SINGLE_VERSION,
      securityLevel: distributedKVStore.SecurityLevel.S1
    };
    this.kvManager.getKVStore<distributedKVStore.SingleKVStore>('storeId', options, (err, store) => {
      if (err) {
        console.error(`Failed to get KVStore: Code:${err.code}, message:${err.message}`);
        return;
      }
      this.kvStore = store;
      this.getAllUsers();
    });
  } catch (e) {
    let error = e as BusinessError;
    console.error(`Failed to create KVManager. Code:${error.code}, message:${error.message}`);
  }
}
```

**`bundleName` in `KVManagerConfig` is not a label, it is an identity check.**
The store lives under the calling app's data directory and the framework
validates that the name belongs to the caller; the shipped
`'com.example.datatest'` is not this app (`AppScope/app.json5` declares
`com.example.localdatabase`), so `createKVManager` fails with an invalid-args
error on a real install. The doc reproduces the wrong name verbatim, so
copying either source reproduces the bug (`HW-15-0089`). The same literal is
passed again to `closeKVStore` in `aboutToDisappear`, so the close fails too.

`securityLevel` is mandatory and permanent - it is fixed when the store is
created and cannot be raised later without deleting the store. `autoSync:
false` keeps this a local store despite the API being called *distributed*
KV; that is the flag that decides it.

**Listing everything out of a KV store - same file** (corrected, see `HW-15-0088`)

```typescript
getAllUsers() {
  // 创建空查询
  let query = new distributedKVStore.Query();

  this.kvStore?.getResultSet(query, (err, resultSet) => {
    if (err) {
      console.error(`Get result set failed. Code: ${err.code}, Message: ${err.message}`);
      return;
    }
    if (resultSet === null) {
      return;
    }
    try {
      this.userInfos = [];
      resultSet.moveToFirst();
      while (!resultSet.isAfterLast()) {
        let entry = resultSet.getEntry();
        this.userInfos.push(new UserInfo(entry.key, entry.value.value.toString()));
        resultSet.moveToNext();
      }
    } finally {
      this.kvStore?.closeResultSet(resultSet);   // FIX: absent in the sample
    }
  });
}
```

**An empty `Query` is how you say "everything" to a KV store** - there is no
`getAll`. The walk is the classic cursor triple: `moveToFirst`,
`isAfterLast`, `moveToNext`, with `getEntry()` returning `{ key, value }` where
`value` is itself a `{ type, value }` wrapper (hence the double
`entry.value.value`).

The missing `closeResultSet` is the defect that only shows up in use: a
`SingleKVStore` allows **eight** open result sets at a time, and every empty
search on this page opens another. The ninth call fails, the list stops
refreshing, and nothing in the UI says why (`HW-15-0088`). Closing in a
`finally` is the only safe form, because the loop body can throw on a
malformed entry.

Note also what this page cannot do: `getUsers(value)` is a `kvStore.get(key)`,
an exact-key lookup dressed as a search. Typing a prefix returns nothing. The
RDB page's equivalent is a predicate, and could as easily be a `LIKE`.

**RDB initialisation - `entry/src/main/ets/pages/RdbStorePage.ets`** (corrected, see `HW-15-0090`)

```typescript
async initDatabase(): Promise<void> {
  const STORE_CONFIG: relationalStore.StoreConfig = {
    name: 'database.db',
    securityLevel: relationalStore.SecurityLevel.S1
  };
  try {
    // FIX: the sample uses the callback form, ignores err, and does not await the DDL
    this.rdbStore = await relationalStore.getRdbStore(this.context, STORE_CONFIG);
    await this.rdbStore.executeSql(
      'CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, UNIQUE(name))');
    this.queryAllUser();
  } catch (e) {
    const error = e as BusinessError;
    console.error(`Failed to init RDB. Code:${error.code}, message:${error.message}`);
  }
}

insertUser() {
  if (this.userName === '') {
    return;                                      // FIX: the sample accepts empty names
  }
  const value: relationalStore.ValuesBucket = { 'name': this.userName };
  const name = this.userName;
  this.rdbStore?.insert('users', value)
    .then(() => {
      this.userInfos.add(name);                  // FIX: the sample adds before the insert resolves
      this.userName = '';
    })
    .catch((err: BusinessError) => {             // UNIQUE(name) rejects duplicates here
      console.error(`Failed to insert. Code:${err.code}, message:${err.message}`);
    });
}
```

**Three things go wrong in the shipped six lines.** `getRdbStore`'s callback
gives you `(err, rdb)`; the sample ignores `err` and assigns `rdb` blindly.
`executeSql` returns a promise that is not awaited, so `queryAllUser()` runs
concurrently with the `CREATE TABLE` - on a fresh install the first query can
fail with *no such table*, and the list comes up empty until the next launch.
And `insert` is fire-and-forget while `this.userInfos.add(...)` runs
unconditionally, so a name that violates `UNIQUE(name)` appears in the list and
vanishes on the next `queryAllUser` (`HW-15-0090`).

The RDB page does get one thing right that the KV page does not:
`queryAllUser` and `queryUser` both call `resultSet.close()` after the
`goToNextRow` walk. The same discipline applied to the KV page's result sets
would have prevented `HW-15-0088`.

**The keyboard-aware sheet - both pages** (as shipped, with the constant the doc misspells)

```typescript
Image($r('app.media.add'))
  .onClick(() => {
    this.isShowSheet = !this.isShowSheet;
  })
  .bindSheet($$this.isShowSheet, this.SheetBuilder(), {
    detents: [Constants.KVSTORE_SHEET_HEIGHT + this.keyHeight],   // 250 + keyboard, in vp
    preferType: SheetType.BOTTOM,
    title: { title: $r('app.string.Add_User') }
  })

aboutToAppear(): void {
  this.initKVManager();
  window.getLastWindow(this.context).then(currentWindow => {
    currentWindow.on('avoidAreaChange', data => {
      if (data.type === window.AvoidAreaType.TYPE_KEYBOARD) {
        this.keyHeight = this.getUIContext().px2vp(data.area.bottomRect.height);
      }
    });
  });
}
```

**`$$this.isShowSheet` is the two-way binding that makes the sheet dismissable.**
With a one-way `this.isShowSheet`, a swipe-down closes the sheet visually and
leaves the flag `true`, so the add button stops working - the same class of bug
as an unhandled popup `onStateChange`. `detents` takes an array of heights; a
single computed entry pins the sheet to exactly the space the form plus the
keyboard needs, which is why the height must be recomputed from
`avoidAreaChange` rather than guessed.

`Constants.KVSTORE_SHEET_HEIGHT` is 250 and `RDBSTORE_SHEET_HEIGHT` is 200 -
the RDB sheet has one field fewer. **The doc's snippet writes
`Constants.KVSORE_SHEET_HEIGHT`**, which does not exist in `Constants.ets` and
will not compile (`HW-15-0009`).

The `avoidAreaChange` listener is registered on both pages and never released;
switching between KV and RDB repeatedly stacks listeners on the same window.

## Permissions & config

**None.** Neither store needs a permission: both live inside the app sandbox,
and `autoSync: false` keeps the KV store off the distributed bus (which is what
would require `ohos.permission.DISTRIBUTED_DATASYNC`).

Configuration that matters instead:

- `KVManagerConfig.bundleName` must be the caller's real bundle name
  (`HW-15-0089`).
- `StoreConfig.securityLevel` is mandatory for both stores and immutable after
  creation. `S1` here; user data usually warrants `S2` or higher.
- The RDB file is `database.db` under the app's database directory; the KV
  store id is the literal `'storeId'`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The KV page's "search" is an exact-key `get`, not a prefix or fuzzy match;
  a miss produces a silent empty list.
- Neither store is deleted anywhere. `closeKVStore` runs on the KV page's
  `aboutToDisappear` (with the wrong bundle name); `RdbStorePage` never closes
  its store at all.
- The RDB page keeps its list in a `Set<string>`, so the `id` column is
  selected, logged and thrown away - the UI shows names only.
- `Constants.DELETE_BUTTON_COLOR` (`'#E84026'`) is passed to `.width()` on both
  sheet columns - a colour string used as a width. ArkUI accepts the string
  and the layout is whatever that parses to; it should be a percentage.
- Both pages hold `getUIContext()?.getHostContext() as common.UIAbilityContext`
  in a field initialiser, which runs before the component is attached.

## Pitfalls

- **`HW-15-0009`** (E/low, confirmed): the doc's snippet writes
  `Constants.KVSORE_SHEET_HEIGHT`; the zip defines `KVSTORE_SHEET_HEIGHT`.
  Copying the documented code fails to compile. Fix: correct the identifier in
  the doc.
- **`HW-15-0088`** (B/medium, confirmed): `getAllUsers` walks a result set and
  never calls `closeResultSet`. A `SingleKVStore` permits eight open result
  sets; every empty search opens one, so after eight refreshes `getResultSet`
  starts failing and the list silently stops updating. Fix: close it in a
  `finally`.
- **`HW-15-0089`** (B/medium, confirmed): `KVManagerConfig.bundleName` is the
  literal `'com.example.datatest'` while the app is `com.example.localdatabase`,
  so `createKVManager` (and the matching `closeKVStore`) fail with invalid-args
  on a real install. The doc repeats the wrong name. Fix: use the app's own
  bundle name, ideally derived from the context.
- **`HW-15-0090`** (B/medium, confirmed): `initDatabase` ignores `getRdbStore`'s
  `err`, does not await `executeSql`, and calls `queryAllUser()` immediately -
  a first-run race against *no such table*. `insertUser` neither awaits nor
  catches an insert that `UNIQUE(name)` will reject, and updates the list
  unconditionally, so a duplicate appears to succeed until the next refresh;
  empty names are accepted too. Fix: await the DDL, then/catch the insert,
  and validate the input.
- **Leaked window listeners** — both pages register `avoidAreaChange` in
  `aboutToAppear` and neither calls `off`. Toggling the KV/RDB switch
  repeatedly stacks handlers, each writing `keyHeight`.
- **Doc slug** — published as `educate-v1_1-ts_23`, an education slug on a
  local-database page. Search by title (使用本地数据库) or by the zip name.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-distributedkvstore.md` - `KVManagerConfig`, `Options`, `getKVStore`, `getResultSet`, `closeResultSet`, the eight-result-set limit
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-distributedkvstore
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore.md` - `StoreConfig`, `getRdbStore`, `executeSql`, `ValuesBucket`, `RdbPredicates`, `ResultSet.close`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `detents`, `SheetType`, and the `$$` two-way binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
