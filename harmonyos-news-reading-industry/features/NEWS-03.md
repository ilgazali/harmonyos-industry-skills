---
id: NEWS-03
title: Minors mode - follow the system's minors protection, or run your own behind a password
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/03_new_minors_protection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325
sample: huawei_industry_tree/11_news_reading/downloads/NewMinorsProtection.zip
kits: ["@kit.AbilityKit", "@kit.AccountKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [commonEventManager, emitter, hilog, minorsProtection, preferences, window, "minorsProtection.supportMinorsMode", "minorsProtection.getMinorsProtectionInfoSync", "minorsProtection.leadToTurnOnMinorsMode", "minorsProtection.leadToTurnOffMinorsMode", "minorsProtection.verifyMinorsProtectionCredential", COMMON_EVENT_MINORSMODE_ON, COMMON_EVENT_MINORSMODE_OFF, canIUse, "@StorageProp", "@StorageLink", "@Watch", NavPathStack, NavDestinationMode, bindSheet]
permissions: []
min_api: 20
modules: [minorsabilities (har), entry (entry)]
findings: [HW-11-0004, HW-11-0005, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when content has to be **filtered by the reader's age**, and the
age might come either from the system or from the app itself. This is the
regulatory shape for reading, news, video and game apps on HarmonyOS: the
device has a minors mode of its own, exposed through
`minorsProtection` in Account Kit, and an app is expected to honour it - but
must still work when the device does not support it, or when the user has told
the app not to follow the system.

The sample builds both tracks and picks between them with one predicate,
`isFollowSystem && isSupportSystemMinorsMode()`. On the system track the app
subscribes to two common events, reads an age band from
`getMinorsProtectionInfoSync()`, and delegates every switch and every identity
check to the system. On the app track it runs its own six-digit password, its
own toggles, and its own age from a profile. Both tracks converge on a single
`AppStorage` key, `minorsAgeLimit`, and every content view filters off that one
value.

That convergence is the transferable idea. **Whatever decides the age, let one
observable value carry it**, and let views react rather than ask. It works the
same way for parental controls, regional content ratings, or a subscription
tier that gates a catalogue.

**Read `HW-11-0004` before shipping the filter.** The comparison in the sample
is off by one band, and the document's prose says the opposite of the
document's own code.

## Feature checklist

- On first launch a centred half-modal sheet offers to turn minors mode on, with
  a policy paragraph, a 确认开启 (confirm) button and a 暂不使用 (not now) button.
- 我的 → 未成年人模式 opens a settings page with four rows: minors mode, use-time
  limit, follow system settings, password management.
- Turning minors mode on for the first time (app track) forces a six-digit
  password to be set and confirmed before the mode engages; turning it off, or
  changing any other switch while it is on, first raises a six-dot dialog.
- On the system track the same actions delegate to
  `verifyMinorsProtectionCredential` and `leadToTurnOnMinorsMode` /
  `leadToTurnOffMinorsMode`, which hand the user to the system UI.
- With minors mode on, articles rated above the reader's age band disappear from
  both the recommendation strip and the ranking strip; an emptied strip is
  replaced by a 暂无内容 (no content) placeholder rather than collapsing.
- Turning minors mode off restores every article immediately, without a reload.
- The mode survives a restart: it is re-read from preferences at ability start,
  and on the system track re-checked against the system's current state.
- Password management is only reachable while the app-track mode is on, and is
  itself behind the password dialog.

## Architecture

Two modules: an `entry` HAP and a `minorsabilities` HAR that owns everything to
do with the mode. The HAR has no UI.

```
entry/src/main/ets
├── component
│   ├── CommonComponent.ets          commonTitle + showCommonTip helpers
│   └── MinorsModeConfirm.ets        the first-launch sheet, a DIALOG NavDestination
├── constants/CommonConstant.ets     lengths, colours, TRUE_RESULT
├── entryability/EntryAbility.ets    preferences open, mode init, avoid areas
├── model
│   ├── DataModel.ets                ArticleContent / RankArticle, each with ageLimit + visible
│   └── ObservedArray.ets            Array<T> subclass with a change() dirty flag
├── pages
│   ├── Index.ets                    home/mine Tabs; pushes MinorsModeConfirm on first build
│   ├── MinorGuardianPage.ets        the four settings rows and all the branching
│   ├── MinorsPassPage.ets           set / change the six-digit password
│   ├── OriginPage.ets               the Navigation host, 15 lines
│   ├── UserInfoPage.ets             the profile that supplies the app-track age
│   └── VerifyPassDialogPage.ets     the six-dot password dialog
└── view
    ├── HomeView.ets                 the two article strips and the age filter
    └── MineView.ets                 the account tab

feature/minorsabilities/src/main/ets
├── constants/MinorsConstant.ets     the three emitter event ids (1, 2, 3)
├── model/UserDataModel.ets          UserInfo, AGE_SEGMENTATION, getAgeLimit()
└── utils
    ├── DataPreferenceUtils.ets      a static wrapper over one preferences store
    └── MinorsProtectionUtils.ets    both tracks: subscriptions, on/off, verify
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is that `minorsabilities/Index.ets`
exports an *instance*, not a class:

```typescript
export default new MinorsProtectionUtils();
export { DataPreferenceUtils } from './src/main/ets/utils/DataPreferenceUtils';
export * from './src/main/ets/model/UserDataModel';
```

So `EntryAbility`, `MinorGuardianPage` and `VerifyPassDialogPage` all talk to
the same object, which is the only thing in the app that holds the two pieces
of mutable policy state (`isFollowSystem`, `isMinorsModeSupport`) and the only
thing that ever calls `minorsProtection` or `commonEventManager`. No page
imports Account Kit. Swapping the system track for a different provider is a
one-file change.

The second half of the decision is the boundary between the utility and the UI.
The utility never touches a view; it writes four `AppStorage` keys
(`minorsModeOn`, `systemTimeLimitOn`, `minorsAgeLimit`, `followSystem`) and the
UI subscribes. `MinorGuardianPage` binds all four with `@StorageLink` so its
switches follow a change that arrived from a system common event, and
`HomeView` binds only `minorsAgeLimit`, with `@Watch`, because the age is the
only thing content cares about.

## Implementation steps

1. **Probe the capability before anything else.**
   `canIUse('SystemCapability.AuthenticationServices.HuaweiID.MinorsProtection')`
   *and* `minorsProtection.supportMinorsMode()` - the syscap guard keeps the
   build running on devices without Account Kit, the runtime call answers
   whether this device offers the mode.
2. **Open the preferences store in `onWindowStageCreate`, before loading
   content**, and read `followSystem` (defaulting to `true`) into `AppStorage`.
   Every later branch reads it.
3. **On the system track, subscribe to both common events**
   (`COMMON_EVENT_MINORSMODE_ON` / `_OFF`) with `createSubscriberSync` +
   `subscribe`. The ON handler is where you read the age band; there is no
   polling API to fall back on.
4. **On the app track, register three `emitter` listeners** - mode, time limit,
   follow-system - and drive the mode by emitting rather than by assignment, so
   the two tracks look identical to the UI.
5. **Store an age *band*, not an age.** `getAgeLimit` maps the profile age into
   `[0,3) [3,8) [8,12) [12,16) [16,18)` and stores the upper bound; the system
   track stores `ageGroup.upperAge`. Both land in the same key.
6. **Gate the first enablement behind password creation**, then open the mode
   in the `onPop` callback of the password page - not before it.
7. **Hash the password before persisting it** (`HW-11-0005`); never compare
   plaintext.
8. **Filter with `<=`, so content rated exactly at the reader's band stays
   visible** (`HW-11-0004`).
9. **Drive visibility, not array membership.** Set `visible` on each item, bind
   `.visibility(...)`, then call the array's `change()` to force a re-render;
   the arrays themselves are never rebuilt. Track whether *anything* survived
   and swap in a placeholder when nothing did, or the layout collapses.
10. **On disable, reset the age key to `'All'` and mark everything visible** -
    the restore path is a separate branch, not the filter with a wide bound.

## Verified snippets

All snippets are from `NewMinorsProtection.zip`. Corrected forms are marked.

**Both tracks, one initialiser — `feature/minorsabilities/src/main/ets/utils/MinorsProtectionUtils.ets`** (as shipped)

```typescript
private youthEventInfo: commonEventManager.CommonEventSubscribeInfo = {
  events: [commonEventManager.Support.COMMON_EVENT_MINORSMODE_ON,
    commonEventManager.Support.COMMON_EVENT_MINORSMODE_OFF]
};

initMinorsProtection(): void {
  this.isFollowSystem = AppStorage.get<boolean>('followSystem') as boolean;
  this.isMinorsModeSupport = this.isSupportSystemMinorsMode();

  if (this.isMinorsModeSupport) {
    try {
      this.youthSubscriber = commonEventManager.createSubscriberSync(this.youthEventInfo);
      if (this.youthSubscriber) {
        commonEventManager.subscribe(this.youthSubscriber,
          (error: BusinessError, data: commonEventManager.CommonEventData) => {
            if (error) {
              this.dealCommonEventAllError(error, this.youthEventInfo);
              return;
            }
            if (data.event === commonEventManager.Support.COMMON_EVENT_MINORSMODE_ON) {
              AppStorage.setOrCreate('minorsModeOn', true);
              AppStorage.setOrCreate('systemTimeLimitOn', true);
              const MINORS_INFO: minorsProtection.MinorsProtectionInfo =
                minorsProtection.getMinorsProtectionInfoSync();
              let ageGroup: minorsProtection.AgeGroup | undefined = MINORS_INFO.ageGroup;
              if (ageGroup) {
                AppStorage.setOrCreate('minorsAgeLimit', ageGroup.upperAge);
              }
              DataPreferenceUtils.putValue('appMinorsModeOn', true);
            }
            if (data.event === commonEventManager.Support.COMMON_EVENT_MINORSMODE_OFF) {
              AppStorage.setOrCreate('minorsModeOn', false);
              AppStorage.setOrCreate('systemTimeLimitOn', false);
              AppStorage.setOrCreate('minorsAgeLimit', 'All');
              DataPreferenceUtils.putValue('appMinorsModeOn', false);
            }
          });
      }
    } catch (e) {
      hilog.error(DOMAIN_ID, TAG, `System minors subscribe failed: ${e.code} ${e.message}`);
    }
  }

  if (!this.isFollowSystem || !this.isMinorsModeSupport) {
    emitter.on({ eventId: MinorsConstant.MINORS_MODE_EVENT_ID }, this.minorsEventCallback);
    emitter.on({ eventId: MinorsConstant.TIME_LIMIT_EVENT_ID }, this.timeEventCallback);
  }
}
```

**The system track is subscribed whenever the device supports it, regardless of
`followSystem`; the app track only when the system track is unavailable or
declined.** That asymmetry is deliberate: the system can turn minors mode on
behind the app's back at any moment, and an app that stopped listening because
the user unticked "follow system" would keep serving adult content to a device
the parent just locked down. The app track, by contrast, is pure app state and
can safely be inert.

`getMinorsProtectionInfoSync()` is called *inside* the ON handler rather than
cached at startup, because the age band is only meaningful while the mode is
on. Note that the handler writes `ageGroup.upperAge` - the top of the band, not
the child's actual age, which the API never discloses.

The follow-system listener has to re-register the other two, because the app
track may never have been subscribed at launch:

```typescript
emitter.on({ eventId: MinorsConstant.FOLLOW_SYSTEM_EVENT_ID }, (eventData: emitter.EventData) => {
  AppStorage.setOrCreate('minorsModeOn', false);
  AppStorage.setOrCreate('minorsAgeLimit', 'All');
  AppStorage.setOrCreate('systemTimeLimitOn', false);
  emitter.emit({ eventId: MinorsConstant.TIME_LIMIT_EVENT_ID });
  if (eventData.data) {
    this.isFollowSystem = eventData.data.followSystem as boolean;
    DataPreferenceUtils.putValue('followSystem', this.isFollowSystem);
    if (!this.isFollowSystem) {
      emitter.on({ eventId: MinorsConstant.MINORS_MODE_EVENT_ID }, this.minorsEventCallback);
      emitter.on({ eventId: MinorsConstant.TIME_LIMIT_EVENT_ID }, this.timeEventCallback);
    }
  }
});
```

Switching tracks also forcibly turns the mode off first. That is the safe
direction: it drops privileges rather than silently carrying a system-granted
age band into app-managed state.

**The age filter — `entry/src/main/ets/view/HomeView.ets`** (corrected, see `HW-11-0004`)

```typescript
@State articles: ObservedArray<ArticleContent> = ARTICLES;
@State haveArticle: boolean = true;
@StorageProp('minorsAgeLimit') @Watch('ageLimitChange') minorAgeLimit: number | string = 'All';

aboutToAppear(): void {
  this.ageLimitChange();
}

ageLimitChange(): void {
  if (this.minorAgeLimit === 'All') {              // mode off: everything back
    this.articles.forEach((article) => {
      article.visible = true;
    });
    this.articles.change();
    this.haveArticle = true;
    return;
  }
  if (typeof this.minorAgeLimit === 'number') {
    let articleRes: boolean = false;
    this.articles.forEach((article) => {
      if (article.ageLimit === 'All') {
        article.visible = true;
      } else if (typeof article.ageLimit === 'number' && article.ageLimit <= this.minorAgeLimit) {
        article.visible = true;                    // FIX: shipped code uses `<`
      } else {
        article.visible = false;
      }
      articleRes = articleRes || article.visible;
    });
    this.articles.change();
    this.haveArticle = articleRes;                 // false -> render the placeholder strip
  }
}
```

**`@StorageProp` with `@Watch` is what makes this reactive without a bus.**
Whichever track wrote `minorsAgeLimit` - a system common event, an emitter
callback, or the settings page - the view recomputes. The same handler is
called from `aboutToAppear`, so a page entered while the mode is already on is
filtered on its first frame rather than on the first change.

The `articleRes` accumulator is not bookkeeping: without it, a fully filtered
strip renders as a zero-height `Row` and the page silently loses a section. The
placeholder needs to know that *nothing* survived, which is an OR across the
loop, not the last item's state.

`ObservedArray.change()` is the sample's dirty-flag trick - a `@State` array of
`@Observed` objects does not re-render when a *field* of an element changes, so
the subclass flips a private boolean to make the array itself look modified.

**Password storage — `entry/src/main/ets/pages/MinorsPassPage.ets`** (corrected, see `HW-11-0005`)

```typescript
.onClick(() => {
  if (this.oneNewPass === '' || this.twoNewPass === '') {
    showCommonTip($r('app.string.input_pass_tip'), this.getUIContext());
    return;
  }
  if (this.oneNewPass !== this.twoNewPass) {
    showCommonTip($r('app.string.not_equal_pass_tip'), this.getUIContext());
    return;
  }
  if (this.oneNewPass.length < 6 || this.twoNewPass.length < 6) {
    showCommonTip($r('app.string.less_length_tip'), this.getUIContext());
    return;
  }
  if (this.type === 1) {                                  // change, not create
    let storagePass = DataPreferenceUtils.getValue('verifyPass', '');
    if (storagePass === '' || storagePass !== hashPass(this.oldPass)) {   // FIX: was a plaintext compare
      showCommonTip($r('app.string.old_pass_error_tip'), this.getUIContext());
      return;
    }
    if (storagePass === hashPass(this.oneNewPass)) {
      showCommonTip($r('app.string.repeat_pass_tip'), this.getUIContext());
      return;
    }
  }
  DataPreferenceUtils.putValue('verifyPass', hashPass(this.oneNewPass));  // FIX: was this.oneNewPass
  this.pathStack.pop(CommonConstant.TRUE_RESULT, true);
})
```

`hashPass` is the piece the sample is missing: a salted digest from
`@kit.CryptoArchitectureKit`, or the credential handed to Asset Store Kit. The
verification side changes to match, in `MinorsProtectionUtils`:

```typescript
let storePass: string = DataPreferenceUtils.getValue('verifyPass', '') as string;
if (storePass !== undefined && storePass !== '' && storePass === hashPass(pass)) {
  return Promise.resolve(true);
}
return Promise.resolve(false);
```

**This password is the only thing standing between a child and the off switch.**
A preferences file is plain XML in the app's data directory; storing the
six digits there in clear defeats the feature on any rooted or backed-up
device, and the comparison being plaintext means a debugger breakpoint reveals
it too. Note also the six digits are validated for length but not for triviality
- `000000` passes.

**Three ways to authorise one switch — `entry/src/main/ets/pages/MinorGuardianPage.ets`** (as shipped)

```typescript
this.settingItem(0, $r('app.string.minors_mode'), () => {
  if (this.followSystem && this.supportSystemMinorsMode && this.minorsModeOn) {
    // system track, turning OFF: an alert that hands over to the system UI
    this.getUIContext().showAlertDialog({
      message: $r('app.string.system_minors_tip'),
      primaryButton: { value: $r('app.string.cancel_button_text'), action: () => {} },
      secondaryButton: {
        value: $r('app.string.close_button_text'),
        action: () => {
          MinorsProtectionUtils.turnOffMinorsMode(this.ctx);
        }
      }
    });
  } else if ((!this.followSystem || !this.supportSystemMinorsMode) && this.minorsModeOn) {
    // app track, turning OFF: our own password dialog
    this.pathStack.pushPath({
      name: 'VerifyPassDialogPage', param: null, onPop: (info: PopInfo) => {
        if (info.result && (info.result as string) === CommonConstant.TRUE_RESULT) {
          MinorsProtectionUtils.turnOffMinorsMode(this.ctx);
        }
      }
    });
  } else if (!(this.followSystem && this.supportSystemMinorsMode) &&
    !DataPreferenceUtils.hasKey('verifyPass')) {
    // turning ON for the first time: force password creation, then open in onPop
    this.pathStack.pushPath({
      name: 'MinorsPassPage', param: 0, onPop: (info: PopInfo) => {
        if (info.result && (info.result as string) === CommonConstant.TRUE_RESULT) {
          MinorsProtectionUtils.openMinorsMode(this.ctx);
        }
      }
    });
  } else {
    MinorsProtectionUtils.openMinorsMode(this.ctx);
  }
});
```

**The whole authorisation policy is one `onPop` callback per branch.** Pushing
a `NavDestination` and acting on its result - rather than opening a dialog and
threading a callback through state - keeps "what happens after the user
proves who they are" next to "what the user asked for". `VerifyPassDialogPage`
declares `.mode(NavDestinationMode.DIALOG)`, so it is a route that looks like a
dialog: the back gesture pops it, and `pop('false', true)` is a legitimate
answer.

Note the toggles themselves are inert - `Toggle(...).hitTestBehavior(HitTestMode.None)`
inside a `Button` whose `onClick` runs this handler. The switch is a picture of
the state, never the state's author. That is the correct shape for any control
whose change must be authorised: let the storage decide when the switch moves.

The password-management row adds one more subtlety, worth copying verbatim:

```typescript
this.pathStack.pushPath({ name: 'MinorsPassPage', param: 1 },
  { launchMode: LaunchMode.NEW_INSTANCE, animated: true });
```

Without `NEW_INSTANCE`, pushing `MinorsPassPage` from inside the `onPop` of a
route that was itself pushed from `MinorsPassPage` reuses the popped instance,
and the user lands back on the page they just left in its old state.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` -
`minorsProtection` is guarded by system capability, not by a permission.

Configuration that matters: `"routerMap": "$profile:route_map"` - every page is
a `NavDestination` reached by name through one `NavPathStack` created in
`OriginPage`; `deviceTypes` is `phone`, `tablet`; the ability pins light mode
with `setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`; and one
preferences store, `minorsInfoStore`, holds `followSystem`, `appMinorsModeOn`,
`userInfo` and `verifyPass`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `minorsProtection` lives behind
  `SystemCapability.AuthenticationServices.HuaweiID.MinorsProtection` and
  requires a Huawei account. Every call sits behind `isSupportSystemMinorsMode()`;
  on a device without it the app silently runs the app track only, which is the
  intended degradation.
- The app-track age comes from an editable in-app profile (`MYSELF` is hardcoded
  to age 15), so a child can raise their own age. It is a demonstration, not an
  age assurance mechanism - only the system track supplies an age the app cannot
  influence. `getAgeLimit` also returns the raw age for anyone 18 or over, since
  `AGE_SEGMENTATION` stops at `[16, 18]`; adults are unfiltered only because
  `EntryAbility` checks `userInfo.age < 18` before calling it.
- Neither the `commonEventManager` subscription nor the three `emitter`
  listeners are ever released, and the follow-system handler can register the
  same two callbacks repeatedly across track switches. Add `emitter.off` and
  `commonEventManager.unsubscribe` on `onDestroy` before shipping.
- `EntryAbility` registers `avoidAreaChange` and never unregisters it, and drops
  the promise from `setWindowLayoutFullScreen(true)` - the same boilerplate
  defect that recurs across these industry samples.
- The use-time limit is a switch and a log line: no timer, no accumulation, no
  enforcement.

## Pitfalls

- **`HW-11-0004`** (B/medium, confirmed): the filter uses strict `<`, so an
  article rated exactly at the reader's band is hidden - a 16-band reader loses
  every 16-rated article. The document's step-4 prose says
  "比较用户年龄是否大于等于内容的限制年龄" (visible when the user's age is
  greater than **or equal to** the limit) while its snippet repeats the `<`, so
  the same page teaches two opposite rules. Fix: `<=` in both loops in
  `HomeView.ets` and in the document's snippet.
- **`HW-11-0005`** (D/medium, confirmed): the minors-mode management password
  is written to preferences in clear (`putValue('verifyPass', this.oneNewPass)`)
  and compared in clear on verification. It is the only barrier protecting the
  mode's settings. Fix: store a salted hash - or the Asset Store Kit credential
  - and compare digests.

## References

- `huawei_industry_tree/11_news_reading/docs/03_new_minors_protection.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325
- `documentation/harmonyos-references/06_application-services/account-api-minorsprotection.md` - `supportMinorsMode`, `getMinorsProtectionInfoSync`, `AgeGroup`, `leadToTurnOn/OffMinorsMode`, `verifyMinorsProtectionCredential`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/account-api-minorsprotection
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`, `emit`, `EventPriority`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-references/03_system/js-apis-inner-commonevent-commoneventdata.md` - the common-event payload delivered to the subscriber
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-commonevent-commoneventdata
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `putSync`, `getSync`, `flush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageProp` / `@StorageLink` and when each is right
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `onPop`, `LaunchMode.NEW_INSTANCE`, `NavDestinationMode.DIALOG`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/03_system/asset-store-module.md` - where the management password should actually live
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/asset-store-module
- `NEWS-01` - the news app skeleton this filtering belongs on top of
