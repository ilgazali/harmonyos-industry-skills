---
id: SHOP-05
title: One coupon per device - gate the claim on a Device Security device token and a server-side mark bit
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/05_get_coupons.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966
sample: huawei_industry_tree/16_shopping/downloads/GetCoupons.zip
kits: ["@kit.DeviceSecurityKit", "@kit.ArkData", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.AbilityKit", "@kit.ArkUI"]
apis: ["deviceCertificate.getDeviceToken", "preferences.getPreferencesSync", putSync, getSync, hasSync, deleteSync, flushSync, "UIContext.getPromptAction", showToast, "@StorageProp", "window.getWindowAvoidArea", safeAreaPadding, hilog]
permissions: ["ohos.permission.INTERNET"]
min_api: 21
modules: [entry]
findings: [HW-16-0006, HW-16-0007, HW-16-0013, HW-16-0027, HW-16-0028]
status: verified-with-fixes
---

## When to use

Load this card when a promotion must be **granted once per physical device**,
not once per account — first-order coupons, new-user gifts, free trials, sign-up
bonuses. The account-based check is trivial and useless: a user creates a second
account and claims again. The device-based check needs an identifier the app is
not allowed to read, and this is the pattern that gets one anyway.

The mechanism is Device Security Kit's **DeviceVerify**. The device produces a
short-lived, opaque `deviceToken`; the app never resolves it to a device ID.
The token travels to *your* server, which asks the Device Security server two
things about it — is this a genuine Huawei device, and are its two per-app
marking bits set. Your server grants the coupon and flips a bit. Nothing on the
device is trusted, and nothing persistent about the device is exposed.

That split is what generalises. Any "once per device" rule follows the same
four beats: get a token, verify it, read the mark, write the mark — with beats
two through four running server-side. Replace "coupon" with "trial period" or
"referral bonus" and the shape does not change. What does change per app is the
meaning of the two bits: the platform defines nothing, the app defines both.

## Feature checklist

- A coupon list page shows two offers; only the first one has a live 领取
  (claim) button.
- On entering the page, the previously stored mark status decides whether the
  button reads 领取 or 已领取 (already claimed) and whether it is dimmed.
- Tapping 领取 runs the full check chain before granting anything.
- Each outcome raises its own toast: claim succeeded, claim failed, already
  claimed, device verification failed, unknown.
- A device that fails verification is **not** marked as having claimed.
- A device that has already been marked is refused without a second grant.
- The claim state survives a restart (preferences are flushed after every
  attempt).

## Architecture

One `entry` module. Three layers, cleanly separated, and no networking at all —
the server round-trips are stubbed with preferences.

```
entry/src/main/ets
├── entryability/EntryAbility.ets      full-screen window, avoid areas -> AppStorage
├── model
│   ├── CouponData.ets                 COUPON_DATA: two static CouponInfo records
│   └── CouponInfo.ets                 the coupon interface (id/name/discount/condition/time/label)
├── pages/MainPage.ets                 @Entry: header + coupon List + the claim button's handler
└── utils
    ├── DataPreferenceUtils.ets        static wrapper over preferences (open/put/get/has/delete/clear/flush)
    └── DeviceVerifyUtils.ets          the whole device-status state machine
```

The documented tree lists `model/CouponInfo.ts`; the zip ships
`CouponInfo.ets` (`HW-16-0007`). Everything else matches.

**The design decision worth copying** is that `DeviceVerifyUtils` exposes a
single `getDeviceStatus(): Promise<string>` returning a *status word*, not a
boolean and not a thrown error. `MainPage` never sees a token, a bit, or an HTTP
code — it switches on `'CollectSuccess' | 'CollectFailed' | 'HasCollected' |
'DeviceVerifyFailed' | 'NoDeviceToken'` and maps each to a string resource. That
is the right boundary: the UI has one decision (which toast, and does the button
flip), and the risk-control logic can be swapped for a real backend call without
touching a line of the page.

The four private steps behind it — `queryDeviceToken`, `verifyDeviceToken`,
`queryDeviceMarkStatus`, `updateDeviceMarkStatus` — map one-to-one onto the
REST APIs the guide names (`getDeviceToken`, `checkDeviceToken`,
`getDeviceStatus`, `setDeviceStatus`). Only the first is a real device API; the
other three are the ones your server owns. Keeping them as separate methods
with the same names is what makes the sample readable as a specification of the
server contract.

Where the structure is a trap: `getDeviceStatus` wraps an `async` executor in
`new Promise`, which is the anti-pattern behind `HW-16-0006`. See the corrected
snippet below.

## Implementation steps

1. **Open the preferences store once**, in `aboutToAppear`, before anything
   reads it — `DataPreferenceUtils.openDataPreference(context, { name: 'deviceVerifyInfo' })`.
   Every accessor in that class is a static over one shared `Preferences`
   handle, so an unopened store is a null dereference, not an empty result.
2. **Seed the button from the stored mark** by calling `queryDeviceMarkStatus()`
   on entry, so a device that already claimed shows 已领取 without a round-trip.
3. **Fetch the device token lazily and cache it** in a field. `getDeviceToken`
   is a network-backed promise; the guide warns explicitly that it can fail on
   an unstable network and that the app must stay usable when it does.
4. **Run the chain sequentially with `await`,** and **`return` after every early
   resolve** (`HW-16-0006`). The shipped version resolves and keeps going, so a
   device that fails verification still gets marked as having claimed — it burns
   the one-per-device coupon for a device that never passed the check.
5. **Only write the mark after the grant succeeds.** `updateDeviceMarkStatus`
   is the last step, not a step run in parallel with the query.
6. **Define what bit0 and bit1 mean, in one place.** The platform gives two
   bits per app per device and assigns them no meaning. This sample uses bit0
   for "claimed" and leaves bit1 unused; `queryDeviceMarkStatus` ORs both,
   which is only correct while bit1 stays unused.
7. **Flush after every attempt** (`DataPreferenceUtils.flushAll()` in the
   `then`), otherwise the mark lives in memory and a kill loses it.
8. **Declare `ohos.permission.INTERNET`** — it is a `system_grant` permission,
   so no request dialog, but the entry is mandatory for the real round-trips.
9. **Do not copy the document's snippet** (`HW-16-0007`, `HW-16-0013`): it
   declares `Promise<string>` and returns nothing, and drops every `await`, so
   the ordering it describes in prose is not the ordering it prints.

## Verified snippets

All snippets are from `GetCoupons.zip`. Corrected forms are marked.

**The status chain — `entry/src/main/ets/utils/DeviceVerifyUtils.ets`** (corrected — see `HW-16-0006`)

```typescript
import { deviceCertificate } from '@kit.DeviceSecurityKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { DataPreferenceUtils } from './DataPreferenceUtils';

const DOMAIN = 0x0000;
const TAG = 'DeviceVerify';

export class DeviceVerifyUtils {
  private deviceToken: string = '';

  async getDeviceStatus(): Promise<string> {
    try {
      if (!this.deviceToken || this.deviceToken === '') {
        // 获取设备的token
        await this.queryDeviceToken();
        if (this.deviceToken === '') {
          return 'NoDeviceToken';                 // FIX: shipped code resolves and falls through
        }
      }
      // 验证该设备是否为真实的华为设备，需要携带token
      const verifyRes: boolean = await this.verifyDeviceToken();
      if (!verifyRes) {
        return 'DeviceVerifyFailed';              // FIX: shipped code went on to mark the device
      }
      // 查询设备的标记状态
      const markStatus: boolean = await this.queryDeviceMarkStatus();
      if (markStatus) {
        return 'HasCollected';                    // FIX: shipped code re-marked an already-marked device
      }
      // 更新设备标记状态，被标记代表已经领取过优惠券
      const updateRes: boolean = await this.updateDeviceMarkStatus(true, false);
      return updateRes ? 'CollectSuccess' : 'CollectFailed';
    } catch (err) {
      hilog.info(DOMAIN, TAG, 'getCoupon Error: %{public}s', (err as BusinessError).message);
      throw err as Error;
    }
  }
}
```

**Two things changed, and only one of them is cosmetic.** The
`new Promise(async (resolve, reject) => …)` wrapper goes away because the body
was already `async`: an async method *is* a promise factory, and wrapping one
in an explicit `Promise` constructor buys nothing while making `resolve` look
like a `return` that it is not. That is the cosmetic half.

The load-bearing half is the four `return`s. In the shipped code every branch
calls `resolve('…')` with **no return after it**, so control falls straight
through to the next step. A promise only settles once, so the caller does see
`'DeviceVerifyFailed'` — but by then `await this.updateDeviceMarkStatus(true, false)`
has already run and the device is marked as having claimed. Against a real
Device Security server that is a permanent, per-device write triggered by a
*failed* verification: the user is refused the coupon and can never claim it
again. The same fall-through fires two extra server requests on every early
exit.

**Token acquisition — same file** (as shipped)

```typescript
queryDeviceToken(): Promise<string> {
  return new Promise((resolve, reject) => {
    // 查询设备token，这里的token实际是临时的，不过可以通过该token来查询设备相关信息
    deviceCertificate.getDeviceToken()
      .then((token: string) => {
        this.deviceToken = token;
        DataPreferenceUtils.putValue('deviceToken', this.deviceToken);
        resolve(token);
      })
      .catch((err: BusinessError) => {
        reject(err);
      });
  });
}
```

**`getDeviceToken` is the only real platform call in the file** — everything
after it is your server's job. The token is temporary and opaque; the reference
describes it as "the device token of a device", and the develop guide is
explicit that the app hands it to the app server, which sends it on to the
Device Security server. Nothing here resolves it to a device identifier, which
is the whole point: the app gets per-device uniqueness without holding a
persistent device ID.

Note that the failure path is a `reject`, and the caller in `getDeviceStatus`
catches it — so a network failure surfaces as an exception, while an *empty*
token surfaces as the `'NoDeviceToken'` status word. Both need handling; the
guide calls this out directly, recommending a retry or a fallback risk factor
rather than letting the feature break.

**The two mark bits — same file** (as shipped)

```typescript
queryDeviceMarkStatus(): Promise<boolean> {
  return new Promise((resolve) => {
    // 向Device Security服务器请求设备的标记状态，bit0、bit1两位，代表共有4种状态
    // 此处用首选项模拟
    let bit0 = false;
    let bit1 = false;
    if (DataPreferenceUtils.hasKey('bit0')) {
      bit0 = DataPreferenceUtils.getValue('bit0', false) as boolean;
    }
    if (DataPreferenceUtils.hasKey('bit1')) {
      bit1 = DataPreferenceUtils.getValue('bit1', false) as boolean;
    }
    resolve(bit0 || bit1);
  });
}

updateDeviceMarkStatus(bit0: boolean, bit1: boolean): Promise<boolean> {
  return new Promise((resolve, reject) => {
    // 向Device Security服务器发起请求更新对应设备的标记状态
    try {
      DataPreferenceUtils.putValue('bit0', bit0);
      DataPreferenceUtils.putValue('bit1', bit1);
      resolve(true);
    } catch (err) {
      hilog.info(DOMAIN, TAG, JSON.stringify(err));
      reject(err);
    }
  });
}
```

**The two bits are the entire storage budget, and their meaning is yours.**
The develop guide states it plainly: "The Device Security server provides two
bits for each app on each device to store and query the device marking status.
The meanings of the two bits are defined by the app," and gives exactly this
example — bit0 = has the new-user gift been claimed. Four states total, per app
per device, and marking is also available at developer granularity (`mode: 2`),
shared across all apps of one developer.

The `bit0 || bit1` read is the part to think about before copying. It means
"marked at all", which is right while only bit0 is in use and wrong the moment
bit1 gets a second meaning — a device marked for a *different* campaign would
be refused this coupon too. If you use both bits, test the one you mean.

The preferences stand-in is honest about being a stand-in: the comments say
`此处省略真实请求过程` (the real request is omitted here) and `此处用首选项模拟`
(simulated with preferences). Do not ship the client-side version — a mark that
lives in the app's own preferences is cleared by reinstalling the app, which
defeats the feature entirely.

**Wiring the status word to the UI — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
aboutToAppear() {
  DataPreferenceUtils.openDataPreference(this.context, { name: 'deviceVerifyInfo' });
  this.deviceVerifyUtils.queryDeviceMarkStatus()
    .then((result: boolean) => {
      this.hasGet = result;
    });
}

// inside the coupon builder, for info.id === 0 only:
Button(this.hasGet ? $r('app.string.has_get') : $r('app.string.get'), { type: ButtonType.Normal })
  .opacity(this.hasGet ? 0.5 : 1)
  .onClick(() => {
    this.deviceVerifyUtils.getDeviceStatus()
      .then((result: string) => {
        let tip: ResourceStr = '';
        switch (result) {
          case 'CollectSuccess':
            this.hasGet = true;
            tip = $r('app.string.collect_success');
            break;
          case 'CollectFailed':
            tip = $r('app.string.collect_failed');
            break;
          case 'HasCollected':
            tip = $r('app.string.has_collected');
            break;
          case 'DeviceVerifyFailed':
            tip = $r('app.string.device_verify_failed');
            break;
          default:
            tip = $r('app.string.unknown');
            break;
        }
        this.getUIContext().getPromptAction().showToast({ message: tip, duration: 1500 });
        DataPreferenceUtils.flushAll();
      })
      .catch((err: BusinessError) => {
        hilog.error(0x0001, 'testTag', 'getCoupon Error: %{public}s', err.message);
      });
  });
```

**`hasGet` is set in exactly one case.** Only `'CollectSuccess'` flips the
button; `'HasCollected'` deliberately does not, because on that path the button
was already showing 已领取 from `aboutToAppear`. The `default` branch catching
`'NoDeviceToken'` under 未知 (unknown) is the weak spot — a token failure is a
transient network problem and deserves a retry affordance, not a dead end.

`flushAll()` runs on both success and failure, which is correct here: the
simulated server writes go through `putSync` into the in-memory preference set
and only reach disk on flush.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` is `system_grant`: no dialog, no `reason`, no `usedScene`
  required. The declaration alone is what the real DeviceVerify round-trips
  need.
- No `routerMap`, no extension abilities. `main_pages` lists `pages/MainPage`
  only.
- `deviceTypes`: `phone`, `tablet`, `2in1`.
- The preferences store is named `deviceVerifyInfo` and holds three keys:
  `deviceToken`, `bit0`, `bit1`.

## Constraints

- API Version 21 Release or later; HarmonyOS 6.0.1 Release SDK or later;
  DevEco Studio 6.0.1 Release or later. `compatibleSdkVersion` is `6.0.1(21)`.
- **DeviceVerify does not support simulators.** Supported devices are phones,
  tablets, PCs/2-in-1 and wearables; TVs from 5.1.1(19). `getDeviceToken`
  itself is available from 5.0.0(12).
- `getDeviceToken` can fail with `201` (no permission), `1003300005` (internal
  error) or `1003300006` (cloud server access failure). All three are
  recoverable — retry or fall back to another risk factor.
- The verification, query and update steps in this zip are **not** real: they
  read and write local preferences. `verifyDeviceToken` returns `true` for any
  device that has a `deviceToken` key at all, so it never fails in the demo.
  The `'DeviceVerifyFailed'` branch is unreachable as shipped, which is
  precisely why `HW-16-0006` survived review.
- The second coupon in the list is decorative: it is rendered at 50% opacity
  with a permanently disabled button and no handler.
- The mark is app-level in this sample (`mode: 1` in REST terms). Developer-level
  marking is shared across all apps of one developer — choose deliberately.

## Pitfalls

- **`HW-16-0006`** (B/medium, confirmed): `getDeviceStatus` keeps executing
  after `resolve()`, so a device that fails verification is still marked as
  having collected the coupon — and every early exit fires two extra server
  requests. Fix: add `return;` after every early resolve, or restructure as
  sequential `return 'X';` statements in an `async` method instead of the
  `new Promise` wrapper.
- **`HW-16-0007`** (E/low, confirmed): the doc's `getDeviceStatus` snippet is
  declared `Promise<string>` but returns nothing, and calls the four steps
  without `await`, so it neither compiles nor enforces the ordering the
  surrounding prose describes. The tree also lists `model/CouponInfo.ts` where
  the zip ships `CouponInfo.ets`. Fix: print the zip's awaited implementation
  and correct the filename.
- **`HW-16-0013`** (E/medium, confirmed, systematic): this doc is one of the
  ~30 shopping/news/media pages whose published snippets are abridged by
  cutting structurally necessary code rather than eliding bodies — here the
  `await`s and the `return`. The zip source is valid in every case; only the
  excerpt is broken. Fix: regenerate excerpts with brace-balanced elision that
  keeps `async`/`await`/`return` skeletons intact.

## References

- `documentation/harmonyos-references/03_system/devicesecurity-deviceverify-api.md` — `getDeviceToken`, its error codes and start version
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/devicesecurity-deviceverify-api
- `documentation/harmonyos-guides/04_system/devicesecurity-deviceverify-develop.md` — the seven-step service process and the meaning of the two mark bits
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/devicesecurity-deviceverify-develop
- `documentation/harmonyos-references/03_system/devicesecurity-deviceverify-getdevicestatus.md` — the REST query your server calls, and `mode` (app- vs developer-level)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/devicesecurity-deviceverify-getdevicestatus
- `documentation/harmonyos-references/03_system/devicesecurity-deviceverify-updatedevicestatus.md` — the REST mark update
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/devicesecurity-deviceverify-updatedevicestatus
- `documentation/harmonyos-references/03_system/devicesecurity-arktsapi-errcode-deviceverify.md` — 201, 1003300005, 1003300006
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/devicesecurity-arktsapi-errcode-deviceverify
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` — `getPreferencesSync`, `putSync`, `flushSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` — `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `huawei_industry_tree/16_shopping/docs/05_get_coupons.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966
