---
id: TRANS-07
title: Boarding and alighting reminders with Notification Kit
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
sample: huawei_industry_tree/06_public_transport/downloads/BusOnOffNotification.zip
kits: ["@kit.NotificationKit", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: [notificationManager.publish, notificationManager.NotificationRequest, notificationManager.ContentType, notificationManager.SlotType, notificationManager.isNotificationEnabled, notificationManager.requestEnableNotification, setInterval, clearInterval, Slider, "@Link"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-06-0039, HW-06-0040, HW-06-0041, HW-06-0042, HW-06-0043, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card when the app must **alert the user at a moment they are not
looking at the screen** - approaching stop, order ready, delivery arriving,
appointment due. It covers the whole Notification Kit path: asking to be allowed
to notify, choosing a slot type, and publishing.

It is also this industry's example of a **timer-driven simulation** with the
lifecycle handled correctly, which is worth having as a contrast to the
automotive dashboard practice where it is not.

## Feature checklist

- On first launch, ask the user to enable notifications - checking first, so an
  already-enabled app does not prompt.
- A toggle per reminder: boarding, alighting.
- The user picks the alighting stop from the route.
- As the vehicle approaches either stop, a notification fires with ringtone,
  vibration, banner or lock-screen presentation.
- A simulated vehicle advances along the route so the flow can be demonstrated.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── components/BottomContent.ets       reminder toggles, stop selection
├── components/BusCome.ets
├── components/BusSlider.ets           the route track, the timer, the triggers
├── components/CustomDialogNotice.ets
├── components/TopContent.ets
├── entryability/EntryAbility.ets      notification permission on startup
├── model/ConstData.ets
├── pages/BusMove.ets                  owns all the journey state
└── utils/BusNotification.ets          the two notification requests
```

`BusMove` owns the state and passes it down with `@Link`, so `BusSlider` and
`BottomContent` mutate the same values: `currentPoint` (where the passenger is),
`onPoint` / `offPoint` (chosen stops, `null` when unset), `onBusNotice` /
`offBusNotice` (the toggles), `busPoint` (the vehicle's half-stop position).

The vehicle advances one **half stop** per second, which is why the trigger
conditions divide `busPoint` by 2 - odd values are between stops, even values
are at one.

## Implementation steps

1. **Ask for notification permission at startup**, checking first
   (`isNotificationEnabled` → `requestEnableNotification`).
2. **Build a `NotificationRequest`** with a content type, a `normal` block and a
   slot type.
3. **Give each distinct notification its own `id`** (`HW-06-0040`).
4. **Fill every field of `normal` with something real** - `additionalText` is
   displayed (`HW-06-0039`).
5. **Resolve the strings from resources** before building the request
   (`HW-06-0043`).
6. **Check `err` in the publish callback** - it is the only failure channel
   (`HW-06-0042`).
7. **Store the interval id and clear it in `aboutToDisappear`.**
8. **Guard against the sentinel your state actually uses** (`HW-06-0041`).

## Verified snippets

All snippets are from `BusOnOffNotification.zip`. Corrected forms are marked.

**Asking to be allowed to notify — `entryability/EntryAbility.ets`** (as shipped)

```typescript
import { notificationManager } from '@kit.NotificationKit';

notificationManager.isNotificationEnabled().then((data: boolean) => {
  hilog.info(DOMAIN_NUMBER, TAG, 'isNotificationEnabled success, data: ' + JSON.stringify(data));
  if (!data) {
    notificationManager.requestEnableNotification(context).then(() => {
      hilog.info(DOMAIN_NUMBER, TAG, `[ANS] requestEnableNotification success`);
    }).catch((err: BusinessError) => {
      if (1600004 === err.code) {
        hilog.error(DOMAIN_NUMBER, TAG,
          `[ANS] requestEnableNotification refused, code is ${err.code}, message is ${err.message}`);
      } else {
        hilog.error(DOMAIN_NUMBER, TAG,
          `[ANS] requestEnableNotification failed, code is ${err.code}, message is ${err.message}`);
      }
    });
  }
}).catch((err: BusinessError) => {
  hilog.error(DOMAIN_NUMBER, TAG,
    `isNotificationEnabled fail, code is ${err.code}, message is ${err.message}`);
});
```

**Copy this shape.** Notification permission is **not** declared in
`module.json5` - there is no `ohos.permission.NOTIFICATION` to request. It is
granted through this API instead, and `isNotificationEnabled` first means an
already-enabled app never re-prompts. `1600004` is the refusal code, worth
separating from a genuine failure.

**Publishing a reminder — `utils/BusNotification.ets`** (corrected, see `HW-06-0039`, `HW-06-0040`, `HW-06-0043`)

```typescript
import { notificationManager } from '@kit.NotificationKit';

const ON_BUS_ID: number = 1;        // FIX: shipped code uses id 1 for both
const OFF_BUS_ID: number = 2;

static onBusNotification(context: Context, stopName: string): void {
  const notificationRequest: notificationManager.NotificationRequest = {
    id: ON_BUS_ID,
    content: {
      notificationContentType: notificationManager.ContentType.NOTIFICATION_CONTENT_BASIC_TEXT,
      normal: {
        // FIX: shipped code hardcodes '请上车' / '车辆即将到站'
        title: context.resourceManager.getStringSync($r('app.string.board_title').id),
        text: context.resourceManager.getStringSync($r('app.string.board_text').id),
        additionalText: stopName        // FIX: shipped code ships 'test_additionalText'
      }
    },
    notificationSlotType: notificationManager.SlotType.SOCIAL_COMMUNICATION,
  };
  notificationManager.publish(notificationRequest, (err: BusinessError) => {
    if (err) {
      Logger.error(TAG, `Failed to publish notification. Code is ${err.code}, message is ${err.message}`);
      return;
    }
    Logger.info(TAG, 'Succeeded in publishing notification.');
  });
}
```

Three things to get right:

- **`id` identifies the notification.** Publishing again with the same id
  *replaces* the existing one rather than adding a second. Distinct reminders
  need distinct ids.
- **`SlotType.SOCIAL_COMMUNICATION`** is what earns the ringtone, vibration and
  banner treatment the document promises. `SERVICE_INFORMATION` and
  `CONTENT_INFORMATION` are quieter; the slot type is the presentation choice.
- **`normal` takes plain strings, not `Resource` objects**, so localised text
  must be resolved through `resourceManager` before the request is built. That
  is why hardcoded literals are so easy to end up with here.

`additionalText` is a third displayed line, not a debug field.

**The timer, with its lifecycle handled — `components/BusSlider.ets`** (corrected, see `HW-06-0041`)

```typescript
private timerID: number = 0;

aboutToAppear() {
  this.timerID = setInterval(() => {
    if (this.busPoint >= 20) {
      return;                       // route finished
    }
    this.busPoint = this.busPoint + 1;              // half a stop per tick
    this.offsetX = this.offsetX + this.halfScalesWidth;

    if (this.ifOnBus && this.busPoint % 2 === 0 && this.busPoint / 2 - 1 === this.currentPoint) {
      this.currentPoint = this.currentPoint + 1;    // passenger rides along
    }
    if ((this.busPoint + 1) / 2 === this.currentPoint && !this.ifOnBus && this.onBusNotice) {
      BusNotification.onBusNotification();          // one half-stop before arrival
    }
    if (this.busPoint / 2 === this.currentPoint && !this.ifOnBus) {
      this.ifOnBus = true;                          // boarded
    }
    if (this.offPoint !== null) {                   // FIX: shipped guard tests undefined
      if ((this.busPoint + 1) / 2 === this.offPoint && this.offBusNotice) {
        BusNotification.offBusNotification();
      }
      if (this.busPoint / 2 === this.offPoint) {
        this.ifOnBus = false;
      }
    }
  }, 1000);
}

aboutToDisappear() {
  clearInterval(this.timerID);      // shipped code does this correctly
}
```

**The `aboutToDisappear` is the part to copy.** The automotive dashboard
practice starts a 100 ms interval and never captures or clears it; this one
stores the id and releases it. That is the difference between a demo timer and a
leak.

The `+ 1` in `(this.busPoint + 1) / 2` is what makes the alert fire *one half
stop before* arrival rather than on arrival - the lead time is encoded in the
condition, not in a separate constant.

**Route track from a Slider — `components/BusSlider.ets`** (as shipped)

```typescript
Stack() {
  Image($r('app.media.bus'))
    .translate({ x: this.offsetX });          // the vehicle moves by offset
  Slider({ min: 0, max: 100, step: 10, value: this.sliderValue })
    .width(this.sliderTrackWidth);            // the track
  Row() {
    ForEach(Array.from({ length: 11 }), (item: number, index) => {
      // one icon per stop, visibility chosen by index vs onPoint/offPoint/currentPoint
    })
  }
}
```

Using a `Slider` purely as a **drawn track** with icons stacked over it, and
moving the vehicle with `translate` rather than by re-laying out, is a cheap way
to build a progress rail. `step: 10` over `min: 0, max: 100` gives the eleven
stop positions.

## Permissions & config

**None in `module.json5`.** Notification permission is granted at runtime
through `requestEnableNotification`, not declared as a permission.

Resource directories: `base`, `dark` - no locale qualifiers at all
(`HW-06-0043`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The industry's architecture practice
  requires API 24.
- The vehicle movement is a `setInterval` simulation, not a real feed. A real
  implementation replaces the timer with arrival data and must then handle the
  app being backgrounded, which this sample does not.
- `normal` content fields are plain strings; resources must be resolved first.
- Publishing with an existing id updates rather than adds.

## Pitfalls

- **`HW-06-0039` — `additionalText` ships as `'test_additionalText'` and
  `'test_additionalText111'`.** That text is displayed to the passenger in the
  banner and on the lock screen.
- **`HW-06-0040` — both reminders use `id: 1`,** so the alighting reminder
  replaces the boarding one and neither can be cancelled independently.
- **`HW-06-0041` — `if (this.offPoint !== undefined)` never guards.** The unset
  value is `null`, so the condition is always true; its comment claims a check
  that does not happen.
- **`HW-06-0042` — the document empties the `publish` callback** and drops
  `additionalText` from the content block, hiding the placeholder above.
- **`HW-06-0043` — the notification strings are hardcoded Chinese** and the
  module ships no locale resource directories at all.

## References

- `documentation/harmonyos-guides/07_application-services/text-notification.md` - publishing basic text notifications
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/text-notification
- `documentation/harmonyos-references/06_application-services/js-apis-notificationManager.md` - `NotificationRequest`, `ContentType`, `SlotType`, `publish`, `requestEnableNotification`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-notificationmanager
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `Slider`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `AUTO-06` - the automotive dashboard practice, whose timer is *not* cleared; compare the two
