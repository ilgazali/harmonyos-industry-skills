---
id: UTIL-11
title: Device-info home-screen widgets - keep form ids in preferences and push updates from the app and from a call event
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/11_device_info_card.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/device_info_card-0000002286708232
sample: huawei_industry_tree/15_utilities/downloads/DeviceInfoCard.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.FormKit", "@kit.IPCKit", "@kit.PerformanceAnalysisKit"]
apis: [batteryInfo, common, commonEventManager, deviceInfo, formBindingData, formInfo, formProvider, hilog, preferences, rpc, storageStatistics, util]
permissions: [ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0031, HW-15-0032, HW-15-0033, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when the app should publish a live value onto the home
screen** - battery level, free storage, step count, an unread badge, a
delivery status - through an ArkTS service widget rather than a notification.

Two update routes are demonstrated, and the pairing is the point. The
**app-pushes** route: the UIAbility subscribes to a common event
(`usual.event.BATTERY_CHANGED`), recomputes the values and calls
`formProvider.updateForm` for every known form id. The **widget-pulls** route:
a refresh icon on the card calls `postCardAction` with `action: 'call'`, which
wakes the UIAbility in the background and runs the method named in `params`.
A widget process cannot read device APIs itself, so every value it shows must
arrive by one of these two paths.

Holding it together is a `formId` registry in `preferences`, written in
`onAddForm` and cleaned in `onRemoveForm`. That registry is the piece most
implementations get wrong, and both of this sample's code defects
(`HW-15-0031`, `HW-15-0032`) sit on it or next to it.

## Feature checklist

- Three widget sizes are offered when the user long-presses the app icon: a
  2x2 battery card, a 2x2 storage card and a 2x4 combined device card.
- Each card renders live values: battery percentage with a coloured bar,
  used/total storage, device marketing name and brand.
- The battery bar turns red at 20% or less and yellow at 30% or less.
- Each card carries a refresh icon; tapping it updates that card without
  opening the app.
- While the app is open, every system battery-change broadcast refreshes all
  installed cards.
- A newly added card shows the last values the app stored, not zeros.
- Removing a card drops its id from the registry so it is no longer updated.
- The in-app page shows the same three cards inline as ordinary components.

## Architecture

One `entry` module. The distinctive shape is the three-way split between the
page, the widget pages, and the two abilities that mediate between them.

```
entry/src/main/ets
├── common
│  ├── model/FormData.ets            the payload sent to a card (formId + 7 fields)
│  ├── model/FormInfo.ets            formId / formName / formDimension, logged on add
│  └── utils
│     ├── Logger.ets
│     └── PreferencesUtil.ets        singleton store: values + the formIdList registry
├── component
│  ├── BatteryInfoCard.ets           in-app twin of the battery widget
│  ├── StorageInfoCard.ets           in-app twin of the storage widget
│  └── LargeCard.ets                 in-app twin of the 2x4 widget
├── entryability/EntryAbility.ets    callee.on for three call methods; window setup
├── entryformability/EntryFormAbility.ets  onAddForm / onRemoveForm
├── pages/DeviceInfoPage.ets         @Entry: reads device APIs, subscribes to battery events
└── widget
   ├── BatteryInfoWidgetCard.ets     @Entry(localStorage), 2x2
   ├── StorageInfoWidgetCard.ets     @Entry(localStorage), 2x2
   └── DeviceInfo2x4WidgetCard.ets   @Entry(localStorage2), 2x4
```

The documented 工程目录 matches the zip file for file, including the
`component` / `widget` split.

**The design decision worth copying** is that `component/` and `widget/` are
deliberately separate implementations of the same visual card. The widget
files are `@Entry(localStorage)` structs reading `@LocalStorageProp` - the
only channel a form process has - and they may only use the widget subset of
ArkUI. The `component/` files are ordinary `@Component`s taking `@Prop`s from
the page. Trying to share one struct between the two worlds fails, because
the state decorators and the available API surface differ. Duplicating the
layout and keeping `FormData` as the single shared contract is the cheaper
correct answer.

The second decision worth copying is that **all three call handlers are
declared as arrow-function class properties**, not methods:

```typescript
updateBatteryInfoCall = (data: rpc.MessageSequence) => { ... };
```

`callee.on(name, handler)` invokes the handler detached from the instance, so
a normal method would lose `this` and `this.context` - which every handler
needs for `preferences`. The arrow property binds it at construction.

## Implementation steps

1. **Declare the form extension** in `module.json5`: an `extensionAbilities`
   entry of `"type": "form"` pointing at `EntryFormAbility`, with metadata
   `ohos.extension.form` -> `$profile:form_config`.
2. **Describe each card in `form_config.json`**: `name`, `src`, `uiSyntax:
   "arkts"`, `defaultDimension` and `supportDimensions`. Exactly one card
   should carry `isDefault: true`.
3. **Persist the form id in `onAddForm`** - `want.parameters
   ['ohos.extra.param.key.form_identity']` - before returning any data.
   Without the registry the app cannot address the card later.
4. **Seed the new card from preferences** in the same call, so it opens with
   the last known values rather than zeros, and branch on
   `want.parameters['ohos.extra.param.key.form_name']` to return the right
   subset per card.
5. **Default the registry to an empty array**, never `['']`
   (`HW-15-0032`) - an empty-string id makes every refresh call
   `updateForm('')`, which fails on every battery event before the first card
   exists.
6. **Remove the id in `onRemoveForm`** so a deleted card stops being pushed to.
7. **Subscribe to `usual.event.BATTERY_CHANGED`** with
   `commonEventManager.createSubscriber` + `subscribe`, and unsubscribe in
   `aboutToDisappear` guarding on truthiness, not `!== null`
   (`HW-15-0031`).
8. **Push with `formProvider.updateForm(formId, formBindingData
   .createFormBindingData(formData))`** for each id in the registry.
9. **Wire the widget's refresh icon** with `postCardAction(this, { action:
   'call', abilityName: 'EntryAbility', params: { formId, method } })` and
   register the *same* `method` string with `this.callee.on(...)` in
   `onCreate` - the doc's step 2 registers `'updateBatteryInfoCard'` while the
   widget posts `'updateBatteryInfoCall'` (`HW-15-0033`).
10. **Release the callee handlers in `onDestroy`** with `callee.off(name)`.

## Verified snippets

All snippets are from `DeviceInfoCard.zip`. Corrected forms are marked.

**Card creation — `entry/src/main/ets/entryformability/EntryFormAbility.ets`** (as shipped)

```typescript
export default class EntryFormAbility extends FormExtensionAbility {
  onAddForm(want: Want): formBindingData.FormBindingData {
    if (!want || !want.parameters) {
      return formBindingData.createFormBindingData('');
    }

    let formId = want.parameters['ohos.extra.param.key.form_identity'] as string;
    let formDimension = want.parameters['ohos.extra.param.key.form_dimension'] as string;
    let formName = want.parameters['ohos.extra.param.key.form_name'] as string;

    let util = PreferencesUtil.getInstance();
    let preferences = util.getPreferences(this.context);
    // Save form id using preferences.
    util.addFormId(preferences, formId);

    if (formName === 'BatteryInfoWidget') {
      let formData = new FormData(formId);
      formData.batterySOCInfo = preferences.getSync('batterySOCInfo', 0) as number;
      return formBindingData.createFormBindingData(formData);
    }

    if (formName === 'StorageInfoWidget') {
      let formData = new FormData(formId);
      formData.storageInfo = preferences.getSync('storageInfo',
        `${this.context.resourceManager.getStringSync($r('app.string.notFoundStorage').id)}`) as string;
      formData.totalStorageGB = preferences.getSync('totalStorageGB', 0) as number;
      formData.usedStorageGB = preferences.getSync('usedStorageGB', 0) as number;
      formData.freeStorageGB = preferences.getSync('freeStorageGB', 0) as number;
      return formBindingData.createFormBindingData(formData);
    }
    // ... DeviceInfo2x4Widget: all seven fields
    return formBindingData.createFormBindingData('');
  }

  async onRemoveForm(formId: string): Promise<void> {
    PreferencesUtil.getInstance().removeFormId(this.context, formId);
  }
}
```

**Three things carry the design.** `form_identity` is the id the whole
lifecycle turns on - store it here or lose the ability to update the card at
all. `form_name` is the discriminator: one `FormExtensionAbility` serves all
three cards, so the branch decides which subset of `FormData` gets bound; the
returned object's property names must match the `@LocalStorageProp` keys in
the widget page exactly. And the defaults passed to `getSync` are the
resource strings 未找到存储信息 / 未找到设备名 - a card added before the app
has ever run shows a real message, not `0` or `undefined`.

`FormExtensionAbility` runs in its own process with no UI, so it cannot call
`storageStatistics` or `batteryInfo` for fresh values here; reading the last
values the app persisted is the only option, and is why `DeviceInfoPage`
writes every field into preferences on every refresh.

**The id registry — `entry/src/main/ets/common/utils/PreferencesUtil.ets`** (corrected, see `HW-15-0032`)

```typescript
getFormIds(preferences: preferences.Preferences): Array<string> {
  if (preferences === null) {
    Logger.error(TAG, `preferences is null`);
    return [];
  }
  return preferences.getSync('formIdList', []) as Array<string>;   // FIX: sample defaults to ['']
}

addFormId(preferences: preferences.Preferences, formId: string): void {
  try {
    if (preferences.hasSync('formIdList')) {
      let formIds = this.getFormIds(preferences);
      if (formIds.indexOf(formId) === -1) {
        formIds.push(formId);
        this.preferencesPut(preferences, 'formIdList', formIds);
      }
    } else {
      this.preferencesPut(preferences, 'formIdList', [formId]);
    }
    this.preferencesFlush(preferences);
  } catch (error) {
    Logger.error(TAG, `Failed to check the key 'formIds'. Code:${error.code}, message:${error.message}`);
  }
}
```

**The default value is the contract.** Every consumer of `getFormIds` guards
with `if (formIdList.length > 0)` and then iterates, so `['']` is not an empty
list - it is a list containing one invalid id. Before any card is added, each
battery broadcast therefore issues a guaranteed-failing
`updateForm('')`, several times a minute while charging. `[]` makes the
existing `length > 0` guards do what they were written to do.

`addFormId` also dedupes with `indexOf(...) === -1`, which matters because
`onAddForm` fires again for the same id when a temporary card is converted to
a permanent one.

Note `getPreferences` calls `removePreferencesFromCacheSync` before
`getPreferencesSync` on every access. That is deliberate: the page process
and the two ability processes each hold their own cache, and without the
eviction a widget process would keep serving values written by the app before
it started.

**The call-event round trip — widget page and UIAbility** (as shipped; see `HW-15-0033` for the doc)

```typescript
// entry/src/main/ets/widget/BatteryInfoWidgetCard.ets
Image($r('app.media.refresh'))
  .onClick(() => {
    postCardAction(this, {
      action: 'call',
      abilityName: 'EntryAbility',
      params: {
        formId: this.formId,
        method: 'updateBatteryInfoCall'     // must equal the callee.on() name exactly
      }
    });
  });

// entry/src/main/ets/entryability/EntryAbility.ets
updateBatteryInfoCall = (data: rpc.MessageSequence) => {
  let params: Record<string, string> = JSON.parse(data.readString()) as Record<string, string>;
  let formId = params.formId;
  if (formId) {
    let batterySOCInfo = batteryInfo.batterySOC;
    // ... recompute storage/device fields, persist them through PreferencesUtil ...
    let formData = new FormData(formId);
    formData.batterySOCInfo = batterySOCInfo;
    let formMsg: formBindingData.FormBindingData = formBindingData.createFormBindingData(formData);
    formProvider.updateForm(formId, formMsg).then(() => {
      hilog.info(0x0000, TAG, `updateForm success.`);
    }).catch((error: Error) => {
      hilog.error(0x0000, TAG, `updateForm failed: ${JSON.stringify(error)}`);
    });
  }
  return null;
};

onCreate(): void {
  try {
    this.callee.on('updateBatteryInfoCall', this.updateBatteryInfoCall);
    this.callee.on('updateStorageInfoCall', this.updateStorageInfoCall);
    this.callee.on('updateForm2x4Call', this.updateForm2x4Call);
  } catch (err) {
    hilog.error(DOMAIN, 'EntryAbility', `${JSON.stringify(err)}`);
  }
}
```

**`method` is a string match, and nothing warns you when it is wrong.**
`action: 'call'` starts `EntryAbility` in the background if it is not running,
then dispatches to the handler registered under `params.method`. A mismatched
name simply drops the call - the card's refresh icon animates and nothing
happens. The document's step 2 shows `this.callee.on('updateBatteryInfoCard',
...)` against a widget that posts `'updateBatteryInfoCall'`, and its own
`off()` example uses the correct name (`HW-15-0033`).

`params` values must be strings - the id arrives as JSON on an
`rpc.MessageSequence` and is read back with `data.readString()`. Registering
in `onCreate` rather than `onWindowStageCreate` is what makes the background
route work: `onWindowStageCreate` does not run when the ability is started
without a window.

The three handlers differ only in scope: the battery one updates just the
calling card, the other two loop over the whole registry. Given they share
about fifty lines of identical recompute-and-persist code, a single helper
taking a target id list would be the obvious cleanup.

**Battery subscription — `entry/src/main/ets/pages/DeviceInfoPage.ets`** (corrected, see `HW-15-0031`)

```typescript
private subscriber?: commonEventManager.CommonEventSubscriber;

subscribeBatteryInfo() {
  let subscribeInfo: commonEventManager.CommonEventSubscribeInfo = {
    events: ['usual.event.BATTERY_CHANGED']
  };
  try {
    commonEventManager.createSubscriber(subscribeInfo,
      (err: BusinessError, commonEventSubscriber: commonEventManager.CommonEventSubscriber) => {
        if (!err) {
          this.subscriber = commonEventSubscriber;
          commonEventManager.subscribe(this.subscriber,
            (err: BusinessError, data: commonEventManager.CommonEventData) => {
              if (err) {
                return;
              }
              this.getDeviceInfo();
              this.sendFormMsg();
            });
          return;
        }
        Logger.error(`Failed to subscribe. Code is ${err.code}, message is ${err.message}`);
      });
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.error(`Failed to subscribe. Code is ${err.code}, message is ${err.message}`);
  }
}

unsubscribeBatteryInfo() {
  if (this.subscriber) {                     // FIX: sample tests `this.subscriber !== null`
    commonEventManager.unsubscribe(this.subscriber, (err: BusinessError) => {
      if (err) {
        Logger.error(`Failed to unsubscribe. Code is ${err.code}, message is ${err.message}`);
      } else {
        this.subscriber = undefined;
      }
    });
  }
}
```

**`?:` means `undefined`, and `undefined !== null` is `true`.** The guard was
written for the case where subscriber creation failed or has not completed
yet - exactly when the field is still `undefined` - and in that case it lets
`unsubscribe(undefined)` through, which is the only path it was supposed to
block. A truthiness check covers both.

Creation is a callback and `aboutToDisappear` can fire before it resolves, so
this is not a theoretical race: open the page and leave immediately and the
handler is still pending.

`getDeviceInfo` and `sendFormMsg` are two near-identical copies of the same
seven reads and seven `preferencesPut` calls; only the trailing
`updateForm` loop differs. Collapsing them is the first cleanup worth making
in this file.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.KEEP_BACKGROUND_RUNNING",
    "reason": "$string:reason_keep_background_running",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

- `KEEP_BACKGROUND_RUNNING` is `system_grant` but still requires `reason` and
  `usedScene`; the reason resource must exist in
  `resources/base/element/string.json`.
- It is declared for the call route, which resumes `EntryAbility` without a
  window. Note the sample never starts a `backgroundTaskManager` continuous
  task, so the permission alone does not keep the process alive - a card
  refreshed long after the app was killed relies on the system starting the
  ability for the call.
- `form_config.json` sets `updateEnabled: false` on all three cards, so the
  system's own timed refresh is off and every update comes from the app or the
  call event - `scheduledUpdateTime: "10:30"` and `updateDuration: 1` are
  inert while that flag is false.
- `deviceTypes` is `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Widget pages run the ArkTS widget subset: no `import` of most kits, no
  network, no timers. Every value must arrive through `FormBindingData` and be
  received with `@LocalStorageProp`.
- `formProvider.updateForm` is rate-limited by the system; the sample calls it
  on every `BATTERY_CHANGED` broadcast, which is frequent while charging.
  Debounce, or compare against the last pushed value first.
- `storageStatistics.getTotalSizeSync` / `getFreeSizeSync` are synchronous
  and called on the UI thread on every event.
- The registry lives in one `preferences` store (`myStore`) shared by three
  processes, with the cache evicted on every read; it is fine at this size but
  is not a transactional store.
- The in-app `component/` cards duplicate the widget layouts and must be kept
  in step by hand.

## Pitfalls

- **`HW-15-0031`** (B/low, confirmed): `unsubscribeBatteryInfo` guards with
  `this.subscriber !== null` on an optional field whose unset value is
  `undefined`, so the guard never protects the failure path it was written
  for and `unsubscribe(undefined)` is called when creation failed or has not
  completed. Fix: `if (this.subscriber)`.
- **`HW-15-0032`** (B/low, confirmed): `getFormIds` defaults the registry to
  `['']`, so before any card is added every battery event iterates a
  one-element list and issues a guaranteed-failing `updateForm('')`.
  Fix: default to `[]`.
- **`HW-15-0033`** (E/low, confirmed): the doc's 实现思路 step 2 registers the
  callee as `'updateBatteryInfoCard'` while the widget posts
  `'updateBatteryInfoCall'` - and the doc's own `off()` line uses the correct
  name. A reader who follows the doc gets a refresh button that never
  dispatches. Fix: correct the doc.

## References

- `documentation/harmonyos-references/03_system/js-apis-battery-info.md` - `batteryInfo.batterySOC`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-battery-info
- `documentation/harmonyos-references/03_system/js-apis-device-info.md` - `marketName`, `brand`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-device-info
- `documentation/harmonyos-references/02_application-framework/js-apis-file-storage-statistics.md` - `getTotalSizeSync`, `getFreeSizeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-storage-statistics
- `documentation/harmonyos-references/02_application-framework/js-apis-app-form-formextensionability.md` - `onAddForm`, `onRemoveForm`, `onAcquireFormState`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-form-formextensionability
- `documentation/harmonyos-references/02_application-framework/js-apis-postcardaction.md` - `postCardAction`, the `call` action and its `params.method`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-postcardaction
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `getSync`, `putSync`, `removePreferencesFromCacheSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/03_application-framework/arkts-ui-widget-event-uiability.md` - refreshing a card through a router or call event
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-ui-widget-event-uiability
- `documentation/harmonyos-guides/03_application-framework/arkts-ui-widget-active-refresh.md` - pushing updates from the app with `updateForm`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-ui-widget-active-refresh
- `documentation/harmonyos-guides/04_system/common-event-subscription.md` - `createSubscriber` / `subscribe` / `unsubscribe`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/common-event-subscription
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.KEEP_BACKGROUND_RUNNING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `huawei_industry_tree/15_utilities/docs/11_device_info_card.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/device_info_card-0000002286708232
