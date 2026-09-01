---
id: LIFE-10
title: Password vault - Asset Store Kit with biometric-gated read, using the preQuery / userAuth / query / postQuery sequence
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
sample: huawei_industry_tree/02_convenient_life/downloads/AssetVerification.zip
kits: ["@kit.AssetStoreKit", "@kit.UserAuthenticationKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["asset.addSync", "asset.removeSync", "asset.preQuerySync", "asset.querySync", "asset.postQuerySync", "asset.AssetMap", "asset.Tag", "asset.AuthType", "asset.ReturnType", "userAuth.getUserAuthInstance", "userAuth.AuthParam", "userAuth.WidgetParam", "userAuth.UserAuthType", "userAuth.AuthTrustLevel", "UserAuthInstance.on('result')", "UserAuthInstance.off('result')", "UserAuthInstance.start", "util.TextEncoder", "util.TextDecoder", Navigation, NavPathStack, NavDestination, pushDestinationByName, routerMap, "@Provide", "@Consume", "@StorageLink", "@State", Tabs, TabsController]
permissions: ["ohos.permission.STORE_PERSISTENT_DATA", "ohos.permission.ACCESS_BIOMETRIC"]
min_api: 20
modules: [entry]
findings: [HW-02-0068, HW-02-0069, HW-02-0070, HW-02-0071, HW-02-0072, HW-02-0073, HW-02-0074, HW-02-0075, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when the application has to **store a short secret on the device
and require the user to prove who they are before it can be read back** -
passwords, card numbers, ID numbers, recovery codes.

Asset Store Kit is the right home for that data: it is a system service, the
secret never lives in your sandbox, and the access-control policy travels with
the record. The part that needs care is the read path, which is not one call but
a four-step handshake:

```
preQuerySync   -> challenge          open a session, get a nonce
userAuth       -> authToken          the user proves identity against that nonce
querySync      -> the secret         hand back challenge + token together
postQuerySync                        close the session
```

Miss the last step and the session leaks; that is the sample's main defect
(`HW-02-0068`).

Take this card for any credential vault, and for the narrower case of "read this
one stored value, but only after a fingerprint". For data that is not short and
not sensitive, use a preferences store or a database instead - Asset Store Kit
is for secrets under 1 KB.

## Feature checklist

- Three tabs: save, delete, query.
- Save writes an alias / secret / label triple, persisted across reinstall, and
  reports "already exists" separately from a generic failure.
- Delete removes by alias and reports "does not exist" separately.
- Query asks for an alias, opens an authentication session, raises the system
  fingerprint prompt, and only then reads the record.
- The retrieved alias, password and label are shown on a pushed
  `NavDestination`.
- Every outcome is a toast; there are no silent failures on the write paths.

## Architecture

One `entry` module. The whole Asset Store surface is behind one static class.

```
entry/src/main/ets
├── constants/Constants.ets       PERIOD 600, SCREEN_LOCK_PASSWORD, sizes
├── entryability/EntryAbility.ets full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── utils/Asset.ets               THE CARD: add / remove / preQuery / auth / query / postQuery
├── view
│   ├── Save.ets                  SavePage  - alias, secret, label
│   ├── Delete.ets                DeletePage - alias
│   └── Query.ets                 CheckPage - alias + the four-step read
└── pages
    ├── MainPage.ets              @Entry - Navigation + Tabs, owns @Provide pageInfo
    └── QueryResult.ets           the pushed result NavDestination
resources/base/profile/route_map.json   name "Query" -> queryBuilder
```

The documented tree matches the zip exactly.

**`asset.AssetMap` is a `Map<Tag, Value>`, and every string must be bytes.**
That is why `Asset.stringToArray` / `arrayToString` exist - `TextEncoder`
in, `TextDecoder` out - and why `convertAssetList` has to decode each tag it
finds before the UI can show it.

**Two different kinds of tag go into the map.** Some describe the record
(`ALIAS`, `SECRET`, `DATA_LABEL_NORMAL_1`, `IS_PERSISTENT`); others describe the
*policy* (`AUTH_TYPE`, `AUTH_VALIDITY_PERIOD`) or carry the *proof*
(`AUTH_CHALLENGE`, `AUTH_TOKEN`). The policy tags are set at write time and are
what forces the handshake at read time.

**Navigation is routed by name through `routerMap`,** not by a `pages` entry:
`MainPage` owns `@Provide pageInfo: NavPathStack`, `CheckPage` takes it with
`@Consume`, and `pushDestinationByName('Query', params, true)` resolves through
`route_map.json` to `queryBuilder`. The result page reads its parameter back in
`onReady` - and also, wrongly, in a field initializer (`HW-02-0072`).

## Implementation steps

1. **Convert every string to `Uint8Array` at the boundary.** Build one
   `stringToArray` / `arrayToString` pair and never pass a raw string into an
   `AssetMap`.
2. **On write, set the access policy with the record:** `AUTH_TYPE = ANY` makes
   the record readable only after authentication, and `IS_PERSISTENT = true`
   keeps it across an uninstall. The persistence flag is what needs
   `ohos.permission.STORE_PERSISTENT_DATA`.
3. **Branch on the asset error codes** rather than showing one message:
   `24000003` is "already exists" on add, `24000002` is "not found" on remove and
   query.
4. **On read, start with `preQuerySync`** and pass `AUTH_VALIDITY_PERIOD` so the
   token stays usable for the whole sequence. It returns the challenge.
5. **Authenticate against that challenge** with
   `userAuth.getUserAuthInstance(authParam, widgetParam)` and `start()`.
   **Handle every result code**, not just success and not-enrolled
   (`HW-02-0069`), and **`off('result')` inside the handler** (`HW-02-0070`).
6. **Query with the challenge and the token together** -
   `AUTH_CHALLENGE` + `AUTH_TOKEN` + `RETURN_TYPE = ALL` - or the service refuses
   to return the secret.
7. **Call `postQuerySync` on every path**, in a `finally` (`HW-02-0068`). The
   reference: "This API must be used with `asset.preQuerySync` together."
8. **Return absence explicitly** from the query helper and navigate only when
   there is a record (`HW-02-0071`).
9. **Declare both permissions** in `module.json5`. Neither needs a runtime
   request - they are system-granted - but both must be declared.

## Verified snippets

All snippets are from `AssetVerification.zip`. Corrected forms are marked.

**Bytes in, bytes out - `AssetVerification.zip#entry/src/main/ets/utils/Asset.ets:36`** (as shipped)

```typescript
static stringToArray(str: string): Uint8Array {
  let textEncoder = new util.TextEncoder;
  return textEncoder.encodeInto(str);
}

static arrayToString(arr: Uint8Array): string {
  let textDecoder = util.TextDecoder.create('utf-8', { ignoreBOM: true });
  return textDecoder.decodeToString(arr, { stream: false });
}
```

**`AssetMap` values are `Uint8Array | number | boolean`,** never strings, so
these two functions are on every path in and out of the service. `ignoreBOM` and
`{ stream: false }` are the same defensive pair as in `LIFE-07`'s rawfile
reader.

**Writing a record - same file, line 51** (as shipped)

```typescript
static async addAsset(account: string, password: string, label: string, context: UIContext) {
  let attr: asset.AssetMap = new Map();
  attr.set(asset.Tag.ALIAS, Asset.stringToArray(account));               // the key
  attr.set(asset.Tag.SECRET, Asset.stringToArray(password));             // the protected value
  attr.set(asset.Tag.DATA_LABEL_NORMAL_1, Asset.stringToArray(label));   // free-form, not protected
  attr.set(asset.Tag.AUTH_TYPE, asset.AuthType.ANY);                     // <- read requires auth
  attr.set(asset.Tag.IS_PERSISTENT, true);                               // survives uninstall
  try {
    asset.addSync(attr);
    Asset.promptAction($r('app.string.save_success'), context);
  } catch (error) {
    if (error.code === 24000003) {                                       // duplicate alias
      Asset.promptAction($r('app.string.exist'), context);
    } else {
      Asset.promptAction($r('app.string.save_failed'), context);
    }
  }
}
```

**`AUTH_TYPE = ANY` at write time is what creates the whole read handshake.**
Without it, `querySync` would return the secret directly and `preQuery`/`userAuth`
/`postQuery` would be unnecessary. The policy lives on the record, not in the
application, which is the point of the kit.

**`DATA_LABEL_NORMAL_1` is not protected.** It is one of a set of label slots
that can be queried without authentication and used for filtering - the right
place for a display name, the wrong place for anything sensitive.

**`IS_PERSISTENT: true` is what `ohos.permission.STORE_PERSISTENT_DATA` is
for.** Drop the permission and this call fails; drop the flag and the record
disappears with the application.

The `24000003` / `24000002` split - "already exists" on add, "not found" on
remove - is the sample's best habit and worth copying to every asset call.

**The read handshake, step 1 - same file, line 130** (corrected, see `HW-02-0074`)

```typescript
static async preQueryAssetPromise(checkAccount: string): Promise<Uint8Array> {
  let assetMap: asset.AssetMap = new Map();
  assetMap.set(asset.Tag.AUTH_VALIDITY_PERIOD, Constants.PERIOD);   // 600 s
  assetMap.set(asset.Tag.AUTH_TYPE, asset.AuthType.ANY);
  if (checkAccount.length > 0) {
    assetMap.set(asset.Tag.ALIAS, Asset.stringToArray(checkAccount));
  }
  try {
    return asset.preQuerySync(assetMap);          // returns the challenge
  } catch (error) {
    hilog.error(0x00, 'Asset', 'preQuerySync failed: %{public}s', JSON.stringify(error));  // FIX
    return new Uint8Array(0);                     // the sample logs nothing here
  }
}
```

**`AUTH_VALIDITY_PERIOD` is how long the token stays good after authentication,**
which has to cover the gap between the fingerprint prompt closing and
`querySync` running. Omit it and the token is single-use with no slack.

The alias is optional here on purpose - a pre-query with no alias opens a session
over every matching record, which is what an account-less bulk query would need
(`HW-02-0073`).

**Step 2, the authentication - same file, line 83** (corrected, see `HW-02-0069` and `HW-02-0070`)

```typescript
static async getAuthToken(challenge: Uint8Array, context: UIContext,
  callback: (isSuccess: boolean, challenge: Uint8Array) => void) {
  let authParam: userAuth.AuthParam = {
    challenge: challenge,                                   // binds this auth to this session
    authType: [userAuth.UserAuthType.FINGERPRINT],
    authTrustLevel: userAuth.AuthTrustLevel.ATL1,
  };
  let widgetParam: userAuth.WidgetParam = { title: Constants.SCREEN_LOCK_PASSWORD };
  try {
    const instance = userAuth.getUserAuthInstance(authParam, widgetParam);
    const handler: userAuth.IAuthCallback = {
      onResult: (result) => {
        instance.off('result', handler);                    // FIX: the sample never unsubscribes
        if (result.result === userAuth.UserAuthResultCode.SUCCESS) {
          callback(true, result.token);
        } else {                                            // FIX: the sample handles only 12500010
          Asset.promptAction($r('app.string.certification'), context);
          callback(false, new Uint8Array(0));
        }
      }
    };
    instance.on('result', handler);
    instance.start();
  } catch (error) {
    Asset.promptAction($r('app.string.certification'), context);
    callback(false, new Uint8Array(0));
  }
}
```

**The `challenge` is the link between the two services.** Asset Store issued it;
UserIAM signs it; `querySync` checks that the token it is handed was issued for
the challenge that opened the session. Passing a stale or fabricated challenge
fails at the query, not at the authentication.

**The shipped handler covers only `12500000` and `12500010`.** The documented set
also includes `12500001` FAIL, `12500003` CANCELED and `12500011` switched to
custom auth - each of which falls through with no callback, so the caller's
cleanup and navigation never run and the button appears dead.

`AuthTrustLevel.ATL1` is the weakest level; a real vault should ask for ATL3 and
list `PIN` alongside `FINGERPRINT` so a user without an enrolled fingerprint can
still get in.

**Step 3, the query - same file, line 163** (corrected, see `HW-02-0071`)

```typescript
static async queryAuthAssetPromise(checkAccount: string, context: UIContext,
  challenge: Uint8Array, authToken: Uint8Array): Promise<Map<string, string> | undefined> {
  let attr: asset.AssetMap = new Map();
  if (checkAccount.length > 0) {
    attr.set(asset.Tag.ALIAS, Asset.stringToArray(checkAccount));
    attr.set(asset.Tag.RETURN_TYPE, asset.ReturnType.ALL);   // include SECRET, not just attributes
    attr.set(asset.Tag.AUTH_CHALLENGE, challenge);
    attr.set(asset.Tag.AUTH_TOKEN, authToken);
  }
  try {
    let data: Array<asset.AssetMap> = asset.querySync(attr);
    const accountList = Asset.convertAssetList(data, []);
    return accountList.length > 0 ? accountList[0] : undefined;   // FIX: sample returns [0] blindly
  } catch (error) {
    if (error.code === 24000002) {
      Asset.promptAction($r('app.string.no_result'), context);
    } else {
      Asset.promptAction($r('app.string.query_failed'), context);
    }
    return undefined;                                             // FIX: sample returns an empty Map
  }
}
```

**`RETURN_TYPE = ALL` is what makes `SECRET` come back.** The default returns
only the attributes; the secret is withheld unless you ask for it *and* present
a valid challenge/token pair. That is three tags that must all be present
together.

**Decoding the result - same file, line 145** (as shipped)

```typescript
static convertAssetList(assetList: asset.AssetMap[], accountList: Map<string, string>[]) {
  for (let account of assetList) {
    let map: Map<string, string> = new Map<string, string>();
    map.set('alias', Asset.arrayToString(account.get(asset.Tag.ALIAS) as Uint8Array));
    if (account.has(asset.Tag.SECRET)) {                       // may be absent - RETURN_TYPE
      map.set('secret', Asset.arrayToString(account.get(asset.Tag.SECRET) as Uint8Array));
    }
    if (account.has(asset.Tag.DATA_LABEL_NORMAL_1)) {
      map.set('label', Asset.arrayToString(account.get(asset.Tag.DATA_LABEL_NORMAL_1) as Uint8Array));
    }
    accountList.push(map);
  }
  return accountList;
}
```

**The `has()` guards are correct and necessary** - `SECRET` is present only when
`RETURN_TYPE` was `ALL` and the token checked out, and a label slot that was
never written is simply absent. `ALIAS` is unguarded because every record has
one.

**The full sequence at the call site - `AssetVerification.zip#entry/src/main/ets/view/Query.ets:118`** (corrected, see `HW-02-0068`, `HW-02-0071`, `HW-02-0073`)

```typescript
.onClick(async () => {
  if (this.checkAccount.length === 0) {
    Asset.promptAction($r('app.string.account_prompt'), this.context);   // FIX: sample does nothing
    return;
  }
  const challenge = await Asset.preQueryAssetPromise(this.checkAccount);
  if (challenge.length === 0) {
    Asset.promptAction($r('app.string.query_failed'), this.context);
    return;                                        // nothing to close - preQuery failed
  }
  this.challenge = challenge;

  Asset.getAuthToken(this.challenge, this.context, async (isSuccess: boolean, authToken: Uint8Array) => {
    try {
      if (!isSuccess) {
        return;
      }
      this.authToken = authToken;
      const account = await Asset.queryAuthAssetPromise(
        this.checkAccount, this.context, this.challenge, this.authToken);
      if (!account) {
        return;                                    // FIX: sample navigates regardless
      }
      this.pageInfo.pushDestinationByName('Query',
        Asset.convertMapArrayToMapItem([account]), true);
    } finally {
      Asset.postQueryAssetPromise(this.challenge); // FIX: sample calls this only on success
    }
  });
})
```

**`postQuerySync` belongs in a `finally`.** Once `preQuerySync` has returned a
challenge, the session exists; cancelling the fingerprint prompt is the common
case and it must still close. The shipped code puts the call inside
`if (isSuccess)`, so every cancelled query orphans a session.

The `challenge.length === 0` early return is the one place where **not** calling
`postQuery` is right: `preQuerySync` threw, so no session was opened.

**`pushDestinationByName` returns a promise** that rejects when the name is not
in the route map. The sample drops it; chaining a `.catch` that logs would make
a `route_map.json` typo visible instead of silent.

**Routing by name - `AssetVerification.zip#entry/src/main/resources/base/profile/route_map.json`** (as shipped)

```json
{
  "routerMap": [
    {
      "name": "Query",
      "pageSourceFile": "src/main/ets/pages/QueryResult.ets",
      "buildFunction": "queryBuilder"
    }
  ]
}
```

with `"routerMap": "$profile:route_map"` in `module.json5` and, in
`QueryResult.ets`:

```typescript
@Builder
export function queryBuilder() {
  Query();
}
```

**This is the declarative alternative to `LIFE-01`'s hand-rolled router.** The
system resolves the name to the file and the builder, and the target page is not
compiled into the caller - so it loads on demand with no `import()` and no
registry of your own. Prefer it whenever the destinations live in the same
module.

## Permissions & config

`AssetVerification.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "routerMap": "$profile:route_map",
    "abilities": [{ /* EntryAbility, entity.system.home */ }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }],
    "requestPermissions": [
      { "name": "ohos.permission.STORE_PERSISTENT_DATA" },
      { "name": "ohos.permission.ACCESS_BIOMETRIC" }
    ]
  }
}
```

Both permissions are declaration-only:

- `ohos.permission.STORE_PERSISTENT_DATA` - required by
  `asset.Tag.IS_PERSISTENT: true`. Without it `addSync` fails.
- `ohos.permission.ACCESS_BIOMETRIC` - required by
  `userAuth.getUserAuthInstance`.

Neither triggers a runtime dialog, so there is no `requestPermissionsFromUser`
call anywhere in the sample and none is needed. The document lists both
correctly at lines 154-155.

Root `build-profile.json5` targets `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 148-150).
- **A fingerprint (or another enrolled credential) must be set on the device.**
  The document says so at line 24: 使用生物识别功能需要设置指纹密码 ("using the
  biometric feature requires a fingerprint password to be set"). Without one,
  `userAuth` returns `12500010` and the read path cannot complete.
- Asset Store Kit is for **short** secrets - the kit's own overview caps them at
  1 KB. Longer data belongs in HUKS or an encrypted database.
- The sample authenticates with `FINGERPRINT` only at `ATL1`. A device with a
  PIN but no fingerprint cannot read anything back.
- `AUTH_VALIDITY_PERIOD` is 600 seconds here; the token is not usable outside
  that window.
- `ALIAS` is the primary key: `addSync` rejects a duplicate with `24000003`, and
  there is no update path in the sample - only add and remove.
- The query is exact-match only. The account-less bulk query the code is
  structured for is never wired up (`HW-02-0073`).
- The retrieved password is rendered as plain `Text` on the result page. That is
  the feature, but it means the value is on screen and in the component tree
  until the page is popped.

## Pitfalls

- **`HW-02-0068` - `postQuerySync` runs only when authentication succeeds.** The
  reference: "This API must be used with `asset.preQuerySync` together." Cancel
  the fingerprint prompt and the session is orphaned. Put it in a `finally`.
- **`HW-02-0069` - the auth handler covers two result codes out of five.**
  `12500001` FAIL, `12500003` CANCELED and `12500011` fall through with no
  callback, so the caller never cleans up and the button looks dead. Use an
  `else`, and use the `userAuth.UserAuthResultCode` enum rather than literals.
- **`HW-02-0070` - `on('result')` has no `off('result')`,** and the
  `UserAuthInstance` is a local that is dropped - so the subscription can never
  be released, because the reference requires the *same* instance to
  unsubscribe.
- **`HW-02-0071` - a failed query still navigates.** The helper returns
  `accountList[0]` (undefined on an empty result) or an empty `Map` from its
  catch, and the caller pushes the result page unconditionally, showing three
  blank fields that read as "this password is empty".
- **`HW-02-0072` - the result page reads its parameter from a `NavPathStack` it
  just constructed,** in a field initializer, and casts the `undefined` away.
  Start with `[]` and fill in `onReady`.
- **`HW-02-0073` - the query button does nothing when the alias is blank** - no
  query, no message - even though both helpers already support the account-less
  form.
- **`HW-02-0074` - `preQueryAssetPromise` swallows its error** and returns an
  empty array with no log, so "no such asset" and "the call failed" are
  indistinguishable, unlike every sibling helper in the same file.
- **`HW-02-0075` - the avoid areas are read once,** with no `avoidAreaChange`
  subscription, under `setWindowLayoutFullScreen(true)`.
- **Do not pass strings into an `AssetMap`.** Every value is
  `Uint8Array | number | boolean`.
- **Do not forget `RETURN_TYPE = ALL`.** Without it `querySync` returns the
  attributes and withholds `SECRET`, and the failure looks like a missing
  record rather than a missing tag.
- **Do not reuse a challenge across sessions.** It is issued by `preQuerySync`
  and invalidated by `postQuerySync`; a second query needs a second handshake.
- **Do not declare `IS_PERSISTENT` without
  `ohos.permission.STORE_PERSISTENT_DATA`.** The add call fails outright.

## References

- `documentation/harmonyos-references/03_system/js-apis-asset.md` - `addSync`, `removeSync`, `preQuerySync` ("After the user authentication is successful, call asset.querySync and asset.postQuerySync"), `querySync`, `postQuerySync` ("must be used with asset.preQuerySync together"), `Tag`, `AuthType`, `ReturnType`, and the 24000xxx error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-asset
- `documentation/harmonyos-references/03_system/js-apis-useriam-userauth.md` - `getUserAuthInstance`, `AuthParam`, `WidgetParam`, `on`/`off('result')`, and the `UserAuthResultCode` values 12500000 / 12500001 / 12500003 / 12500010 / 12500011
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-useriam-userauth
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.STORE_PERSISTENT_DATA`, `ohos.permission.ACCESS_BIOMETRIC`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` - the `routerMap` tag
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack.pushDestinationByName`, `getParamByIndex`, `NavDestination.onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `TextEncoder.encodeInto`, `TextDecoder.decodeToString`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-01` - the same industry's framework sample, whose Asset Store module is dead code (`HW-02-0010`) and whose login hardcodes its key (`HW-02-0007`); this card is what that one should have done
