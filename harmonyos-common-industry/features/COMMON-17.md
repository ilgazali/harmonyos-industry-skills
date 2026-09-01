---
id: COMMON-17
title: Silent login - remember the signed-in choice in PersistentStorage and keep accounts in an RDB store
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
sample: huawei_industry_tree/19_common_technical_solutions/downloads/SilentLogin.zip
kits: ["@kit.ArkData", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["PersistentStorage.persistProp", "@StorageLink", "@StorageProp", "AppStorage.get", "relationalStore.getRdbStore", "relationalStore.StoreConfig", "relationalStore.SecurityLevel", "RdbStore.executeSql", "RdbStore.insert", "RdbStore.update", "RdbStore.delete", "RdbStore.query", "relationalStore.RdbPredicates", "relationalStore.ValuesBucket", "ResultSet.getColumnIndex", "ResultSet.getString", "ResultSet.getDouble", "ResultSet.goToFirstRow", "ResultSet.goToNextRow", "ResultSet.close", Navigation, NavPathStack, NavDestination, "NavPathStack.pushPathByName", "NavPathStack.replacePathByName", routerMap, Checkbox, TextInput, TextInputController, "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0039, HW-19-0040, HW-19-0041, HW-19-0042, HW-19-0043, HW-19-0044, HW-19-0181, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an application must **stop showing the login page after the
first successful sign-in** - the user ticks "remember me" once and every later
launch goes straight into the app.

Two independent pieces of state make that work: the *account record* (who the
user is, kept in a relational store) and the *silent-login flag* (whether to skip
the login page, kept in `PersistentStorage`).

**Read the security findings before reusing any of this.** The shipped sample
stores passwords as AES ciphertext under a key that is written into the source
file (HW-19-0039), in a store created at the lowest security level
(HW-19-0041). The structure of the feature is sound; the credential handling is
not.

## Feature checklist

The application must:

- Keep the account table in a relational store, created once with a schema and a
  security level appropriate to credentials.
- On login, look the user up; update the record if it exists, insert it if it
  does not.
- Verify the password against the stored value - and store that value as a
  one-way hash, not as reversible ciphertext (HW-19-0039).
- Persist the "remember me" flag with `PersistentStorage.persistProp`, registered
  **before** any component binds the key (HW-19-0044).
- Bind the checkbox both ways: its rendered state comes from the persisted flag,
  and its `onChange` value goes back into it (HW-19-0043).
- Branch on the flag when entering the protected area: skip straight to the
  content when set, route to the login page when not.
- Give each `TextInput` its own controller (HW-19-0042).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | full-screen layout, publishes the two insets and keeps them current, loads `pages/MainPage`, registers the persisted `isRemembered` flag |
| `common/Constants.ets` | layout constants plus `STORE_CONFIG` (store name and security level) and `ACCOUNT_TABLE` (table name, CREATE statement, query columns) |
| `common/AccountInfo.ets` | the `AccountInfo` row class (`id`, `userName`, `password`) and the `AccountTable` descriptor interface |
| `database/Rdb.ets` | generic wrapper over `relationalStore`: `getRdbStore`, `insertData`, `deleteData`, `updateData`, `query` |
| `database/AccountTable.ets` | account-specific layer: predicates, the `AccountInfo` <-> `ValuesBucket` mapping, and result-set decoding |
| `pages/MainPage.ets` | the sports home; owns the `NavPathStack` via `@Provide('pageInfos')` and contains the silent-login branch |
| `pages/LoginPage.ets` | the login form, the credential check and the remember-me checkbox |
| `pages/IndoorRun.ets` | the protected screen reached after login |
| `components/ProgressBar.ets`, `SportsData.ets` | home-screen widgets |
| `resources/base/profile/route_map.json` | routes `LoginPage` and `IndoorRun` |

**The two-layer database wrapper is the reusable part.** `Rdb` knows nothing
about accounts: it takes a table name, a CREATE statement and a column list at
construction, lazily opens the store, creates the table with `executeSql`, and
exposes callback-style CRUD. `AccountTable` supplies the account semantics - the
`id` predicate for update/delete, `generateBucket()` mapping `userName` onto the
`phone` column, and the `ResultSet` decoding loop.

**Control flow.**

1. `EntryAbility` registers the persisted flag and loads `MainPage`.
2. `MainPage`'s start button reads `AppStorage.get<boolean>('isRemembered')`:
   true pushes `IndoorRun` directly, false pushes `LoginPage`.
3. `LoginPage.aboutToAppear` opens the store and loads every account row into
   `accountList`.
4. The login button looks up the typed user name in that list. Found: verify the
   password, `updateData`, then `replacePathByName('IndoorRun')`. Not found:
   `insertData` (the sample treats first login as registration), then
   `replacePathByName('IndoorRun')`.
5. The checkbox writes `isRemembered`, which `PersistentStorage` mirrors to disk.

Note `replacePathByName` rather than `pushPathByName` - the login page is
replaced so the back gesture does not return to it.

**Where the shipped flow actually breaks.** `ACCOUNT_TABLE.columns` lists a
`userName` column the table does not have, so the query in step 3 fails, the
callback never fires, `accountList` stays empty, and step 4 always takes the
insert branch - no password is ever verified (HW-19-0040).

## Implementation steps

1. **Define the store and table.** `StoreConfig` with a name and a security level
   matched to the data (not S1 for credentials - HW-19-0041), and a CREATE
   statement. Keep the query column list in sync with the CREATE statement
   (HW-19-0040).
2. **Write the generic Rdb wrapper**: lazy `getRdbStore`, `executeSql` for the
   CREATE, and callback-style `insert` / `update` / `delete` / `query`.
3. **Write the table-specific layer**: `RdbPredicates` on `id` for update and
   delete, a `ValuesBucket` mapping, and a `ResultSet` decode loop that walks
   `goToFirstRow` / `goToNextRow` and closes the set.
4. **Register the persisted flag before loading content**:
   `PersistentStorage.persistProp('isRemembered', false)` at the top of
   `onWindowStageCreate` (HW-19-0044).
5. **Bind the flag in the login page** with `@StorageLink('isRemembered')` and in
   the checkbox with `.select(this.isRemembered)` +
   `.onChange((value: boolean) => this.isRemembered = value)` (HW-19-0043).
6. **Load the account list in `aboutToAppear`**, chaining
   `getRdbStore` -> `query`.
7. **Implement the login button**: hash the entered password, look up the user
   name, compare hashes on the found path, insert on the not-found path, and
   `replacePathByName` into the protected screen either way.
8. **Branch on the flag at the entry point** to skip the login page.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The silent-login branch

`SilentLogin.zip#SilentLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
@Entry
@Component
struct MainPage {
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  // ...
  .onClick(() => {
    if (AppStorage.get<boolean>('isRemembered')) {
      this.pageInfos.pushPathByName('IndoorRun', null);
    } else {
      this.pageInfos.pushPathByName('LoginPage', null);
    }
  });
}
```

### Registering the persisted flag (as shipped - see HW-19-0044)

`SilentLogin.zip#SilentLogin/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
windowStage.loadContent('pages/MainPage', (err) => {
  if (err.code) {
    hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
    return;
  }
  // 静默登录选项初始化
  PersistentStorage.persistProp('isRemembered', false);   // FIX: move above loadContent
  hilog.info(0x0000, 'testTag', 'Succeeded in loading the content.');
});
```

### The store and table descriptors

`SilentLogin.zip#SilentLogin/entry/src/main/ets/common/Constants.ets`

```ts
static readonly STORE_CONFIG: relationalStore.StoreConfig = {
  name: 'database.db',
  securityLevel: relationalStore.SecurityLevel.S1   // FIX (HW-19-0041): too low for credentials
};
static readonly ACCOUNT_TABLE: AccountTable = {
  tableName: 'accountTable',
  sqlCreate: 'CREATE TABLE IF NOT EXISTS accountTable (id INTEGER PRIMARY KEY AUTOINCREMENT, phone TEXT NOT NULL, password TEXT NOT NULL)',
  columns: ['id', 'userName', 'password']          // FIX (HW-19-0040): 'phone', not 'userName'
};
```

### The generic RDB wrapper

`SilentLogin.zip#SilentLogin/entry/src/main/ets/database/Rdb.ets`

```ts
export default class Rdb {
  private rdbStore: relationalStore.RdbStore | null = null;
  private tableName: string;
  private sqlCreateTable: string;
  private columns: Array<string>;

  getRdbStore(context: UIContext, callback: Function = () => {}) {
    if (this.rdbStore !== null) {
      callback();
      return;
    }
    let contact = context.getHostContext() as Context;
    relationalStore.getRdbStore(contact, Constants.STORE_CONFIG, (err, rdb) => {
      if (err) {
        hilog.error(0x0000, 'testTag', 'Failed to get RdbStore: %{public}s', JSON.stringify(err));
        return;
      }
      this.rdbStore = rdb;
      this.rdbStore.executeSql(this.sqlCreateTable);
      callback();
    });
  }

  query(predicates: relationalStore.RdbPredicates, callback: Function = () => {}) {
    if (this.rdbStore) {
      this.rdbStore.query(predicates, this.columns, (err, resultSet) => {
        if (err) {
          hilog.error(0x0000, 'testTag', 'Failed to query database: %{public}s', JSON.stringify(err));
          return;
        }
        callback(resultSet);
        resultSet.close();
      });
    }
  }
}
```

### The account layer

`SilentLogin.zip#SilentLogin/entry/src/main/ets/database/AccountTable.ets`

```ts
query(callback: Function) {
  let predicates = new relationalStore.RdbPredicates(Constants.ACCOUNT_TABLE.tableName);
  this.accountTable.query(predicates, (resultSet: relationalStore.ResultSet) => {
    let count: number = resultSet.rowCount;
    if (count === 0 || typeof count === 'string') {
      callback([]);
    } else {
      resultSet.goToFirstRow();
      const result: AccountInfo[] = [];
      for (let i = 0; i < count; i++) {
        let tmp: AccountInfo = { id: 0, userName: '', password: '' };
        tmp.id = resultSet.getDouble(resultSet.getColumnIndex('id'));
        tmp.userName = resultSet.getString(resultSet.getColumnIndex('phone'));
        tmp.password = resultSet.getString(resultSet.getColumnIndex('password'));
        result[i] = tmp;
        resultSet.goToNextRow();
      }
      callback(result);
    }
  });
}

function generateBucket(account: AccountInfo): relationalStore.ValuesBucket {
  let obj: relationalStore.ValuesBucket = {};
  obj.phone = account.userName;
  obj.password = account.password;
  return obj;
}
```

### The login handler (as shipped - see HW-19-0039)

`SilentLogin.zip#SilentLogin/entry/src/main/ets/pages/LoginPage.ets`

```ts
@State keys: string = 'CjZwwvt13DJTTOCD0/z1cw=='; // 密钥   // FIX (HW-19-0039): hardcoded key

Button($r('app.string.login'))
  .enabled(this.userName !== '' && this.password !== '')
  .onClick(async () => {
    this.encryptionPwd = CryptoJS.AES.encrypt(this.password, this.keys).toString();
    let accountInfo = new AccountInfo();
    accountInfo.userName = this.userName;
    accountInfo.password = this.encryptionPwd;

    let index: number = this.accountList.findIndex(v => v.userName === this.userName);
    if (index > -1) {
      // 更新已存储的用户信息
      let meta: AccountInfo = this.accountList.find(v => v.userName === this.userName) as AccountInfo;
      if (this.password === CryptoJS.AES.decrypt(meta.password, this.keys).toString(CryptoJS.enc.Utf8)) {
        accountInfo.id = meta.id;
        this.accountTable.updateData(accountInfo, (success: boolean) => {
          this.accountList.splice(index, Constants.ONE, accountInfo);
          this.uiContext.getPromptAction().showToast({ message: $r('app.string.success') });
          this.pageInfos.replacePathByName('IndoorRun', null);
        });
      } else {
        this.uiContext.getPromptAction().showToast({ message: $r('app.string.fail') });
      }
    } else {
      this.accountTable.insertData(accountInfo, (id: number) => {
        accountInfo.id = id;
        this.accountList.push(accountInfo);
        this.uiContext.getPromptAction().showToast({ message: $r('app.string.register_success') });
        this.pageInfos.replacePathByName('IndoorRun', null);
      });
    }
  });
```

### The remember-me checkbox (as shipped - see HW-19-0043)

`SilentLogin.zip#SilentLogin/entry/src/main/ets/pages/LoginPage.ets`

```ts
@StorageLink('isRemembered') isRemembered: boolean = false;

Checkbox({ name: 'checkbox', group: 'checkboxGroup' })
  .size({ width: Constants.CHECKBOX_SIZE, height: Constants.CHECKBOX_SIZE })
  .select(false)                       // FIX: .select(this.isRemembered)
  .selectedColor(Constants.FONT_COLOR_BLUE)
  .shape(CheckBoxShape.ROUNDED_SQUARE)
  .onChange(() => {
    this.isRemembered = !this.isRemembered;   // FIX: onChange((value: boolean) => this.isRemembered = value)
  });
```

### Route table

`SilentLogin.zip#SilentLogin/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    { "name": "LoginPage", "pageSourceFile": "src/main/ets/pages/LoginPage.ets", "buildFunction": "LoginPageBuilder" },
    { "name": "IndoorRun", "pageSourceFile": "src/main/ets/pages/IndoorRun.ets", "buildFunction": "IndoorRunBuilder" }
  ]
}
```

## Permissions & config

**No permissions are declared and none are needed** - the relational store lives
in the application sandbox and nothing leaves the device.

`SilentLogin.zip#SilentLogin/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]`, `"routerMap": "$profile:route_map"`,
the single `EntryAbility` with the home skill, and an `EntryBackupAbility`
backup extension. There is no `requestPermissions` block.

The only dependency is declared at project level,
`SilentLogin.zip#SilentLogin/oh-package.json5`:

```json5
{
  "modelVersion": "6.0.0",
  "dependencies": {
    "@ohos/crypto-js": "^2.0.4"
  }
}
```

Note that `@ohos/crypto-js` is a third-party JavaScript crypto library. For
credential handling prefer the platform's own Crypto Architecture Kit and
Universal Keystore Kit, which can keep key material out of the application
process entirely.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The store security level cannot be lowered later**: "You cannot change the
  security level of an RDB store from a higher level to a lower one." Choose it
  before the first release.
- **The security level also governs cross-device sync**: "To perform data sync
  operations, the RDB store security level must be lower than or equal to that of
  the peer device."
- **`persistProp` must precede any access to the key**, or the previous run's
  value is overwritten by the AppStorage default.
- **RDB query column lists are validated against the schema** - a name that is
  not a column makes the query fail rather than returning null for it.
- **A single data record cannot exceed 2 MB**, per the `query` reference, or the
  `ResultSet` getters may fail.
- **The password entry screen cannot be screen-recorded** - the document states
  "输入密码界面无法录制。" ("The password input screen cannot be recorded.")
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The AES key is a literal in the login page, which is incorrect.**
  `'CjZwwvt13DJTTOCD0/z1cw=='` ships with the app, so the reversible ciphertext
  in the database is recoverable by anyone who reads the package. Store a salted
  one-way hash instead, and keep any real key in the system key store.
  (HW-19-0039)
- **`columns: ['id', 'userName', 'password']` is incorrect.** The table's columns
  are `id`, `phone`, `password`; the query therefore fails, the callback never
  runs, `accountList` stays empty and every login is treated as a new
  registration - so the password check the document describes never executes.
  Use `['id', 'phone', 'password']`. (HW-19-0040)
- **`SecurityLevel.S1` for a credential store is incorrect.** S1 is documented for
  "non-sensitive system data such as wallpapers"; use S3 or above, and decide
  before release because the level cannot be lowered. (HW-19-0041)
- **The password field is bound to `controller1`, the user-name field's
  controller, which is incorrect** - `controller2` is declared and unused.
  (HW-19-0042)
- **`.select(false)` with a negating `onChange` is incorrect.** The checkbox never
  reflects the persisted preference, so a returning user sees it unticked and
  ticking it turns silent login *off*. Bind `.select(this.isRemembered)` and take
  the new value from the callback argument. (HW-19-0043)
- **`persistProp` inside the `loadContent` callback is the ordering the guide
  names as incorrect**, even though nothing in the shipped code binds the key
  early enough to be bitten by it yet. (HW-19-0044)
- **`Rdb.query`'s error branch returns without calling the callback.** A failed
  query is indistinguishable from "no rows" at the call site - which is what makes
  HW-19-0040 silent. Report failures through the callback.
- **First login is registration.** The sample inserts an account for any unknown
  user name with no verification step; that is demo behaviour, not a sign-up
  flow.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-rdbstore.md` -
  `query(predicates, columns, callback)` ("Columns to query"), `insert`,
  `update`, `delete`, `executeSql`, error codes, and the 2 MB record limit.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-rdbstore
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-e.md` -
  the `SecurityLevel` enumeration, its S1/S2/S3 descriptions, the
  cannot-be-lowered rule and the sync note.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-e
- `documentation/harmonyos-guides/03_application-framework/arkts-persiststorage.md` -
  the persistProp ordering requirement and the "Accessing a Property in
  AppStorage Before PersistentStorage" section.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-persiststorage
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` -
  `@StorageLink` / `@StorageProp` and the AppStorage-overwrites-PersistentStorage
  note.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-appstorage
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/data-persistence-by-rdb-store -
  the RDB persistence guide the document points to.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-checkbox -
  `Checkbox`, its `select` attribute and `onChange` signature.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
