---
id: OFFICE-29
title: Showing enterprise employee details on the incoming-call screen - CallerInfoQueryExtensionAbility backed by a shared RDB store
industry: 05_office
doc: huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
sample: huawei_industry_tree/05_office/downloads/CallerInfoProviderDemo.zip
kits: ["@kit.CallServiceKit", "@kit.ArkData", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [CallerInfoQueryExtensionAbility, "CallerInfoQueryExtensionAbility.onQueryCallerInfo", CallerInfo, CallerInfoQueryExtensionContext, "relationalStore.getRdbStore", "relationalStore.StoreConfig", "relationalStore.SecurityLevel", "RdbStore.executeSql", "RdbStore.query", "RdbStore.insert", "relationalStore.RdbPredicates", "ResultSet.goToNextRow", "ResultSet.close", "common.Context", UIAbility, "windowStage.loadContent"]
permissions: []
min_api: 15
modules: [entry]
findings: [HW-05-0155, HW-05-0156, HW-05-0157, HW-05-0158, HW-05-0159, HW-05-0160, HW-05-0161, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an **enterprise** app must put a colleague's name, staff
number, department and job title on the system incoming-call screen, so that a
call from an internal number is identified even though the app is not running.

The shape of the solution is unusual and worth internalising:

- The app does **not** observe calls. It registers an **ExtensionAbility of type
  `callerInfoQuery`**, and the system dialer calls into it.
- The extension runs **in its own process, with the app closed**. Whatever it
  needs must already be on disk - here, an RDB store the main ability seeded on
  a previous launch.
- The capability is **gated**: it is available only to enterprise developers,
  requires an approved application in AppGallery Connect, a re-issued debug
  profile, and a user-facing caller-ID switch that must be on.

This is the only card in `05_office` where the entry point is an
`ExtensionAbility` the system starts on its own.

## Feature checklist

The application must be able to:

- Create an RDB store holding the employee directory and seed it.
- Skip re-seeding on later launches.
- Expose an `ExtensionAbility` of type `callerInfoQuery` in `module.json5`.
- Override `onQueryCallerInfo(phoneNumber)` and answer within the dialer's time
  budget.
- Look the incoming number up in the store from the extension's own context.
- Resolve a `CallerInfo` when the number belongs to an employee and reject when
  it does not - and reject **also** when a previous call did match.

## Architecture

Single `entry` HAP, two entry points, one shared database:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | `UIAbility`; creates `RdbTest.db`, creates the `EMPLOYEE` table, seeds one test row |
| `entrycallability/EntryCallAbility.ets` | `CallerInfoQueryExtensionAbility`; overrides `onQueryCallerInfo`, opens the same store and looks the number up |
| `entrybackupability/EntryBackupAbility.ets` | backup extension stub |
| `pages/Index.ets` | a placeholder page - the UI plays no part in the feature |

```
first launch (app open)                    every incoming call (app closed)
-----------------------                    --------------------------------
EntryAbility.onWindowStageCreate           system dialer
  getRdbStore(RdbTest.db)                    |
  executeSql(CREATE TABLE EMPLOYEE)          v
  queryData() -> already seeded?            EntryCallerInfoQueryExtAbility
  insert(one employee row)                    onQueryCallerInfo(phoneNumber)
        |                                       getRdbStore(RdbTest.db)
        v                                       query EMPLOYEE where phoneNumber = ?
   customDir/subCustomDir/RdbTest.db <--------- resolve(CallerInfo) | reject
```

The `EMPLOYEE` table:

```sql
CREATE TABLE IF NOT EXISTS EMPLOYEE (
  ID INTEGER PRIMARY KEY AUTOINCREMENT,
  phoneNumber TEXT NOT NULL,
  contactName TEXT NOT NULL,
  employeeId  TEXT NOT NULL,
  department  TEXT NOT NULL,
  position    TEXT NOT NULL
)
```

`CallerInfo` - the object the dialer renders - has exactly four fields:
`contactName`, `employeeId`, `department`, `position`. The phone number is the
lookup key, not part of the answer.

## Implementation steps

1. **Get the capability granted first.** Apply in AppGallery Connect under
   *Development and services > Project settings > Manage open capabilities*,
   search for "enterprise", and request the enterprise caller-information
   capability with a reason and an attachment. Nothing below works without it.
2. **Enable Push Kit.** Call Service Kit depends on it; the official
   *Preparations* page states "you need to enable Push Kit before enabling Call
   Service Kit". This is step 6 of the document.
3. **Re-apply for the debug profile** after the capability is granted and
   replace it in DevEco Studio.
4. **Create and seed the store from the UIAbility**, awaiting the CREATE TABLE
   before querying or inserting (HW-05-0156). Use the real test number as the
   key, not the shipped `'xxx'` placeholder (HW-05-0160).
5. **Write the extension.** Subclass `CallerInfoQueryExtensionAbility`, cast
   `this.context` to `common.Context` so it can be handed to
   `relationalStore.getRdbStore`, and override `onQueryCallerInfo`.
6. **Keep the result local to the call.** Return what this query found; never
   answer from fields left over by a previous call (HW-05-0155).
7. **Register the extension** in `module.json5` with `"type": "callerInfoQuery"`.
8. **Turn the switch on for debugging**: Phone app > More > Settings > Caller
   ID, then enable the switch for your app. In production, detect this state
   with `queryNumberIdentifySwitchState` / `isSupportEnterpriseNumberIdentify`
   and deep-link to `callsetting://number_identity` (HW-05-0161).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Registering the extension

`CallerInfoProviderDemo.zip#entry/src/main/module.json5`

```json5
"extensionAbilities": [
  {
    "name": "EntryBackupAbility",
    "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
    "type": "backup",
    "exported": false,
    "metadata": [
      { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
    ],
  },
  {
    "name": "EntryCallerInfoQueryExtAbility",
    "srcEntry": "./ets/entrycallability/EntryCallAbility.ets",
    "type": "callerInfoQuery"
  },
]
```

Three fields and nothing else - `name`, `srcEntry`, `type`. This matches the
official registration snippet exactly. `exported` is not set: the system dialer
reaches the extension through the `callerInfoQuery` type, not by an explicit
start.

### The extension

`CallerInfoProviderDemo.zip#entry/src/main/ets/entrycallability/EntryCallAbility.ets`

```ts
import { CallerInfoQueryExtensionAbility, CallerInfo } from '@kit.CallServiceKit';
import { common } from '@kit.AbilityKit';
import { relationalStore } from '@kit.ArkData';
import { hilog } from '@kit.PerformanceAnalysisKit';

const DOMAIN = 0x0000;

export default class EntryCallerInfoQueryExtAbility extends CallerInfoQueryExtensionAbility {
  isSuccess:boolean = false;          // ability state, never reset - HW-05-0155
  contactName: string = '';
  employeeId: string = '';
  department: string = '';
  position: string = '';

  async queryData(phoneNumber: string): Promise<void> {
    // let context = this.context as common.ExtensionContext;
    let context = this.context as common.Context;
    const STORE_CONFIG: relationalStore.StoreConfig = {
      name: 'RdbTest.db',
      securityLevel: relationalStore.SecurityLevel.S3,
      encrypt: false,
      customDir: 'customDir/subCustomDir',
      isReadOnly: false
    };
    let store = await relationalStore.getRdbStore(context, STORE_CONFIG);
    let predicates1 = new relationalStore.RdbPredicates('EMPLOYEE');
    predicates1.equalTo('phoneNumber', phoneNumber);
    let resultSet =
      await store.query(predicates1, ['ID', 'phoneNumber', 'contactName', 'employeeId', 'department', 'position']);
    // resultSet是一个数据集合的游标，默认指向第-1个记录，有效的数据从0开始。
    while (resultSet.goToNextRow()) {
      const id = resultSet.getLong(resultSet.getColumnIndex('ID'));
      this.contactName = resultSet.getString(resultSet.getColumnIndex('contactName'));
      this.employeeId = resultSet.getString(resultSet.getColumnIndex('employeeId'));
      this.department = resultSet.getString(resultSet.getColumnIndex('department'));
      this.position = resultSet.getString(resultSet.getColumnIndex('position'));
      this.isSuccess = true;
    }
    // 释放数据集的内存
    resultSet.close();
  }

  async onQueryCallerInfo(phoneNumber: string): Promise<CallerInfo> {
    try {
      await this.queryData(phoneNumber);
    } catch (error) {
      hilog.error(DOMAIN, 'testTag', `error msg is ${error.message}`);
    }

    return new Promise<CallerInfo>((resolve, reject) => {
      // 在此处实现根据号码查询企业联系人的业务逻辑
      if (this.isSuccess) {
        resolve({
          contactName: this.contactName,
          employeeId: this.employeeId,
          department: this.department,
          position: this.position
        });
      } else {
        reject({
          code: 10001,
          msg: 'query fail'
        });
      }
    });
  }
}
```

Two things are exactly right here and should be copied: `this.context as
common.Context` (the extension context is an `ExtensionContext`, and
`getRdbStore` wants a `Context`), and `resultSet.close()` with the comment
explaining it - the reference warns that an unclosed result set leaks FDs or
memory.

The result state is the problem. Corrected shape, with the answer local to the
call:

```ts
export default class EntryCallerInfoQueryExtAbility extends CallerInfoQueryExtensionAbility {
  private async queryEmployee(phoneNumber: string): Promise<CallerInfo | undefined> {
    const context = this.context as common.Context;
    const store = await relationalStore.getRdbStore(context, STORE_CONFIG);
    const predicates = new relationalStore.RdbPredicates('EMPLOYEE');
    predicates.equalTo('phoneNumber', phoneNumber);
    const resultSet = await store.query(predicates,
      ['contactName', 'employeeId', 'department', 'position']);
    try {
      if (!resultSet.goToNextRow()) {
        return undefined;
      }
      return {
        contactName: resultSet.getString(resultSet.getColumnIndex('contactName')),
        employeeId: resultSet.getString(resultSet.getColumnIndex('employeeId')),
        department: resultSet.getString(resultSet.getColumnIndex('department')),
        position: resultSet.getString(resultSet.getColumnIndex('position'))
      };
    } finally {
      resultSet.close();
    }
  }

  async onQueryCallerInfo(phoneNumber: string): Promise<CallerInfo> {
    const info = await this.queryEmployee(phoneNumber);
    if (!info) {
      hilog.info(DOMAIN, 'testTag', 'no employee matched');
      throw { code: 10001, msg: 'query fail' } as BusinessError;
    }
    return info;
  }
}
```

`goToNextRow()` once, not a `while` loop: the dialer shows a single contact, and
the phone number is the key, so more than one row is a data error rather than a
list to iterate.

### Creating and seeding the store

`CallerInfoProviderDemo.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
const STORE_CONFIG: relationalStore.StoreConfig= {
  name: 'RdbTest.db',
  securityLevel: relationalStore.SecurityLevel.S3,
  encrypt: false,
  customDir: 'customDir/subCustomDir',
  isReadOnly: false
};

const SQL_CREATE_TABLE = 'CREATE TABLE IF NOT EXISTS EMPLOYEE (ID INTEGER PRIMARY KEY AUTOINCREMENT, phoneNumber TEXT NOT NULL, contactName TEXT NOT NULL, employeeId TEXT NOT NULL, department TEXT NOT NULL, position TEXT NOT NULL)';

relationalStore.getRdbStore(this.context, STORE_CONFIG, async (err, store) => {
  if (err) {
    hilog.error(DOMAIN, 'testTag', `Failed to get RdbStore. Code:${err.code}, message:${err.message}`);
    return;
  }

  store.executeSql(SQL_CREATE_TABLE).then(() => {          // not awaited - HW-05-0156
    hilog.info(DOMAIN, 'testTag', 'create employee table success!');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag', `Failed to executeSql. Code:${err.code}, message:${err.message}`);
    return;                                                 // returns from the catch handler only
  });

  await this.queryData(store);

  hilog.info(DOMAIN, 'testTag', 'isQuery: %s', String(this.isQuery));   // <private> - HW-05-0158
  if (this.isQuery) {
    return;                                                 // step 3: skip seeding on later launches
  }

  const valueBucket1: relationalStore.ValuesBucket = {
    'phoneNumber': this.phoneNumber,
    'contactName': '李四',
    'employeeId': '123456',
    'department': '开发者平台',
    'position': '工程师'
  };
  store.insert('EMPLOYEE', valueBucket1, (err: BusinessError, rowId: number) => {
    if (err) {
      hilog.error(DOMAIN, 'testTag', `Failed to insert data. Code:${err.code}, message:${err.message}`);
      return;
    }
    hilog.info(DOMAIN, 'testTag', `Succeeded in inserting data. rowId:${rowId}`);
  });
});
```

Corrected ordering:

```ts
try {
  await store.executeSql(SQL_CREATE_TABLE);
  await this.queryData(store);
} catch (err) {
  hilog.error(DOMAIN, 'testTag', 'Failed to init store. Code:%{public}d, message:%{public}s',
    (err as BusinessError).code, (err as BusinessError).message);
  return;
}
if (this.isQuery) {
  return;
}
await store.insert('EMPLOYEE', valueBucket1);
```

### The number that must be edited

`CallerInfoProviderDemo.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
export default class EntryAbility extends UIAbility {
  isQuery: boolean = false;
  phoneNumber: string = 'xxx'; // 需手动添加电话号码
```

`phoneNumber` is both the seeded record's key and the "have I seeded already?"
probe. Until it is replaced with a real number the sample cannot match any call,
and the document never says so (HW-05-0160).

## Permissions & config

**No permissions.** `CallerInfoProviderDemo.zip#entry/src/main/module.json5` has
no `requestPermissions` array, and none is needed: the app never reads the call
log, the contacts or the telephony state - the dialer pushes the number into the
extension. The Call Service Kit *Preparations* page lists `MICROPHONE` and
`CAMERA`, but both are for VoIP calls, not for caller identification.

What gates this feature instead of permissions:

| Gate | Where |
| --- | --- |
| Enterprise developer account | AppGallery Connect |
| Approved "enterprise caller information" capability | AGC > Manage open capabilities |
| Push Kit enabled | AGC, prerequisite of Call Service Kit |
| Re-issued debug profile | AGC, after the capability is granted |
| Caller-ID switch on for this app | Phone app > More > Settings > Caller ID |

`CallerInfoProviderDemo.zip#build-profile.json5`

```json5
"signingConfig": "CallerInfoProviderDemo",
"compatibleSdkVersion": "5.0.3(15)",
"runtimeOS": "HarmonyOS",
"buildOption": {
  "strictMode": { "caseSensitiveCheck": true, "useNormalizedOHMUrl": true }
}
```

This contradicts the document's 约束与限制, which demands API 20 and the
HarmonyOS 6.0.0 SDK, and there is no `targetSdkVersion` at all (HW-05-0159).

`deviceTypes` is `["phone", "tablet", "2in1"]`; the `CallerInfoQueryExtensionContext`
reference lists Phone, PC/2in1, Tablet and Wearable.

## Constraints

- **Enterprise-only.** The official guide opens with "This function is available
  only for enterprise app developers." A consumer app cannot ship this.
- **Only one contact is ever displayed.** "If the same contact person exists in
  multiple apps, the apps will be sorted lexicographically by app package name,
  and the first result will be displayed." Your bundle name decides whether you
  win.
- **`AbilityStage` is created before `onQueryCallerInfo` runs.** The official
  guide warns: "When the onQueryCallerInfo method is called, the system first
  creates an AbilityStage instance for the app. Therefore, avoid adding overly
  complex or time-consuming logic within AbilityStage to prevent call timeouts."
  Keep `AbilityStage.onCreate` empty - this sample has no `AbilityStage` at all,
  which is the safest form.
- **The extension runs with the app closed.** Everything it needs must be in the
  sandbox already. Opening the RDB store per call is the cost of that; it happens
  on the call path, so keep the query indexed and narrow.
- **`CallerInfo` carries four fields only** - `contactName`, `employeeId`,
  `department`, `position`.
- **Versions.** `CallerInfoQueryExtensionContext` exists since **5.0.2(14)**; the
  document claims API Version 20 / HarmonyOS 6.0.0 SDK / DevEco Studio 6.0.0,
  while the project builds against 5.0.3(15).
- **The switch is the user's.** Until the per-app caller-ID switch is enabled
  nothing calls the extension, and the app is given no signal - query it with
  `queryNumberIdentifySwitchState` and `isSupportEnterpriseNumberIdentify`, and
  deep-link to the fixed URL `callsetting://number_identity` with
  `openLink(link, { appLinkingOnly: false })`.
- **The reference pages for `CallerInfoQueryExtensionAbility` and
  `numberIdentify` are stubs in this repository** (11 lines each), so the exact
  signature of `onQueryCallerInfo`, the shape of the rejection value, and the
  `SwitchState` enum could not be verified against the reference; they are
  documented here from the guide and the sample.
- **Rejection shape is unspecified.** The official sample rejects with a bare
  string (`reject("error reason")`), the sample here with
  `{ code: 10001, msg: 'query fail' }`. Neither is typed; treat the value as
  opaque to the dialer.

## Pitfalls

- **Holding the query result in ExtensionAbility fields is incorrect.**
  `isSuccess`, `contactName`, `employeeId`, `department` and `position` are only
  ever written when a row matches, so a later call from a number that matches
  nothing still resolves - with the previous employee's identity on a stranger's
  incoming call. The official sample deliberately keeps `let isSuccess` local to
  the promise. (HW-05-0155)
- **Not awaiting `executeSql(CREATE TABLE)` before querying the table is
  incorrect.** The create is fired and dropped, the query runs immediately after,
  `queryData` has no try/catch, and its rejection is swallowed by the async
  callback - so a failed race leaves the EMPLOYEE table unseeded with no visible
  error. The `return;` inside the `.catch` returns from the handler, not from the
  callback. (HW-05-0156)
- **Interpolating employee data into the hilog format string is incorrect.** The
  values are baked into the text before hilog sees it, so the privacy identifier
  cannot apply and the log ties a phone number to a named employee's staff
  number, department and title in plaintext. (HW-05-0157)
- **`'%s'` without `{public}` is incorrect** for the `isQuery` / `isSuccess`
  flags - the two diagnostics that say whether the feature worked print as
  `<private>`. (HW-05-0158)
- **The document says the sample needs API Version 20 and the HarmonyOS 6.0.0
  SDK, which is incorrect** for the project as shipped: `build-profile.json5`
  sets `compatibleSdkVersion: "5.0.3(15)"` and declares no `targetSdkVersion`.
  (HW-05-0159)
- **The document says "manually store one employee's information for testing"
  without saying where, which is incorrect guidance** - the reader must replace
  `phoneNumber: string = 'xxx'` in `EntryAbility.ets:26`, otherwise the seeded
  row is keyed to the literal string `'xxx'` and no call can ever match it.
  (HW-05-0160)
- **Telling the user to flip the caller-ID switch by hand and stopping there is
  incomplete.** The page the document links documents
  `queryNumberIdentifySwitchState`, `isSupportEnterpriseNumberIdentify` and the
  `callsetting://number_identity` deep link; without them an app cannot tell that
  it is switched off. (HW-05-0161)

## References

Reference and guide pages used to verify this card:

- `documentation/harmonyos-guides/07_application-services/callservice-enterprise-contact-display.md` -
  the enterprise-only restriction, the single-contact rule, the AGC application
  flow, the debug-profile replacement, the `onQueryCallerInfo` sample with its
  local `isSuccess`, the `module.json5` registration, the AbilityStage timeout
  warning, the two switch-state APIs and the `callsetting://number_identity`
  deep link.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/callservice-enterprise-contact-display
- `documentation/harmonyos-guides/07_application-services/call-preparations.md` -
  "you need to enable Push Kit before enabling Call Service Kit", and the
  MICROPHONE/CAMERA permissions that apply to VoIP rather than to this feature.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/call-preparations
- `documentation/harmonyos-references/06_application-services/callservicekit-callerinfoquery-extension-context.md` -
  `CallerInfoQueryExtensionContext` inherits from `ExtensionContext`, since
  5.0.2(14), stage model only.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/callservicekit-callerinfoquery-extension-context
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-resultset.md` -
  `close()` and the FD/memory leak warning that the sample honours.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-resultset
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - the mandatory
  privacy identifier in `hilog` format strings.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- The reference pages
  `documentation/harmonyos-references/06_application-services/callservicekit-callerinfoquery-extension-ability.md`
  and `callservicekit-numberldentify.md` are 11-line stubs in this repository;
  `onQueryCallerInfo` and the switch-state APIs were verified against the guide
  above instead.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/callservicekit-callerinfoquery-extension-ability
